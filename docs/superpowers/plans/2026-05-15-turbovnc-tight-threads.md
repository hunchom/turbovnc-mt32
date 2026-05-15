# TurboVNC Configurable Tight Encoding Threads — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Raise the server-side multithreaded Tight encoder thread cap from 4 to a configurable 32, race-free, with dynamic per-frame scaling.

**Architecture:** Per-thread private zlib contexts replace the 4 shared per-client streams; the 4 wire stream IDs are reused round-robin with a stream-reset handshake at each thread boundary; `nt <= 4` stays byte-identical to stock. One client-side viewer line is required for the reset handshake.

**Tech Stack:** C (X.org / TurboVNC `Xvnc` server), zlib, pthreads; Java (TurboVNC viewer); CMake/RPM build.

**Source tree:** `/Users/rogerfrench/turbovnc-src` (clone of `github.com/TurboVNC/turbovnc`, VERSION 3.3.1).
**Server source dir:** `unix/Xvnc/programs/Xserver/hw/vnc/`.

**Task order (hard dependency):** A → B → C → D. Section A's struct/cap changes must land before B/C reference the new fields. All before/after blocks are exact-match anchors — apply by text match, not line number, if earlier tasks shift lines.

---

## Section A — Structures & thread lifecycle (`rfb.h`, `tight.c`)

### Task A1: Raise the thread cap

**Files:** Modify `unix/Xvnc/programs/Xserver/hw/vnc/rfb.h:88-90`

- [ ] **Step 1: Change the cap.** Before:
```c
/* Maximum number of threads to use for multithreaded encoding, regardless of
   the CPU count */
#define MAX_ENCODING_THREADS 4
```
After:
```c
/* Maximum number of threads to use for multithreaded encoding, regardless of
   the CPU count */
#define MAX_ENCODING_THREADS 32
```

The only `MAX_ENCODING_THREADS`-sized arrays are `tight.c` `thnd[]` and `tparam[]` — both file-scope statics, grow safely in `.bss`.

### Task A2: Fix the `thnd[]` initializer

**Files:** Modify `tight.c:131`

- [ ] **Step 1.** Before: `static pthread_t thnd[MAX_ENCODING_THREADS] = { 0, 0, 0, 0 };`
After: `static pthread_t thnd[MAX_ENCODING_THREADS] = { 0 };`

### Task A3: Add per-thread zlib fields to `threadparam`

**Files:** Modify `tight.c` `threadparam` struct (lines ~133-151)

- [ ] **Step 1.** Insert five fields after the `int streamId, baseStreamId, nStreams;` line. Before:
```c
  int streamId, baseStreamId, nStreams;
  pthread_mutex_t ready, done;
```
After:
```c
  int streamId, baseStreamId, nStreams;
  /* per-thread zlib state -> nt>4 path. each thread deflates own zs[] only. */
  z_stream zs[4];
  Bool zsActive[4];
  int zsLevel[4];
  Bool resetStream[4];     /* next subrect on wire id k must carry reset bit */
  unsigned int clientGen;  /* generation of client these zs belong to */
  pthread_mutex_t ready, done;
```
`z_stream` is in scope (`rfb.h` pulls `<zlib.h>`). The `[4]` is the wire-stream count — never scale it to `MAX_ENCODING_THREADS`.

### Task A4: Per-thread zlib lifecycle

**Files:** Modify `tight.c` `InitThreads` (~289) and `ShutdownTightThreads` (~324)

- [ ] **Step 1: Document lazy init in `InitThreads`.** After the `memset(tparam, 0, ...)` line add:
```c
  /* zs[] left zeroed -> lazy deflateInit2 in CompressData on first use. */
```
Lazy init avoids ~32 MB of zlib heap for 128 streams most frames never use; mirrors the stock lazy `cl->zsStruct` path.

