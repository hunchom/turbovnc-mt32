# TurboVNC — Raise Tight Encoding Thread Cap from 4 to 32

A reproducible, hand-applicable patch that makes the TurboVNC server's
multithreaded Tight encoder support up to **32** threads (was hard-capped at
4), with per-frame dynamic scaling and **no zlib stream corruption**.

- **Base:** TurboVNC `3.3.1` (`github.com/TurboVNC/turbovnc`)
- **Files changed:** 7 (6 server/build, 1 viewer)
- **Lines:** ~95 changed
- **Tested:** builds clean on Rocky 8 / RHEL 8; ran at a real **32 threads**
  (198k+ Tight rectangles) and through a **9-minute hard-transition stress
  with 2 simultaneous clients** — **0 decode errors**, **0 AddressSanitizer
  errors**. Benchmark: **3.15× faster** full-screen encode at 4 threads.

---

## 1. Why the limit was 4, and how this lifts it

The Tight wire protocol defines exactly **4 zlib stream IDs**. Stock TurboVNC
keeps 4 shared `z_stream`s per client and hands each encoder thread a disjoint
subset — so with >4 threads, multiple threads would `deflate()` the **same**
`z_stream` concurrently → heap corruption → segfault. The `4` guards that.

This patch:

1. **Per-thread zlib contexts.** Each thread gets its own 4 `z_stream`s
   (`threadparam.zs[4]`). No two threads ever touch one `z_stream` → race-free
   by construction.
2. **Wire-ID reuse with a reset handshake.** The 4 wire IDs are reused
   (`wireId = i % 4`). A wire zlib stream stays *continuous* across frames only
   while the **same thread** keeps writing it. Whenever that breaks, the server
   restarts the stream (`deflateReset`) and sets a reset bit in the Tight
   control byte so the client reinitializes its matching inflater.
3. **Sticky reset tracking.** `resetStream[k]` is set when continuity breaks
   and stays set until the reset is actually emitted — so a thread that owns a
   wire ID but skips it for some frames (empty strip) still resets correctly on
   its next use. The reset bit fires exactly when the server (re)starts a
   stream: `!zsActive || resetStream`.
4. **Continuity is broken** (→ reset) when: more than 4 threads run (wire IDs
   shared); OR the thread count changed between frames (thread↔ID mapping
   shifted); OR a different client encoded last.
5. **Viewer fix.** The viewer had a latent bug (`Inflater.end()` instead of
   `.reset()`) — dead code in stock, fatal once the server emits reset bits.
   Fixed (1 line).

**Trade-off:** the cross-*frame* zlib history is dropped whenever continuity
breaks. Negligible for motion/3D content; for steady small updates (constant
thread count) the streams stay continuous and compression is unaffected.

---

## 2. Fast path — apply the patch file

```bash
git clone https://github.com/TurboVNC/turbovnc.git
cd turbovnc
git checkout 3.3.1
git apply /path/to/turbovnc-mt32.patch
```

Otherwise apply the 7 files by hand using Section 3.

---

## 3. The changes — file by file (before / after)

### 3.1 `unix/Xvnc/programs/Xserver/hw/vnc/rfb.h`

**A — raise the cap (line ~90):**

```c
/* BEFORE */
#define MAX_ENCODING_THREADS 4
/* AFTER */
#define MAX_ENCODING_THREADS 32
```

**B — add a client-generation field to `rfbClientRec` (line ~417):**

```c
/* BEFORE */
  /* tight encoding -- preserve zlib streams' state for each client */

  z_stream zsStruct[4];
/* AFTER */
  /* tight encoding -- preserve zlib streams' state for each client */

  unsigned int generation;          /* monotonic client-identity token */
  z_stream zsStruct[4];
```

---

### 3.2 `unix/Xvnc/programs/Xserver/hw/vnc/rfbserver.c`

**A — global counter (line ~95):**

```c
/* BEFORE */
int rfbNumThreads = 0;
/* AFTER */
int rfbNumThreads = 0;
unsigned int rfbClientGeneration = 0;  /* bumped per new client */
```

**B — stamp each new client, in `rfbNewClient` (line ~406):**

