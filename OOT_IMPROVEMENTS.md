# Ocarina of Time improvements (Kajcsi/Lichii fork)

This branch, `oot-improvements`, is [zortos293/Lichii](https://github.com/zortos293/Lichii) `nx2`
plus fixes aimed at The Legend of Zelda: Ocarina of Time (`0400486024DEC000`). Upstream `nx2` is
merged in regularly, so everything from zortos293/Lichii is here as well.

## Changes on top of upstream

| Change | Status |
| --- | --- |
| **Pause, don't reset, the occlusion counter at each Vulkan submission** (`vk_scheduler.cpp`, `EndPendingOperations`). The reset (from Eden b1f3b568fb) dropped samples counted so far whenever a submission landed between the game's counter reset and its report. The split GPU thread's deferred submissions make that frequent, and OoT then flashes Link white for one frame. | **Fixes most of the white flashes on Link.** 36 s of consecutive frames with Link on screen, one binary, old vs new: 15 and 9 flashes vs 0 and 0. Packaged build: 14 flashes before, 1 after. Upstream PR open on zortos293/Lichii. |
| **Bindless handles read from the host-aligned storage base** (`video_core/shader_handle_program.h`, Vulkan and OpenGL pipelines, unit test in `tests/video_core/texture_handles.cpp`). Handle programs record storage offsets relative to the buffer address aligned down to the host's storage-buffer alignment, as `StorageOffset` does; the CPU evaluator used the unaligned address. | Correct and covered by a unit test. **No visible effect in Ocarina so far:** an A/B run on an RTX 4090 (alignment 16) in Kokiri Forest compared both reads for 524,288 handle evaluations and they never differed, because the game's storage pointers are already 16-byte aligned. The commit message's claim that it fixes the missing vegetation is not borne out. zortos293 found the same issue independently (`fix/oot-fog`, `EDEN_NX2_HANDLE_ALIGN`). |
| **Bound the running occlusion query** (`vk_query_cache.cpp`, `CloseCounter`). With the counter paused at each submission, the query runs until the game resets or reports it. OoT's title screen and file select do neither, so it grew to 16,384 slots over all 64 sample banks. Past one bank (256 slots) it is now dropped as a reset would. | An AMD RX 9070 XT tester saw a hang right after picking the save with `EDEN_NX2_SPLIT=2` with the pause alone. Not reproducible on NVIDIA, AMD retest pending. Gameplay reset-to-report windows (longest 36 slots over 4 submissions) are untouched. Added to the same upstream PR. |
| **Wait for pending query writes before an MME read of them** (`maxwell_3d.cpp`, `WaitForQueryWrites`). Query results reach guest memory only once the fencing thread writes them, and the game's macro can read the address before that. | Removes flashes when the fencing thread is slowed artificially (8 ms per operation: 0 flashes). Doesn't explain every rare natural flash yet, so it stays on the fork. |
| **Retry early UMWAIT wakes until the Event deadline** (`common/thread.cpp`, `Event::WaitFor`). On Intel CPUs with WAITPKG (12th gen and later), any early wake was reported as a timeout, even when the event had been set. | Upstream's Event tests pass on an i9-12900K (7 of 11 failed before). No in-game change in OoT: 0 flashes either way, fps within run-to-run noise. Upstream PR open (zortos293/Lichii#137). |
| **Flush pending query writes before native MME DMA reads**, cherry-picked from Eglantine `oot-jojo` (`50db84983f`). | Affects High GPU accuracy only (with the GPU thread split off). At Low accuracy, report payloads are written immediately and the split already drains before the read. |

## Tested

Windows, RTX 4090, Docked, Low GPU accuracy, `EDEN_NX2_SPLIT=2`, Kokiri Forest (Link's house
area): no shaders rejected (build 75 rejected several gather programs here), no white haze or
patches over Link.

White flashes are counted from about 14 consecutive frames per second over 36 s, captured through
the debug server, with Link on screen idling and then walking. An earlier check of 30 frames sampled
a few seconds apart missed the flashes entirely.

Latest run, outside Link's house with upstream's built-in Max Resolution patch applied: 0 flashes
in 568 frames, 62.2 fps. The first runs after the merge were slower (41 and 49 fps) while new
pipelines compiled.

## Known leads, not fixed yet

- A rarer white flash on Link remains after the occlusion counter fix: 1 in 1,356 captured
  frames. The cause is not identified yet. At Low accuracy the report write still waits for the
  host fence while the guest's syncpoint advances immediately, so a late write is one candidate.
  The query-write wait above covers the macro's reads, but a natural run with it still caught
  flashes, so at least part of it is something else.
- Compiling some new pipelines while the camera pans reads texture descriptors past the TIC
  limit (`index <= tic_limit` asserts followed by garbage formats 119, 127, 21) in
  `GenericEnvironment::ReadTextureInfo`.
- The issues collected in zortos293/Lichii#101 (missing vegetation, the `c72b6ea746c00433`
  half-resolution layer, fog volumes) are still open upstream.