- [ ] **Step 2: `deflateEnd` per-thread streams in `ShutdownTightThreads`.** In the cleanup loop, before `memset(&tparam[i], 0, ...)`. Before:
```c
  for (i = 0; i < rfbNumThreads; i++) {
    free(tparam[i].tightAfterBuf);
    free(tparam[i].tightBeforeBuf);
    if (i != 0) free(tparam[i].updateBuf);
    if (tparam[i].j) tjDestroy(tparam[i].j);
    if (!REGION_NAR(&tparam[i].losslessRegion))
      REGION_UNINIT(pScreen, &tparam[i].losslessRegion);
    if (!REGION_NAR(&tparam[i].lossyRegion))
      REGION_UNINIT(pScreen, &tparam[i].lossyRegion);
    memset(&tparam[i], 0, sizeof(threadparam));
  }
```
After:
```c
  for (i = 0; i < rfbNumThreads; i++) {
    int k;
    free(tparam[i].tightAfterBuf);
    free(tparam[i].tightBeforeBuf);
    if (i != 0) free(tparam[i].updateBuf);
    if (tparam[i].j) tjDestroy(tparam[i].j);
    /* deflateEnd active per-thread streams before memset -> no zlib heap leak */
    for (k = 0; k < 4; k++) {
      if (tparam[i].zsActive[k]) {
        deflateEnd(&tparam[i].zs[k]);
        tparam[i].zsActive[k] = FALSE;
      }
    }
    if (!REGION_NAR(&tparam[i].losslessRegion))
      REGION_UNINIT(pScreen, &tparam[i].losslessRegion);
    if (!REGION_NAR(&tparam[i].lossyRegion))
      REGION_UNINIT(pScreen, &tparam[i].lossyRegion);
    memset(&tparam[i], 0, sizeof(threadparam));
  }
```
Single-entry per init cycle (`!threadInit` guard) → `deflateEnd` runs exactly once per active stream, no double-free.

- [ ] **Step 3: Build-verify.** `cd /Users/rogerfrench/turbovnc-src && mkdir -p build && cd build && cmake -G"Unix Makefiles" .. && make` — clean compile, no new warnings.

- [ ] **Step 4: Commit.** `feat(tight): raise thread cap to 32 and add per-thread zlib state`

---

## Section B — zlib stream parallelism (`tight.c`)

### Task B1: Zero-height strip guard

**Files:** Modify `tight.c:408-409`

- [ ] **Step 1.** Before:
```c
  nt = min(rfbNumThreads, w * h / MAXRECTSIZE);
  if (nt < 1) nt = 1;
```
After:
```c
  nt = min(rfbNumThreads, w * h / MAXRECTSIZE);
  nt = min(nt, h);                    /* never a 0-height strip */
  if (nt < 1) nt = 1;
```
With `nt <= h`, integer `h/nt >= 1` → every strip has ≥1 row.

- [ ] **Step 2: Commit.** `fix(tight): clamp Tight thread count to rectangle height`

### Task B2: Stream-assignment block for nt>4

**Files:** Modify `tight.c:419-435`

- [ ] **Step 1: Clear `resetStream` per-frame.** After the `rfbAutoLosslessRefresh` `REGION_INIT` block, before the stream-assignment comment, insert:
```c
    tparam[i].resetStream[0] = tparam[i].resetStream[1] =
      tparam[i].resetStream[2] = tparam[i].resetStream[3] = FALSE;
```

- [ ] **Step 2: Replace the assignment block.** Before:
```c
    /* Tight encoding has only a limited number of zlib streams (4).  The
       streams must all be left open as long as the client is connected, or
       performance suffers.  Thus, multiple threads can't use the same zlib
       stream.  We divide the pool of 4 evenly among the available threads (up
       to the first 4 threads), and if each thread has more than one stream, it
       cycles between them in a round-robin fashion. */
    if (i < 4) {
      int n = min(nt, 4);
      tparam[i].baseStreamId = 4 / n * i;
      if (i == n - 1) tparam[i].nStreams = 4 - tparam[i].baseStreamId;
      else tparam[i].nStreams = 4 / n;
      tparam[i].streamId = tparam[i].baseStreamId;
    }
```
After:
```c
    /* Tight has only 4 wire zlib stream ids (0..3).  nt<=4: divide the 4 ids
       disjointly, streams persist cross-frame, no resets -> wire output
       byte-identical to stock.  nt>4: pin thread i to wire id i%4, deflate
       through its own t->zs, reset that id before the thread's first zlib
       subrect so the client realigns its inflate context. */
    if (nt <= 4) {
      if (i < 4) {
        int n = min(nt, 4);
        tparam[i].baseStreamId = 4 / n * i;
        if (i == n - 1) tparam[i].nStreams = 4 - tparam[i].baseStreamId;
        else tparam[i].nStreams = 4 / n;
        tparam[i].streamId = tparam[i].baseStreamId;
      }
    } else {
      int wireId = i % 4;
      tparam[i].baseStreamId = wireId;       /* single wire id */
      tparam[i].nStreams = 0;                /* no round-robin */
      tparam[i].streamId = wireId;
      /* threads 0..3 first-claim ids 0..3; i>=4 shares an id -> reset */
      tparam[i].resetStream[wireId] = (i >= 4) ? TRUE : FALSE;
    }
```