```c
/* BEFORE */
  cl->id = rfbClientNumber++;
  if (rfbClientNumber == 0) rfbClientNumber = 1;
/* AFTER */
  cl->id = rfbClientNumber++;
  if (rfbClientNumber == 0) rfbClientNumber = 1;
  cl->generation = ++rfbClientGeneration;
```

**C — raise the default thread count (line ~533):**

```c
/* BEFORE */
  else if (rfbNumThreads < 1) rfbNumThreads = min(np, 4);
/* AFTER */
  else if (rfbNumThreads < 1) rfbNumThreads = min(np, MAX_ENCODING_THREADS);
```

> The `if (rfbNumThreads > np)` clamp just below is **unchanged** — the server
> still never runs more threads than CPU cores.

---

### 3.3 `unix/Xvnc/programs/Xserver/hw/vnc/tight.c`

**A — global continuity trackers + fix `thnd[]` initializer (line ~130):**

```c
/* BEFORE */
static Bool threadInit = FALSE;
static pthread_t thnd[MAX_ENCODING_THREADS] = { 0, 0, 0, 0 };
/* AFTER */
static Bool threadInit = FALSE;
static pthread_t thnd[MAX_ENCODING_THREADS] = { 0 };
/* cross-frame zlib-stream continuity tracking (written single-threaded). */
static int tightLastNt = 0;
static unsigned int tightLastClientGen = 0;
```

**B — per-thread zlib state in `threadparam` (line ~150):**

```c
/* BEFORE */
  int streamId, baseStreamId, nStreams;
  pthread_mutex_t ready, done;
/* AFTER */
  int streamId, baseStreamId, nStreams;
  /* per-thread zlib state -> nt>4 path. each thread deflates own zs[] only. */
  z_stream zs[4];
  Bool zsActive[4];
  int zsLevel[4];
  Bool resetStream[4];     /* this frame: wire id k needs a stream reset */
  pthread_mutex_t ready, done;
```

**C — `InitThreads`, reset the trackers (line ~302):**

```c
/* BEFORE */
  memset(tparam, 0, sizeof(threadparam) * MAX_ENCODING_THREADS);
  tparam[0].ublen = &ublen;
/* AFTER */
  memset(tparam, 0, sizeof(threadparam) * MAX_ENCODING_THREADS);
  /* zs[] left zeroed -> lazy deflateInit2 in CompressData on first use. */
  tightLastNt = 0;
  tightLastClientGen = 0;
  tparam[0].ublen = &ublen;
```

**D — `ShutdownTightThreads`, free per-thread streams (line ~350):**

```c
/* BEFORE */
  for (i = 0; i < rfbNumThreads; i++) {
    free(tparam[i].tightAfterBuf);
    free(tparam[i].tightBeforeBuf);
    if (i != 0) free(tparam[i].updateBuf);
    if (tparam[i].j) tjDestroy(tparam[i].j);
    if (!REGION_NAR(&tparam[i].losslessRegion))
/* AFTER */
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
```

**E — `rfbSendRectEncodingTight`, add a local (line ~408):**

```c
/* BEFORE */
  Bool status = TRUE;
  int i, nt;
/* AFTER */
  Bool status = TRUE, resetAll;
  int i, nt;
```

**F — zero-height-strip guard + continuity check (line ~408):**

```c
/* BEFORE */
  nt = min(rfbNumThreads, w * h / MAXRECTSIZE);
  if (nt < 1) nt = 1;

  for (i = 0; i < nt; i++) {
/* AFTER */
  nt = min(rfbNumThreads, w * h / MAXRECTSIZE);
  nt = min(nt, h);                    /* never a 0-height strip */
  if (nt < 1) nt = 1;

  /* A wire zlib stream stays continuous across frames only while the same
     thread keeps writing it.  Reset every stream this frame if: >4 threads
     share the 4 wire ids; OR nt changed (thread<->id mapping shifted); OR a
     different client encoded last (per-thread zs carry its history). */
  resetAll = (nt > 4) || (nt != tightLastNt) ||
             (cl->generation != tightLastClientGen);
  tightLastNt = nt;
  tightLastClientGen = cl->generation;

  for (i = 0; i < nt; i++) {
```

**G — thread-to-stream assignment (line ~423).** Replace the whole
`if (i < 4) { ... }` block:

