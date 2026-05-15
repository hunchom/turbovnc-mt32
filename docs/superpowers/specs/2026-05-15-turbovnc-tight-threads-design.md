# TurboVNC — Configurable Tight Encoding Thread Count (4 → up to 32)

**Date:** 2026-05-15
**Status:** Approved design — ready for implementation planning
**Source tree:** `github.com/TurboVNC/turbovnc` (cloned at `~/turbovnc-src`)

## 1. Goal

Raise the maximum number of threads used by the server-side multithreaded
Tight encoder from a hardcoded 4 to a configurable cap of 32, with:

1. No additional lock/stream contention.
2. Real, measured speedup — not just a higher number.
3. Per-frame dynamic thread scaling (auto-pick count, manual override kept).
4. Every impacted code region updated.
5. No segfault, no protocol corruption, no compression-ratio regression for
   the `nt <= 4` case.

## 2. Environment

- Server: RHEL 8, GNOME Shell + gnome-session under VirtualGL, AMD Radeon GPU,
  80 physical cores, 70 GB RAM.
- Client: TurboVNC viewer, Tight encoding. **Client decodes only — no client
  code change is required by this design.**
- Deploy: rebuild the official TurboVNC RPM, install the intended way.

## 3. Root cause — why the limit is 4

The Tight encoder splits one framebuffer rectangle into `nt` horizontal
strips, one worker thread per strip (`tight.c:rfbSendRectEncodingTight`,
~line 391). Each strip's basic-compression subrects are deflated through a
**zlib stream**.

The RFB Tight wire protocol defines exactly **4 zlib stream IDs** (0–3). The
server holds 4 `z_stream` structs in the per-client record:

```
rfb.h:419   z_stream zsStruct[4];
rfb.h:420   Bool     zsActive[4];
rfb.h:421   int      zsLevel[4];
```

`tight.c:rfbSendRectEncodingTight` (~lines 425–435) divides those 4 streams
across `min(nt,4)` threads via `baseStreamId` / `nStreams` / `streamId`. With
`nt <= 4` each thread owns a disjoint subset → no two threads ever touch the
same `z_stream`.

With `nt > 4`, the assignment block is gated by `if (i < 4)`. Threads with
`i >= 4` keep the zeroed defaults: `baseStreamId = 0`, `nStreams = 0`,
`streamId = 0`. In `CompressData` (`tight.c:1126`) `pz = &cl->zsStruct[streamId]`
→ every thread `i >= 4` deflates into `cl->zsStruct[0]` **concurrently** with
thread 0. Concurrent `deflate()` on one `z_stream` corrupts its internal heap
state → segfault or garbage on the wire.

`MAX_ENCODING_THREADS` = 4 (`rfb.h:90`) exists to prevent exactly this.

## 4. Approach — per-thread zlib contexts + reset-on-handoff

Selected over two rejected alternatives:

- **Rejected: bump the cap, keep dividing the 4 shared streams.** Causes the
  corruption in §3.
- **Rejected: cap zlib subrects at 4, scale only JPEG past 4.** A strip
  contains mixed content; cannot cleanly separate zlib vs JPEG work per strip,
  and GNOME UI (zlib-heavy) would get no benefit.

### 4.1 Cap

`MAX_ENCODING_THREADS`: 4 → 32 (`rfb.h:90`). Runtime still clamps to the
online CPU count.

### 4.2 Safety invariant — `nt <= 4` is bit-identical to stock

When `nt <= 4`, threads use disjoint wire stream IDs, never reset a stream,
and streams persist across frames — exactly as stock TurboVNC. The patched
server produces byte-identical wire output to stock whenever ≤4 threads run.
**All new risk is confined to the `nt > 4` code path.** This invariant is the
backbone of the safety argument and must be preserved by every change.

### 4.3 `nt > 4` path

1. **Per-thread zlib state.** Move the 4 `z_stream`s from the shared
   per-client `cl->zsStruct` into the per-thread `threadparam`:
   ```
   z_stream zs[4];
   Bool     zsActive[4];
   int      zsLevel[4];
   ```
   `threadparam tparam[]` is a process-global static array, so per-thread
   streams persist across frames (preserving cross-frame compression history
   when a thread keeps its strip). `CompressData` targets `t->zs[...]` instead
   of `cl->zsStruct[...]`. No shared server-side deflate state → race-free by
   construction. `cl->zsStruct` is left in place for the ZRLE / Zlib encodings
   that also use it — those are untouched.

2. **Wire stream ID reuse.** Only 4 wire IDs exist. Threads map
   `wireId = i % 4`. Worker output buffers are concatenated in fixed thread
   order (0,1,2,…) — already true in stock — so all bytes for a given wire ID
   are contiguous and ordered on the wire within a frame.

3. **Reset on handoff.** When thread A and thread B both emit on `wireId k`,
   thread B's `z_stream` is an independent deflate history. B must signal the
   client to reinitialize its inflate context for stream `k` before B's data.
   The Tight compression-control byte's low nibble (bits 0–3) are per-stream
   reset flags; emitting `1 << wireId` forces the client reset. Each thread in
   the `nt > 4` path resets its wire stream(s) on the first subrect of its
   strip.

