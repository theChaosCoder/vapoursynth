# Known issue: VSFrame can outlive its VSCore (use-after-free)

**Status:** destructor path mitigated (commit caching `enableFrameRefDebug`); a
complete fix still requires a core-lifetime change — see *Recommended fix*.
**Severity:** high (heap use-after-free / undefined behavior).
**Discovered:** during the audit2 ASAN/UBSAN verification pass. Reproduces on the
pre-audit base commit too, i.e. it is a pre-existing bug, not introduced by the
audit fixes.

## Summary

A `VSFrame` stores only a raw `VSCore *core` and does **not** keep the core
alive. The `VSCore` object is deleted by `VSCore::filterInstanceDestroyed()` as
soon as `numFilterInstances` reaches 0 (`src/core/vscore.cpp`). Framebuffers,
however, are explicitly allowed to outlive the core — `MemoryUse` implements an
`on_core_freed()` deferral for exactly that, and `VSCore::freeCore()` even logs
"Core freed but N bytes still allocated in framebuffers".

So when a frame outlives its core (any C-API / embedding / host that keeps a
`VSFrame` reference after the core and its nodes are gone), every `VSCore`
dereference made through `VSFrame::core` is a use-after-free.

## Affected code

- `VSFrame::~VSFrame()` — read `core->enableFrameRefDebug` (always executed).
  **Mitigated**: the flag is now cached into the frame at construction
  (`VSFrame::frameRefDebug`) so the destructor no longer touches the core in the
  default configuration.
- `VSFrame::getVideoFormatV3()` — calls `core->VideoFormatToV3(...)` lazily
  (API3 compatibility). **Still affected** if invoked on a frame that outlived
  its core.
- The frame-ref-debug bookkeeping (`core->frameRefMutex`, `core->frameRefs`),
  reached only when `enableFrameRefDebug` is on. **Still affected** in that
  debug configuration.

## ASAN evidence

```
==ERROR: AddressSanitizer: heap-use-after-free ... in VSFrame::~VSFrame()
  READ of size 1 ...
  #0 VSFrame::~VSFrame()                  src/core/vscore.cpp:348 (core->enableFrameRefDebug)
  #1 freeFrame(VSFrame const*)
  ...
freed by thread T0 here:
  #1 VSCore::filterInstanceDestroyed()    src/core/vscore.cpp (delete this)
  #2 VSCore::freeCore()
previously allocated by thread T0 here:
  #1 createCore(int)
```

The freed 576-byte region is the `VSCore` object; the read is `enableFrameRefDebug`
located inside it.

## Reproduction

Build with `-Db_sanitize=address,undefined`, then (frames intentionally kept
alive past the core reference):

```python
import vapoursynth as vs
for it in range(4):
    core = vs.core
    core.max_cache_size = 64
    c = core.std.BlankClip(format=vs.RGB24, width=1920, height=1080, length=24, color=[1,2,3])
    c = core.std.BoxBlur(c, hradius=4, vradius=4, hpasses=2, vpasses=2)
    c = core.resize.Bilinear(c, width=1280, height=720)
    held = [c.get_frame(i) for i in range(24)]
    held = held[12:]
    del c, core      # drop core ref while frames are still held
    del held         # frames released after the core -> teardown path
```

Run (Python is not instrumented, so preload the ASAN runtime):

```sh
LD_PRELOAD=$(gcc -print-file-name=libasan.so) ASAN_OPTIONS=detect_leaks=0 \
PYTHONPATH=builddir-asan LD_LIBRARY_PATH=builddir-asan python3 repro.py
```

## Recommended complete fix

Keep the `VSCore` alive while any frame references it, mirroring the existing
`MemoryUse` deferral:

1. Add an atomic frame counter on `VSCore` (e.g. `std::atomic<long> numFrames`).
2. Increment it in every `VSFrame` constructor, decrement it in `~VSFrame()`.
3. In `filterInstanceDestroyed()`, only `delete this` once **both**
   `numFilterInstances == 0` **and** `numFrames == 0`; likewise have the last
   frame's decrement trigger the delete when filters are already gone.

Trade-offs to weigh before applying:
- One extra atomic increment/decrement per frame lifecycle (frames are a hot
  path).
- Changed teardown ordering (the core outlives its nodes until the last frame is
  freed).

The interim caching fix removes the use-after-free for the common (non-debug)
destructor path with no performance cost; the steps above close the remaining
opt-in/API3 paths.