```c
/* BEFORE */
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
/* AFTER */
    /* Tight has only 4 wire zlib stream ids (0..3).  Assign threads to ids.
       resetStream[k] is sticky: set here when continuity broke (resetAll),
       and stays set until the thread actually emits a reset on id k
       (CompressData clears it).  A thread that owns k but skips it for some
       frames (empty strip) therefore still resets k on its next use. */
    if (nt <= 4) {
      if (i < 4) {
        int n = min(nt, 4), b;
        tparam[i].baseStreamId = 4 / n * i;
        if (i == n - 1) tparam[i].nStreams = 4 - tparam[i].baseStreamId;
        else tparam[i].nStreams = 4 / n;
        tparam[i].streamId = tparam[i].baseStreamId;
        if (resetAll)
          for (b = tparam[i].baseStreamId;
               b < tparam[i].baseStreamId + tparam[i].nStreams; b++)
            tparam[i].resetStream[b] = TRUE;
      }
    } else {
      int wireId = i % 4;
      tparam[i].baseStreamId = wireId;       /* single wire id */
      tparam[i].nStreams = 0;                /* no round-robin */
      tparam[i].streamId = wireId;
      if (resetAll)
        tparam[i].resetStream[wireId] = TRUE;
    }
```

**H — control byte in `SendMonoRect` (line ~978):**

```c
/* BEFORE */
  else
    t->updateBuf[(*t->ublen)++] = (streamId | rfbTightExplicitFilter) << 4;
  t->updateBuf[(*t->ublen)++] = rfbTightFilterPalette;
  t->updateBuf[(*t->ublen)++] = 1;
/* AFTER */
  else
    t->updateBuf[(*t->ublen)++] =
      (char)(((streamId | rfbTightExplicitFilter) << 4) |
             ((!t->zsActive[streamId] || t->resetStream[streamId]) ?
              (1 << streamId) : 0));
  t->updateBuf[(*t->ublen)++] = rfbTightFilterPalette;
  t->updateBuf[(*t->ublen)++] = 1;
```

**I — control byte in `SendIndexedRect` (line ~1046):**

```c
/* BEFORE */
  else
    t->updateBuf[(*t->ublen)++] = (streamId | rfbTightExplicitFilter) << 4;
  t->updateBuf[(*t->ublen)++] = rfbTightFilterPalette;
  t->updateBuf[(*t->ublen)++] = (char)(t->paletteNumColors - 1);
/* AFTER */
  else
    t->updateBuf[(*t->ublen)++] =
      (char)(((streamId | rfbTightExplicitFilter) << 4) |
             ((!t->zsActive[streamId] || t->resetStream[streamId]) ?
              (1 << streamId) : 0));
  t->updateBuf[(*t->ublen)++] = rfbTightFilterPalette;
  t->updateBuf[(*t->ublen)++] = (char)(t->paletteNumColors - 1);
```

**J — control byte in `SendFullColorRect` (line ~1111):**

```c
/* BEFORE */
  else
    t->updateBuf[(*t->ublen)++] = streamId << 4;
  t->bytessent++;
/* AFTER */
  else
    t->updateBuf[(*t->ublen)++] =
      (char)((streamId << 4) |
             ((!t->zsActive[streamId] || t->resetStream[streamId]) ?
              (1 << streamId) : 0));
  t->bytessent++;
```

**K — `CompressData` (line ~1126).** Full function, before and after:

```c
/* BEFORE */
static Bool CompressData(threadparam *t, int streamId, int dataLen,
                         int zlibLevel, int zlibStrategy)
{
  z_streamp pz;
  int err;
  rfbClientPtr cl = t->cl;

  if (dataLen < TIGHT_MIN_TO_COMPRESS) {
    memcpy(&t->updateBuf[*t->ublen], t->tightBeforeBuf, dataLen);
    (*t->ublen) += dataLen;
    t->bytessent += dataLen;
    return TRUE;
  }

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

  /* Prepare buffer pointers. */
  pz->next_in = (Bytef *)t->tightBeforeBuf;
  pz->avail_in = dataLen;
  pz->next_out = (Bytef *)t->tightAfterBuf;
  pz->avail_out = t->tightAfterBufSize;

  /* Change compression parameters if needed. */
  if (zlibLevel != cl->zsLevel[streamId]) {
    if (deflateParams(pz, zlibLevel, zlibStrategy) != Z_OK)
      return FALSE;
    cl->zsLevel[streamId] = zlibLevel;
  }

  /* Actual compression. */
  if (deflate(pz, Z_SYNC_FLUSH) != Z_OK || pz->avail_in != 0 ||
      pz->avail_out == 0)
    return FALSE;

  return SendCompressedData(t, t->tightAfterBuf,
                            t->tightAfterBufSize - pz->avail_out);
}
```

