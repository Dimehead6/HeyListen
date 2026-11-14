# Readbacks, the replay's start and the frame's loop (stream R2, 2026-11-14, main PC)

Question (stream C1): 81% of the 30.7 ms frame on the PC 3050 is a thread waiting inside the renderer, the present on the replay and the
game's CPU reads of GPU results (`invalidate`) on the replay and the GPU's write-back. Which reads wait, why, and can they be
answered without waiting with the picture right; why does the replay start late in the frame. Everything below is measured on the main
PC (Ryzen 7 7800X3D, RTX 5070, castle-move, 1280x720, main 349e5685 plus this stream's logging), the machine otherwise free for the
speed rows, the frame rate a run of 110 s cut at 66 s as always. The PC 3050 was not used.

## Tools built (logging only, default off)

- `RX_VK_RB_PROF=1` (`vk_replay.cpp`, `RbScope`): every `invalidate` timed by phase (the ticket's wait for the replay section in
  progress, the catch-up, the wait for the write-back's recording split into the flush's late-fix wait / the queue's lock / the
  sending / the recordings' semaphore, the GPU timeline, the strict wait, the texture catch-up), by thread, by range and by call site
  (the exe's frames above the renderer, resolved against the link map), a line every 300 presents.
- `HOS_RB_LR_LOG=1` (`host/nvn_native.c`): the CPU's reads of GPU results by guest return address, size and thread.
- `RX_VK_TL=<present>` or `RX_VK_TL_S=<seconds after the first present>`: a timeline of 3 frames of the renderer's threads (submits,
  replays and their stops at event waits, the CPU's event signals, presents and their drains, fence waits, readbacks with the writer's
  value, recordings sent and flushed, the late fixes), in the log as `vk: tl ...` lines (plain text).

## What a frame reads (castle-move, 89-97 fps)

33-34 `invalidate` calls a frame, 19.4 of them on ranges with write-backs, 6-10 of those pending (the writer not done). The time
they wait, summed over the threads that read (several wait at once, so this is not the chain's loss):

| | ms a frame |
|---|---:|
| all readbacks | 6.7-8.0 |
| the ticket (the replay section in progress when the fence was polled) | 1.2-1.4 |
| the write-back's recording | 5.3-6.7 |
| of it: the flush's wait for the pending late fix (`late_fix_wait_unlocked`, the GPU timeline) | 5.1-6.4 |
| of it: the queue's lock, the sending, the recordings' semaphore | 0.1 + 0.0 + 0.1 |
| the GPU timeline wait (`wait_value`), the strict wait, textures | 0.0 |

By thread: ModuleSystemWorker1 2.25, 2 1.82, 3 0.85, 4 0.47, 5 0.33, the main thread 2.28. By range, one call a frame each:
0x113c88b000 +0x1040 (the light meter's buffer, 2.0 ms), a 0x40 range (1.97), the two 0x2000 ranges that alternate frame by frame
(0.97 and 1.0); the rest (0x3980 x2, 0x8b040, 0x9b040, 0x44b040, 0x1b580) wait 0.01-0.05 ms. By caller (guest return address):
the meter's 0x1040 x3 and 0xc0 from the ring helper `f_80357060` (7 callers), the 0x2000 from `f_8007c120` and `f_8034f440`, the
0x40 from `f_80354060` (many addresses in a row, one wait for all), the big ranges from `f_80362ff0`, 0x1b580 from `f_80364080`,
0x3980 from `f_80365f00` and `f_800998a0`. Only 0.2-0.6 of the 33 calls a frame have the newest submission as their writer: the
game reads what the **previous frame's** heavy submission wrote (value of the queue's 4th submit), after polling that submission's
fence (answered "done" early from the recording, see `direct-emission.md` section 12). So Longshot's `RX_VK_READBACK_LATE=newest`
does not apply here (0.2-0.6 calls a frame match it); its by-size rule would name the 0x2000 and 0x40 ranges.

What the wait is for: the GPU's completion of the previous frame's heavy submission. The readbacks call `wait_recorded`, which flushes
the held recordings first (`queue_flush_for`), and the flush's first step waits for the pending late fix, the timeline value of a
capture's late page, which is that same submission's write-back. The recordings' semaphore itself is signalled before the wait begins
or within 0.1 ms. (The first piece of the submission is what `value_rec` keeps, so on this driver, where `RX_VK_READBACK_STRICT` is
off, that wait is the only one that makes a readback see the whole submission's write-back.)

## The frame's loop, from the timeline (`RX_VK_TL_S`)

One frame, in order:
1. The game's CPU phase (main thread, module workers, executors) until the main thread signals the CPU event that the heavy queue's
   first command buffer waits for (0x1006e38000 / 0x1006e38004, alternating by frame).
2. Until then the renderer has only the Presentation Thread's four small submits (each stops at a GPU-event wait, 0.05-0.1 ms of
   replay each) and the main thread's two held submits (capture 0.6 ms, replay 0.1-0.2 ms each). The heavy submission stops at the CPU
   event: **this is why the replay is idle for the first ~10 ms of the frame on the 3050: it has nothing it may run**. The game
   submits the work before its data is final and holds the GPU with an event the CPU sets when it is.
3. The signal; the order thread (`sync_watch`, `Sleep(1)`) or the Presentation Thread's present finds it 0.5-1 ms later (the order
   thread polls; `RX_VK_WATCH_KICK=1` wakes it at the signal), and the heavy submission replays: 4-9 ms here, in pieces that stop at
   GPU-event waits; the present, called at about the same time, waits for the section's lock.
4. The present returns when the drain is done; the next frame's CPU phase begins.
5. In the next CPU phase, about 3-4 ms in, the main thread and the module workers read the previous frame's results (above) and wait
   for the GPU to be done with that heavy submission, 0 to 5 ms (1.4 ms in the frame traced), then go on to signal the event.

The loop is therefore: replay end of frame N-1 -> GPU done with it -> readbacks of frame N -> event signal of frame N -> replay of frame N.
The GPU's own work is 2.6-2.9 ms a frame, with 9 ms of gaps between recordings (the log): the wait looks like the replay handing
the GPU its last recordings late rather than the GPU's speed (an inference: see "Not verified").

## What was tried, speed and picture

Speed: castle-move, the whole machine held (`slot.sh all`), the same binary with only environment switches differing, arms
interleaved (the order reversed each round), 110 s a run cut at 66 s, fps of the run. Picture: `RX_VK_FRAMELOG` for 25 s of each run
(2,400-2,700 frames at 320x180), `scripts/framelog.py` for its flash events, and the mean luma of every frame (an exposure that
pulses or is wrong moves it; `RX_VK_EXPOSURE_LOG=2` was logged too, but the word it names stays 1.0 in this scene, so it says nothing
here). Shots every 600 presents were kept in `build/s5/r2-fl-*-shots/` (local).

| arm | fps, each run | mean | against the baseline | picture (luma mean, range, pulses, framelog flash events) |
|---|---|---:|---:|---|
| baseline (all switches off) | 97.6, 97.7, 97.0, 97.8, 97.2, 98.1 | 97.6 | | 40.6, 26.8-53.5, 0, 4 one-block events |
| `RX_VK_WATCH_KICK=1` | 104.0, 104.1, 105.0, 104.0, 104.1 | 104.2 | **+6.8%** | 41.0, 27.9-54.6, 0, 0 events: right |
| `RX_VK_READBACK_LATE=2000,40` | 98.3, 98.0, 98.2 | 98.2 | +0.6% (inside the spread) | 40.6, 28.7-53.0, 0, 0 events: right |
| `WATCH_KICK=1` + `READBACK_LATE=2000,40` | 105.3, 105.1, 104.4 | 104.9 | +7.5%, +0.7 over the kick alone | not run (each alone is right) |
| `RX_VK_NO_READBACK_WAIT=1` (the bound) | 115.5, 112.9 | 114.2 | +17% | 42.2, 27.8-67.7, **24 pulses**, frame-to-frame step up to 11 (0.5 in the baseline): wrong |
| (built, taken out) `RX_VK_WAIT_REC_SENT=1`: no flush before the wait for a recording already sent | 109.6 (one run) | | +12% | **177.7 mean, 245.9 max: burnt out to white**: wrong |

Of the 12 runs of the first screen, two did not leave the title screen (0 frames measured, the game's present count stops at 39.4 s
after the first present, 360 fps of a 50-draw frame): one of the `NO_READBACK_WAIT` runs and one of the `WATCH_KICK` runs. None of the
9 runs of the second screen, none of the baseline or `READBACK_LATE` runs (9 and 3). The scripted input is the first suspect (the same
instant in both); the renderer's change is not excluded: not established. Said plainly: too few runs to rule out that `WATCH_KICK`
makes the stall more likely.

### What each result says

- **The readbacks wait for the GPU's completion of the previous frame's heavy submission, through the flush's late-fix wait.** 5.1-6.4
  of the 5.3-6.7 ms a frame that the recording's wait costs is `late_fix_wait_unlocked` in `queue_flush_for`, called by `wait_recorded`
  before it waits on the recordings' semaphore. The late page is always the same one, 0x100d6c8000 (pool 0x100c000000 +0x16c8000, not a
  readback page): the page of the GPU event words, which the heavy submission's last event signal writes back (log line "late page
  deferred: ... write-back 2415, submission 2417": the write-back of the previous frame's heavy submission against the capture of the
  next frame's held submit; 598 fixes in 600 presents). It is that wait that holds the frame's other recordings (11.5 command buffers a
  frame held until the fix thread sends them), and it is that wait the readbacks join.
- **Skipping that wait (`WAIT_REC_SENT`) is fast and wrong.** The recordings' semaphore a readback then waits on is the one of the FIRST
  recording the writing submission went into (`value_rec` keeps the first), signalled long before the submission's write-back of the
  meter has landed; on this driver `RX_VK_READBACK_STRICT` is off, so nothing else waits for the whole submission. The flush's late-fix
  wait was the only thing that did, by being the same GPU completion. The meter then read stale sums: the exposure went to white (mean
  luma 177). That wait must not be removed without replacing it by one on the timeline value of the range's write-back
  (`RX_VK_READBACK_STRICT=1` does that, and its own wait flushes through the same late fix, so it costs the same).
- **The answer-late rules do not gain here.** `newest` names 0.2-0.6 of the 33 calls a frame. The by-size rule for the two rings that
  are not the meter (0x2000, 0x40: 3.9 ms of the 8 summed) gives +0.6%: the meter's ranges (0x1040 x3, 0xc0), read by the same threads
  at the same moment, still wait for the same GPU completion, and the threads wait together, so removing some of the waits shortens
  nothing. Only removing all (`NO_READBACK_WAIT`, +17%) moves the frame, and it breaks the meter.
- **Reading the previous frame's value is not what the game is built for.** The game reads, in frame N, what frame N-1's heavy
  submission wrote, after polling that submission's fence; on the console the poll can say "not done" and the game then skips the read
  (its own protocol). The meter is read and reset (0 written), and the GPU adds the next frame's sums into the reset value: a skipped or
  a stale read doubles the next measure (78, 163, 254 in sprint 5's log) and the exposure moves. That is also what the 24 pulses of
  `NO_READBACK_WAIT` are. A rule for the meter's ranges has to be exact.
- **The replay's late start is the game's own wait, not the renderer's.** The heavy queue's first command buffer waits for a CPU event
  (0x1006e38000 / 0x1006e38004), which the main thread signals at the end of its CPU phase. Until then the replay has the Presentation
  Thread's four small submits (0.05-0.1 ms each, stopped at GPU-event waits) and the main thread's two held submits, and nothing else it
  may run: the idle first ~10 ms of the 3050's frame (a 10-14 ms CPU phase there; 5-8 ms on this PC). After the signal the order thread
  (`sync_watch`, `Sleep(1)` with a 1 ms timer) finds it 0.5-1 ms later (0.56, 0.95, 0.64 ms in three traced frames);
  **`RX_VK_WATCH_KICK=1` wakes it at the signal and is worth +6.8% here**, picture right, because that 0.5-1 ms is on the frame's loop
  (signal, replay, present, next frame's CPU phase, signal). The replay stays serial: the replay of frame N cannot overlap the CPU phase
  of frame N+1, because frame N+1's readbacks wait for frame N's heavy GPU work, and its event signal, which the next replay waits for,
  comes after them.

## Switches built (all default off)

| switch | what it does | state |
|---|---|---|
| `RX_VK_WATCH_KICK=1` | the CPU's event signal sets an event the order thread waits on (1 ms timeout), so the stopped queue's replay starts at the signal | +6.8% castle-move, picture right; recommended on once the stalled runs are explained |
| `RX_VK_READBACK_LATE=<rules>` | Longshot's #238 (`newest`, a hex size), plus `-<hex size>` to never; a ranged readback is read without its wait | no gain here; picture right for `2000,40`; the meter must not be named |
| `RX_VK_RB_PROF=1`, `RX_VK_TL=<present>` / `RX_VK_TL_S=<s>`, `HOS_RB_LR_LOG=1` | the measurements above | logging only |

## Not verified

- One scene (castle-move), one PC. Kokiri-idle and hyrule-walk were not run; the 3050 was not used (its numbers, 81% in waits, are C1's,
  not reproduced here: on this PC the readbacks wait 1.4-2.0 ms a frame on the chain, not 9 ms).
- The two runs that stayed on the title screen (above).
- `WATCH_KICK`'s picture was checked by one 25 s capture (2,545 frames) and no shots were compared by eye, only the luma figures.
- The GPU completion's own time: the log's GPU time is 2.6-2.9 ms a frame and its "GPU tail" 0.4-0.7 ms, but a recording submitted at
  4.5 ms of the traced frame was not done on the timeline before 9.9 ms. Where the other ~3 ms go (the segments of the heavy command
  buffer are recorded together by the parallel replay and reach the queue together; the queue submit's own time was not logged until
  the last build) is the open question and the one worth the next stream's time: the heavy submission's GPU work is not pipelined with
  its replay.

## Hand-over

- Branch `stream/r2-readback`, worktree `E:\OOT\HeyListen-wt\r2-readback`. The tools: `RX_VK_RB_PROF` (`RbScope` in `vk_replay.cpp`),
  `RX_VK_TL` (`tl_ev`, `tl_at_present`; its events include the recordings sent, the flushes, the late fixes and the queue submit's
  start and end), `HOS_RB_LR_LOG` (`nvn_native.c`). Scratch scripts, not in the repository, under `E:\OOT\scratch-from-c\r2`:
  `tlshow.py` (prints a `vk: tl` dump), `sym.py` (resolves the RB_PROF call sites against the link map), `lumastat.py` (mean luma per
  frame of a frame log), `flrun.sh` (a run with the frame log for 25 s), `ab.sh` (interleaved arms).
- Next, in order: (1) log the queue submit's duration and the gap between a recording's submit and the GPU's start (the events are in:
  run `RX_VK_TL_S=78` and read `rec-qsubmit+/-` against the timeline's counter); (2) if the heavy submission's recordings reach the
  queue late, submit the parallel replay's segments as they finish instead of together, or record the first segment short, so that the
  GPU's 3 ms overlaps the replay's 4-8; the readbacks' wait then ends sooner and the frame with it; (3) register the meter's
  write-back per recording instead of per submission, so a readback waits for the piece that wrote it; (4) explain the stalled runs
  before `WATCH_KICK` is adopted.
- Speed rows are in this document; the raw runs are in `build/s5/r2ab1-*`, `r2ab2-*`, `r2ab3-*` and `r2-fl-*` (local).