- [ ] **Step 2: Commit.** `feat(tight): map threads to wire stream ids when nt > 4`

### Task B3: `CompressData` — per-thread zlib + client-generation guard

**Files:** Modify `tight.c` `CompressData` (~1126-1176)

- [ ] **Step 1: Add generation guard + retarget to `t->zs`.** Before:
```c
  if (zlibLevel == 0 && cl->enableTightWithoutZlib)
    return SendCompressedData(t, t->tightBeforeBuf, dataLen);

  pz = &cl->zsStruct[streamId];

  /* Initialize compression stream if needed. */
  if (!cl->zsActive[streamId]) {
    pz->zalloc = Z_NULL;
    pz->zfree = Z_NULL;
    pz->opaque = Z_NULL;

    err = deflateInit2(pz, zlibLevel, Z_DEFLATED, MAX_WBITS, MAX_MEM_LEVEL,
                       zlibStrategy);
    if (err != Z_OK)
      return FALSE;

    cl->zsActive[streamId] = TRUE;
    cl->zsLevel[streamId] = zlibLevel;
  }
```
After:
```c
  if (zlibLevel == 0 && cl->enableTightWithoutZlib)
    return SendCompressedData(t, t->tightBeforeBuf, dataLen);

  /* tparam[] is process-global -> a thread's zs may carry a prior client's
     deflate history.  on client change, end every active stream. */
  if (t->clientGen != cl->generation) {
    int s;
    for (s = 0; s < 4; s++) {
      if (t->zsActive[s]) {
        deflateEnd(&t->zs[s]);
        t->zsActive[s] = FALSE;
      }
    }
    t->clientGen = cl->generation;
  }

  pz = &t->zs[streamId];

  /* Initialize compression stream if needed. */
  if (!t->zsActive[streamId]) {
    pz->zalloc = Z_NULL;
    pz->zfree = Z_NULL;
    pz->opaque = Z_NULL;

    err = deflateInit2(pz, zlibLevel, Z_DEFLATED, MAX_WBITS, MAX_MEM_LEVEL,
                       zlibStrategy);
    if (err != Z_OK)
      return FALSE;

    t->zsActive[streamId] = TRUE;
    t->zsLevel[streamId] = zlibLevel;
  }
```

- [ ] **Step 2: Retarget the level check.** Before:
```c
  if (zlibLevel != cl->zsLevel[streamId]) {
    if (deflateParams(pz, zlibLevel, zlibStrategy) != Z_OK)
      return FALSE;
    cl->zsLevel[streamId] = zlibLevel;
  }
```
After:
```c
  if (zlibLevel != t->zsLevel[streamId]) {
    if (deflateParams(pz, zlibLevel, zlibStrategy) != Z_OK)
      return FALSE;
    t->zsLevel[streamId] = zlibLevel;
  }
```

- [ ] **Step 3: Clear the reset flag after a successful deflate.** After:
```c
  if (deflate(pz, Z_SYNC_FLUSH) != Z_OK || pz->avail_in != 0 ||
      pz->avail_out == 0)
    return FALSE;
```
insert:
```c
  /* reset bit emitted in caller's control byte; fresh deflateInit2 above
     realigned server+client streams -> clear so later subrects don't re-reset */
  t->resetStream[streamId] = FALSE;
```

- [ ] **Step 4: Commit.** `feat(tight): give each Tight worker its own zlib contexts`

### Task B4: Emit the reset bit in the control bytes

**Files:** Modify `tight.c` at the three control-byte writes (~978, ~1046, ~1111)

Protocol confirmed: `common/rfb/rfbproto.h:828-846` — compression-control byte low nibble bits 0-3 are per-stream reset flags; `rfbproto.h:943-946` requires the decoder to reset on those bits. Wire id is bits 4-5. Standard Tight, nothing invented.