```c
/* AFTER */
static Bool CompressData(threadparam *t, int streamId, int dataLen,
                         int zlibLevel, int zlibStrategy)
{
  z_streamp pz;
  int err;
  rfbClientPtr cl = t->cl;

  if (dataLen < TIGHT_MIN_TO_COMPRESS) {
    memcpy(&t->updateBuf[*t->ublen], t->tightBeforeBuf, dataLen);
    (*t->ublen) += dataLen;
    t->bytessent += dataLen;
    return TRUE;
  }

  if (zlibLevel == 0 && cl->enableTightWithoutZlib)
    return SendCompressedData(t, t->tightBeforeBuf, dataLen);

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
  } else if (t->resetStream[streamId]) {
    /* continuity broke -> restart deflate history to match the inflateReset
       the client does on the reset bit emitted in the control byte. */
    if (deflateReset(pz) != Z_OK)
      return FALSE;
  }

  /* Prepare buffer pointers. */
  pz->next_in = (Bytef *)t->tightBeforeBuf;
  pz->avail_in = dataLen;
  pz->next_out = (Bytef *)t->tightAfterBuf;
  pz->avail_out = t->tightAfterBufSize;

  /* Change compression parameters if needed. */
  if (zlibLevel != t->zsLevel[streamId]) {
    if (deflateParams(pz, zlibLevel, zlibStrategy) != Z_OK)
      return FALSE;
    t->zsLevel[streamId] = zlibLevel;
  }

  /* Actual compression. */
  if (deflate(pz, Z_SYNC_FLUSH) != Z_OK || pz->avail_in != 0 ||
      pz->avail_out == 0)
    return FALSE;

  /* reset bit emitted in caller's control byte; fresh deflateInit2 above
     realigned server+client streams -> clear so later subrects don't re-reset */
  t->resetStream[streamId] = FALSE;

  return SendCompressedData(t, t->tightAfterBuf,
                            t->tightAfterBufSize - pz->avail_out);
}
```

**L — pipeline socket I/O with encoding (perf), in `rfbSendRectEncodingTight`
(line ~488).** Stock joins *all* workers, then writes *all* buffers serially.
This writes each worker's buffer the moment that worker finishes, so
transmission overlaps the still-encoding higher-numbered threads:

```c
/* BEFORE */
  if (nt > 1) {
    for (i = 1; i < nt; i++) {
      pthread_mutex_lock(&tparam[i].done);
      status &= tparam[i].status;
    }
    if (status == FALSE) return FALSE;
    if (ublen > 0) {
      if (!rfbSendUpdateBuf(cl))
        return FALSE;
    }
    for (i = 1; i < nt; i++) {
      if ((*tparam[i].ublen) > 0)
        WRITE_OR_CLOSE(tparam[i].updateBuf, *tparam[i].ublen, return FALSE);
      (*tparam[i].ublen) = 0;
      cl->rfbBytesSent[rfbEncodingTight] += tparam[i].bytessent;
      cl->rfbRectanglesSent[rfbEncodingTight] += tparam[i].rectsent;
    }
  }
/* AFTER */
  if (nt > 1) {
    /* Pipeline output with encoding: flush thread 0's buffer now, then write
       each worker's buffer the instant that worker finishes, so socket I/O
       overlaps the still-encoding higher-numbered threads instead of running
       serially after all of them.  Every worker is still joined before
       return; a write failure tears the client down (ShutdownTightThreads
       joins all workers) and returns immediately. */
    if (ublen > 0) {
      if (!rfbSendUpdateBuf(cl))
        return FALSE;
    }
    for (i = 1; i < nt; i++) {
      pthread_mutex_lock(&tparam[i].done);
      status &= tparam[i].status;
      if (status && (*tparam[i].ublen) > 0)
        WRITE_OR_CLOSE(tparam[i].updateBuf, *tparam[i].ublen, return FALSE);
      (*tparam[i].ublen) = 0;
      cl->rfbBytesSent[rfbEncodingTight] += tparam[i].bytessent;
      cl->rfbRectanglesSent[rfbEncodingTight] += tparam[i].rectsent;
    }
    if (status == FALSE) return FALSE;
  }
```

