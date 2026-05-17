# turbovnc-mt32

Patch for the [TurboVNC](https://github.com/TurboVNC/turbovnc) `3.3.1` server
that raises the multithreaded **Tight encoding** thread cap from a hardcoded
**4** to a configurable **32**, with per-frame dynamic scaling and no zlib
stream corruption.

## Why

The Tight wire protocol has only 4 zlib stream IDs; stock TurboVNC shares 4
`z_stream`s per client, so >4 encoder threads would `deflate()` the same
stream concurrently → heap corruption. This patch gives every thread its own
zlib contexts and reuses the 4 wire IDs with a per-frame stream-reset
handshake, so the cap can safely go to 32. A wire stream is reset whenever
its cross-frame continuity breaks (>4 threads, thread-count change, or client
change); a sticky reset flag guarantees no reset is ever missed.

## Verified

Built and run on Rocky 8 / RHEL 8 (`Xvnc 3.3.1.mt32`):

- Real **32 threads** at 2560×1440, 198k+ Tight rectangles — **0 decode errors**
- **9-minute hard-transition stress, 2 simultaneous clients**, 62k+ rectangles — **0 decode errors**
- **AddressSanitizer** clean under real-32-thread load (no corruption / leak / UAF)
- Clean across `-nthreads` 1/2/4/8/16/32, client reconnects, and tiny windows
- Benchmark: **3.15× faster** full-screen encode at 4 threads (8-core VM)

## Contents

| Path | What |
|------|------|
| [`TURBOVNC-32-THREADS-GUIDE.md`](TURBOVNC-32-THREADS-GUIDE.md) | **Start here.** Step-by-step guide: every change before/after, build, install, and how to set the thread count in `turbovncserver.conf` |
| `turbovnc-mt32.patch` | Combined patch — `git apply` against a `3.3.1` checkout |
| `patches/` | One patch file per changed source file |
| `patched-files/` | The 7 modified files in full, with original path layout |
| `docs/superpowers/specs/` | Design spec |
| `docs/superpowers/plans/` | Implementation plan |

## Quick apply

```bash
git clone https://github.com/TurboVNC/turbovnc.git
cd turbovnc && git checkout 3.3.1
git apply /path/to/turbovnc-mt32.patch
```

Then build the RPM and set `$serverArgs = "-nthreads 32";` in
`turbovncserver.conf` — see the guide.

## Files changed

7 files, ~90 lines: `rfb.h`, `tight.c`, `rfbserver.c`, `init.c`,
`Xvnc.man.in`, `CMakeLists.txt`, and `TightDecoder.java` (a required
one-line viewer fix).