4. **Multi-client / reconnect guard.** Because `tparam[]` is process-global,
   a thread's `zs` may be reused by a different client. Inheriting another
   client's deflate history corrupts decoding. Guard: store a client
   *generation token* (monotonic counter, not the raw `rfbClientPtr` — avoids
   ABA when a freed client struct is re-malloc'd at the same address) on each
   thread; when it changes, reset that thread's streams. Single-client steady
   state incurs zero extra resets.

### 4.4 Tradeoff (accepted)

`nt > 4` loses cross-*frame* zlib history on shared wire streams → a small
compression-ratio loss for *static* screen content. Self-limiting: static
content produces small rectangles → dynamic scaling (§4.5) keeps `nt` low →
the `nt > 4` path only engages on large motion frames, which carry no useful
temporal history anyway. For the VirtualGL-composited workload this is a
non-issue.

### 4.5 Dynamic thread scaling

- Default `rfbNumThreads` when `-nthreads` / `TVNC_NTHREADS` are unset:
  `min(np, 4)` → `min(np, MAX_ENCODING_THREADS)` (`rfbserver.c:533`).
- Per-frame count `nt = min(rfbNumThreads, w*h / MAXRECTSIZE)` (`tight.c:408`)
  is unchanged — it already scales the count *down* for small rectangles. This
  is the dynamic engine; small updates never over-spawn → no contention.
- Add guard `nt = min(nt, h)` — never create a 0-height strip (latent bug at
  high thread counts; `h / nt` can round to 0).
- `-nthreads N` / `TVNC_NTHREADS=N` still force a fixed N (override).
  `-nthreads 0` is accepted as explicit auto.

## 5. Impacted regions — complete list

| # | File | Region | Change |
|---|------|--------|--------|
| 1 | `unix/Xvnc/.../hw/vnc/rfb.h` | line 90 | `MAX_ENCODING_THREADS` 4 → 32 |
| 2 | `tight.c` | line 131 | `thnd[]={0,0,0,0}` → `{0}` (must zero-fill all 32) |
| 3 | `tight.c` | `threadparam` struct (~133) | add `zs[4]`, `zsActive[4]`, `zsLevel[4]`, reset flag, client-gen token |
| 4 | `tight.c` | `InitThreads` (~289) | per-thread zlib state init |
| 5 | `tight.c` | `ShutdownTightThreads` (~324) | `deflateEnd` each active per-thread stream |
| 6 | `tight.c` | line 408 | add `nt = min(nt, h)` guard |
| 7 | `tight.c` | lines 425–435 | stream-assignment block extended for `nt > 4` |
| 8 | `tight.c` | `CompressData` (~1126) | target `t->zs`; emit reset; client-gen guard |
| 9 | `tight.c` | control bytes at ~978, ~1046, ~1111 | OR in reset bit when required |
| 10 | `rfbserver.c` | lines 533–536 | default raised; clamp logic |
| 11 | `init.c` | lines 1897–1900 | `-nthreads` help text (max 4 → 32) |
| 12 | `doc/Xvnc.man` | `-nthreads` entry | document new max + dynamic default |
| 13 | java viewer Tight decoder | — | **read-only verification** it honors stream-reset bits |
| 14 | `release/` (RPM spec / version) | — | bump version string so the rebuilt RPM is identifiable |

## 6. Correctness arguments

- **Race-freedom (`nt > 4`):** each thread deflates only into its own
  `t->zs[]`. No shared mutable deflate state. The only shared structures
  (output concatenation, byte counters) are already serialized in stock via
  the per-thread `done` mutex join.
- **Wire sync:** for any wire ID `k`, server bytes are contiguous and
  thread-ordered; each producing thread resets `k` before its data; client
  inflate context `k` is therefore realigned at every thread boundary.
- **Multi-client:** client-generation guard forces a reset when a thread's
  `zs` crosses a client boundary.
- **`nt <= 4`:** disjoint streams, no resets — provably identical to stock.
- **Edge cases:** `min(nt, h)` prevents 0-height strips; `min(nt, np)`
  prevents oversubscription; `w*h / MAXRECTSIZE` prevents over-splitting small
  rectangles.

## 7. Verification plan

Performed before delivery, on the Rocky 8.10 build VM (`rocky_8_10_vm`):

1. Clean `cmake` build, RHEL8/Rocky8 toolchain.
2. `nt <= 4` wire-identical: argued from §4.2 + smoke test against stock.
3. **ThreadSanitizer** build, sessions at `nt` = 8, 16, 32 → zero data races.
4. **ASan** build, exercise: window resize, client disconnect/reconnect, two
   simultaneous clients, auto-lossless-refresh enabled.
5. Edge cases: tiny window (`h < nt`), `w*h / MAXRECTSIZE` rounding boundaries.
6. Benchmark `glxspheres` fullscreen at `nt` = 1, 2, 4, 8, 16, 32 → speedup
   curve and the knee (expected knee ~8–16). GPU benchmark numbers to be
   re-run by the user on the real AMD Radeon box.

## 8. Deliverable

- Exact per-file diff hunks (line numbers + before/after), formatted for
  manual paste into GitHub / hand-copy.
- Reproduce / build doc: how to apply the changes, rebuild the RPM, install.
- Benchmark results table from §7.6.

A merged branch / PR is **not** required — the user hand-copies the changes.

## 9. Open items for the planning phase to verify

- Confirm the TurboVNC Java/native viewer Tight decoder honors control-byte
  low-nibble stream-reset bits (expected: yes — standard Tight). If not:
  fallback caps the zlib path at 4 threads while the JPEG path still scales.
- Confirm exact `MAXRECTSIZE` value per compress level and whether a minimum
  pixels-per-thread floor needs tuning for 32-way splits.
- Confirm no other encoder (ZRLE, Zlib) shares state disturbed by moving
  Tight's streams out of `cl->zsStruct`.
- Confirm worker pthread stack sizing at 32 threads is acceptable (optionally
  set a smaller stack via `pthread_attr_setstacksize`).