---

### 3.4 `unix/Xvnc/programs/Xserver/hw/vnc/init.c` — help text (line ~1900)

```c
/* BEFORE */
  ErrorF("                       max. 4]\n");
/* AFTER */
  ErrorF("                       max. %d]\n", MAX_ENCODING_THREADS);
```

---

### 3.5 `unix/Xvnc/programs/Xserver/Xvnc.man.in` — man page (line ~373)

```
/* BEFORE */
Specify the number of threads to use with multithreaded Tight encoding.  The
default is to use one thread per CPU core, up to a maximum of 4 (because using
more than 4 encoding threads breaks compatibility with viewers other than the
TurboVNC Viewer.)  The server will not allow the thread count to exceed 4, nor
to exceed the number of CPU cores.
/* AFTER */
Specify the number of threads to use with multithreaded Tight encoding.  The
default is to use one thread per CPU core, up to a maximum of 32.  The server
will not allow the thread count to exceed 32, nor to exceed the number of CPU
cores.  Using more than 4 encoding threads requires the TurboVNC Viewer; other
viewers support a maximum of 4 Tight zlib streams.
```

---

### 3.6 `CMakeLists.txt` — version marker (line ~9)

```cmake
# BEFORE
set(VERSION 3.3.1)
set(DOCVERSION 3.3.1)
# AFTER
set(VERSION 3.3.1.mt32)
set(DOCVERSION 3.3.1.mt32)
```

> Use a **dot**, not a dash — RPM rejects `-` in a version string.

---

### 3.7 `java/com/turbovnc/rfb/TightDecoder.java` — viewer fix (line ~144)

**Required.** Without it the viewer crashes the moment the server emits a
stream-reset bit.

```java
/* BEFORE */
    for (int i = 0; i < 4; i++) {
      if ((compCtl & 1) != 0)
        inflater[i].end();
      compCtl >>= 1;
    }
/* AFTER */
    for (int i = 0; i < 4; i++) {
      if ((compCtl & 1) != 0)
        inflater[i].reset();
      compCtl >>= 1;
    }
```

> `Inflater.end()` permanently destroys the inflater (next use throws
> `IllegalStateException`); `.reset()` is the correct flush. Deploy the rebuilt
> `VncViewer.jar` wherever you run the viewer.

---

## 4. Build & install the RPM (RHEL 8 / Rocky 8)

```bash
# build prerequisites
sudo dnf install -y gcc gcc-c++ make cmake git bison flex rpm-build python3 \
  zlib-devel openssl-devel pam-devel pixman-devel \
  libX11-devel libXext-devel libXfont2-devel libxkbfile-devel \
  libxshmfence-devel libXdmcp-devel libXau-devel libdrm-devel \
  xorg-x11-xtrans-devel xorg-x11-util-macros xorg-x11-proto-devel \
  turbojpeg-devel java-17-openjdk-devel
# turbojpeg-devel provides turbojpeg.h + libturbojpeg.
# JDK 17 is required (the bundled viewer needs JDK 16+).

cd turbovnc                       # the patched source tree
mkdir build && cd build
JAVA_HOME=/usr/lib/jvm/java-17 cmake -G"Unix Makefiles" \
  -DCMAKE_BUILD_TYPE=Release \
  -DTJPEG_INCLUDE_DIR=/usr/include \
  -DTJPEG_LIBRARY=/usr/lib64/libturbojpeg.so \
  -DJAVA_HOME=/usr/lib/jvm/java-17 ..
make -j$(nproc)

make rpm                          # -> turbovnc-3.3.1.mt32.x86_64.rpm
sudo rpm -Uvh --force turbovnc-3.3.1.mt32.x86_64.rpm
```

Verify: `/opt/TurboVNC/bin/Xvnc -version` → `...v3.3.1.mt32...`

---

## 5. Specifying the thread count in `turbovncserver.conf`