- [ ] **Step 1: `SendMonoRect` (~978).** Before:
```c
  else
    t->updateBuf[(*t->ublen)++] = (streamId | rfbTightExplicitFilter) << 4;
```
After:
```c
  else
    t->updateBuf[(*t->ublen)++] =
      (char)(((streamId | rfbTightExplicitFilter) << 4) |
             (t->resetStream[streamId] ? (1 << streamId) : 0));
```

- [ ] **Step 2: `SendIndexedRect` (~1046).** Same before/after as Step 1 (identical line text).

- [ ] **Step 3: `SendFullColorRect` (~1111).** Before:
```c
  else
    t->updateBuf[(*t->ublen)++] = streamId << 4;
```
After:
```c
  else
    t->updateBuf[(*t->ublen)++] =
      (char)((streamId << 4) |
             (t->resetStream[streamId] ? (1 << streamId) : 0));
```

For `nt <= 4`, `resetStream[]` is always `FALSE` → the OR contributes 0 → byte-identical to stock. JPEG/fill control bytes emit no zlib data → intentionally not modified.

- [ ] **Step 4: Commit.** `feat(tight): emit stream-reset bit on wire-id handoff`

---

## Section C — Config surface, build, docs (`rfb.h`, `rfbserver.c`, `init.c`, man, RPM)

### Task C1: `generation` field on `rfbClientRec`

**Files:** Modify `rfb.h` (~416-421)

- [ ] **Step 1.** Before:
```c
  /* tight encoding -- preserve zlib streams' state for each client */

  z_stream zsStruct[4];
```
After:
```c
  /* tight encoding -- preserve zlib streams' state for each client */

  unsigned int generation;          /* monotonic client-identity token */
  z_stream zsStruct[4];
```
`cl->zsStruct` stays — ZRLE/Zlib encoders still use it.

### Task C2: Global generation counter + raised default (`rfbserver.c`)

**Files:** Modify `rfbserver.c` at ~95, ~405-406, ~533

- [ ] **Step 1: Global.** Before: `int rfbNumThreads = 0;`
After:
```c
int rfbNumThreads = 0;
unsigned int rfbClientGeneration = 0;  /* bumped per new client */
```

- [ ] **Step 2: Stamp `cl->generation` in `rfbNewClient`.** Before:
```c
  cl->id = rfbClientNumber++;
  if (rfbClientNumber == 0) rfbClientNumber = 1;
```
After:
```c
  cl->id = rfbClientNumber++;
  if (rfbClientNumber == 0) rfbClientNumber = 1;
  cl->generation = ++rfbClientGeneration;
```
Pre-increment → first client gets 1; a worker's zeroed `clientGen` (0) always mismatches → forced reset on first frame. Monotonic → ABA-safe.

- [ ] **Step 3: Raise the default.** Before: `else if (rfbNumThreads < 1) rfbNumThreads = min(np, 4);`
After: `else if (rfbNumThreads < 1) rfbNumThreads = min(np, MAX_ENCODING_THREADS);`
The `> np` clamp and the `TVNC_NTHREADS` / `-nthreads` overrides (already `MAX_ENCODING_THREADS`-bounded) are untouched.

- [ ] **Step 4: Commit.** `feat: raise default Tight thread count, add client generation counter`

### Task C3: `-nthreads` help text (`init.c`)

**Files:** Modify `init.c` (~1900)

- [ ] **Step 1.** Before: `  ErrorF("                       max. 4]\n");`
After: `  ErrorF("                       max. %d]\n", MAX_ENCODING_THREADS);`
(The `1 <= N <= %d` line already uses `MAX_ENCODING_THREADS` — auto-corrects.)

### Task C4: Man page (`Xvnc.man.in`)

**Files:** Modify `unix/Xvnc/programs/Xserver/Xvnc.man.in` (~373-379)

- [ ] **Step 1.** Replace the `-nthreads` paragraph. Before:
```
default is to use one thread per CPU core, up to a maximum of 4 (because using
more than 4 encoding threads breaks compatibility with viewers other than the
TurboVNC Viewer.)  The server will not allow the thread count to exceed 4, nor
to exceed the number of CPU cores.
```
After:
```
default is to use one thread per CPU core, up to a maximum of 32.  The server
will not allow the thread count to exceed 32, nor to exceed the number of CPU
cores.  Using more than 4 encoding threads requires the TurboVNC Viewer; other
viewers support a maximum of 4 Tight zlib streams.
```

### Task C5: RPM version marker (`CMakeLists.txt`)