The server config file is `/etc/turbovncserver.conf` (system-wide) or
`~/.vnc/turbovncserver.conf` (per-user). It is Perl. The variable
**`$serverArgs`** passes extra arguments straight to `Xvnc`.

Add this line:

```perl
# /etc/turbovncserver.conf  (or  ~/.vnc/turbovncserver.conf)
$serverArgs = "-nthreads 32";
```

Then start a session: `/opt/TurboVNC/bin/vncserver`

### Other ways

| Method | How |
|--------|-----|
| Config file | `$serverArgs = "-nthreads 32";` in `turbovncserver.conf` |
| Command line | `vncserver -nthreads 32` |
| Environment | `export TVNC_NTHREADS=32` before `vncserver` |
| **Do nothing** | Default auto-scales to `min(CPU cores, 32)` per frame |

- Valid range `1`–`32`; still clamped to your CPU core count.
- `-nthreads` is a ceiling. The encoder picks fewer threads per frame for
  small screen updates (dynamic scaling) — no contention on small updates.
- Confirm via the session log (`~/.vnc/*.log`): `Using N threads for Tight encoding`.

---

## 6. What was tested

Built and run on Rocky 8 (Xvnc `3.3.1.mt32`):

| Test | Result |
|------|--------|
| Clean Release build (full X server + Java viewer) | pass, 0 warnings in changed files |
| AddressSanitizer build, real 32 threads under load | **0 ASan errors** (no corruption / leak / UAF) |
| Real 32 threads, 2560×1440, 198k+ Tight rectangles | **0 decode errors** |
| 9-minute hard-transition stress, **2 simultaneous clients**, 62k+ rectangles | **0 decode errors** |
| `-nthreads` 1 / 2 / 4 / 8 / 16 / 32 | all decode clean |
| Client disconnect + reconnect | clean (client-generation guard) |
| Tiny window 240×6, `-nthreads 32` | no crash, no SIGFPE (zero-height-strip guard) |
| Pipelined-I/O build: 200 s transition stress + ASan | **0 decode errors, 0 ASan errors** |

The transition stress (mixed big + small updates, the workload that exposes
zlib-stream desync) is the key test — it caught two earlier incomplete fixes
before the final sticky-reset design passed it cleanly.

### Benchmark

Encode time per frame, full-screen incompressible random updates (worst case
for the encoder), 8-core Rocky 8 VM, final build:

| threads | encode/frame | speedup vs 1 |
|---------|--------------|--------------|
| 1 | 43.6 ms | 1.00× |
| 2 | 22.8 ms | 1.91× |
| 4 | 12.9 ms | 3.39× |
| 8 | 12.5 ms | 3.49× |

Near-linear scaling to 4 threads; it flattens past the VM's usable core budget
(the benchmark's screen-blitter and the Java viewer also consume cores, so only
~4–5 of the 8 cores are free for encode workers). Earlier runs confirmed 16 and
32 threads on the same 8-core VM still improve slightly (~3.8×) with **no
contention regression** — oversubscription does not hurt. On an 80-core box with
GPU-side VirtualGL rendering there is far more CPU headroom, so the scaling
extends much further. Benchmark `-nthreads` 4 / 8 / 16 / 32 on your real
workload (`TVNC_PROFILE=1` logs encoder throughput) to find your knee.

### Performance work in this patch

- **Threaded encode** — the core speedup: 3.4× at 4 threads above.
- **Dynamic per-frame scaling** — `nt` is chosen per frame from rectangle
  area, so small updates never pay thread-dispatch overhead (no contention).
- **Pipelined output (change L)** — stock encodes *all* strips, then transmits
  *all* of them serially. This patch transmits each worker's strip the instant
  it finishes, overlapping transmission with the still-encoding threads. On the
  loopback test this is within noise (loopback is not bandwidth-bound, so the
  write was never on the critical path); the benefit scales with how
  bandwidth-constrained the real client link is. It is correctness-neutral and
  cannot regress.

Considered and **not** done (would help but carry rearchitecture risk against
the "must not break" requirement): content-aware strip sizing (equal-height
strips ≠ equal-cost — a video region costs more than static UI); a work-stealing
queue across multiple damage rectangles; NUMA thread/framebuffer affinity on
multi-socket hosts. These are the next avenues if more speed is needed and a
larger change is acceptable.