**Files:** Modify `CMakeLists.txt:9-10`

- [ ] **Step 1.** Before:
```
set(VERSION 3.3.1)
set(DOCVERSION 3.3.1)
```
After:
```
set(VERSION 3.3.1.mt32)
set(DOCVERSION 3.3.1.mt32)
```
Dot, not dash — RPM rejects `-` in the `Version:` field. `3.3.1.mt32` sorts after `3.3.1`.

- [ ] **Step 2: Commit.** `chore: docs, help text, and version marker for mt32 build`

---

## Section D — Viewer reset-bit fix (`TightDecoder.java`) — REQUIRED

The viewer reads the reset bits but calls `inflater[i].end()` (destroys the Inflater) instead of `.reset()`. Dead code in stock (server never emits reset bits at runtime); the nt>4 path makes it live → `IllegalStateException` on the next `inflate()`. One-line fix.

### Task D1: Fix the reset handler

**Files:** Modify `java/com/turbovnc/rfb/TightDecoder.java:144`

- [ ] **Step 1.** Before:
```java
    // Flush zlib streams if we are told by the server to do so.
    for (int i = 0; i < 4; i++) {
      if ((compCtl & 1) != 0)
        inflater[i].end();
      compCtl >>= 1;
    }
```
After:
```java
    // Flush zlib streams if we are told by the server to do so.
    for (int i = 0; i < 4; i++) {
      if ((compCtl & 1) != 0)
        inflater[i].reset();
      compCtl >>= 1;
    }
```
`Inflater.reset()` = `inflateReset` (keeps the stream alive, re-armed) — the correct flush. Matches the viewer's own `reset()` method (line 63).

- [ ] **Step 2: Commit.** `fix(viewer): reset Tight inflater instead of destroying it`

---

## Section E — Verification (run on `rocky_8_10_vm`)

### Task E1: Clean build
- [ ] cmake Release build, server + viewer, on Rocky 8.10. `Xvnc -version` shows `3.3.1.mt32`.

### Task E2: ThreadSanitizer
- [ ] clang `-fsanitize=thread` build. Run sessions at `-nthreads` 8/16/32 with large-area motion (≥1900×1000) so all strips spawn. **Pass = zero data races mentioning `deflate`/`CompressData`/`z_stream`/`threadparam`.**

### Task E3: AddressSanitizer
- [ ] clang `-fsanitize=address,undefined` build. Exercise: window resize, client disconnect/reconnect ×10, two simultaneous clients, auto-lossless-refresh. Adversarial: interleave reconnect with two clients (ABA window for the generation guard). **Pass = no ASan/UBSan errors.**

### Task E4: Edge cases
- [ ] Tiny window `h < nthreads` (200×16, 200×4, 1×1 at `-nthreads 32`). **Pass = no SIGFPE, no overflow, viewer renders correctly.**

### Task E5: `nt<=4` wire-identical
- [ ] Capture wire stream (stock vs patched) at `-nthreads` 1/2/4 under a deterministic workload. **Pass = identical Tight rectangle payloads.**

### Task E6: Benchmark
- [ ] `nthreads` 1/2/4/8/16/32 throughput table via `TVNC_PROFILE=1`. Knee expected ~8-16. GPU numbers re-run by the user on the real AMD box.

### Task E7: Functional decode
- [ ] Patched viewer + patched server at `-nthreads 32` — sustained motion 5 min, no crash, no visual corruption (confirms the reset handshake end-to-end).

---

## Self-Review

- **Spec coverage:** every spec §5 impacted region maps to a task (A1/A2 cap+init, A3 struct, A4 lifecycle, B1 guard, B2 assignment, B3 CompressData, B4 control bytes, C1-C4 config/docs, C5 RPM). Spec §9 viewer item → resolved as Section D (was "verify", upgraded to a required fix by the finding). Verification §7 → Section E.
- **Placeholder scan:** none — all before/after blocks are verbatim source.
- **Type consistency:** `zs`/`zsActive`/`zsLevel`/`resetStream`/`clientGen` (threadparam), `generation` (rfbClientRec), `rfbClientGeneration` (global) — used identically across A/B/C. Literal `4` = wire-stream count everywhere (never `MAX_ENCODING_THREADS`).
- **Scope change vs spec:** spec §2 said "no client code change required" — Section D's one viewer line overrides that. Documented.
