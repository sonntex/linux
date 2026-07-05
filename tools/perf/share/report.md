# CoreSight (cs_etm) `perf script --itrace` Segfault — Investigation Report

**Reporter:** Nikolay Beloborodov
**Date:** 2026-07-04
**Host:** aarch64 devserver (devbig072) — reproduces off-target from a copied `perf.data`
**Trace source:** production aarch64 box, `perf record -e cs_etm/autofdo/u -c 1000000 -a` (per-CPU AutoFDO CoreSight capture, ~169 MB)
**Repo:** `Leo-Yan/linux` @ `v6.19-rc6-dbg` (Linux 6.19-rc6 + 3 decoder-debug patches)

---

## TL;DR

- The **unpatched** perf (`/usr/bin/perf`, v6.19.0-rc6) **segfaults** (`SIGSEGV`) decoding the trace — reproduces the original report deterministically.
- The **debug-patched** perf (the 3 patches on top of 6.19-rc6) does **NOT** crash. It aborts early with `EINVAL` at `PERF_RECORD_AUXTRACE_INFO` processing and never reaches the decode path where the crash occurs.
- Root-cause signal from the debug logs: **queue 31 is non-empty (`empty=0`) but has zero trace IDs (`nr=0`), an invalid sink (`sink=0xffffffff`), and no decoder is created (`decoder=(nil)`).**
- **Caveat:** the debug patch's new `"no Trace ID mapping for non-empty queue"` guard returns `EINVAL` and short-circuits *before* the real crash site. So it masks the segfault rather than reaching/logging it. The actual nil-deref happens later, during decode, which only the unpatched binary reaches.

---

## Data provenance — how `perf.data` was gathered & transferred

**1. Recorded on the production target** (aarch64), host `twshared42125.03.cco5`, as root:
```
[root@twshared42125.03.cco5 ~]# perf record -e cs_etm/autofdo/u -c 1000000 -a -o /tmp/perf.data -- sleep 10
  [ perf record: Woken up 394 times to write data ]
  Warning:
  Processed 2649029 events and lost 2 chunks!    Check IO/CPU overload!
  Warning:
  Processed 5604 samples and lost 100.00%!
  [ perf record: Captured and wrote 168.713 MB /tmp/perf.data ]
```
- System-wide (`-a`) per-CPU AutoFDO CoreSight capture: `-e cs_etm/autofdo/u`, period `-c 1000000`, over a `sleep 10` window.
- Output `/tmp/perf.data`, 168.713 MB. Note the recorder warnings: 2 lost chunks and "5604 samples and lost 100.00%".

**2. Segfault first observed on the same target box** (this is the original report):
```
[root@twshared42125.03.cco5 ~]# perf script --itrace=il64 --fields=comm,pid,tid,time,ip,brstack --ns -i /tmp/perf.data
  Segmentation fault (core dumped)
```
(The reproduction in this report drops the `time` field but is otherwise the same `--itrace=il64` invocation.)

**3. Copied to the devserver via scp** — pulled from the target to `devbig072` (aarch64 devserver where the investigation was done):
```
[nbeloborodov@devbig072]~% scp root@twshared42125.03.cco5:/tmp/perf.data ./
  perf.data   100%  169MB   1.4GB/s   00:00
```
File landed at `~/perf.data` on `devbig072` (~169 MB) and is the exact file used for every run in this report.

---

## Reproduction matrix

| Binary | Patches | Reaches decode? | Result |
|--------|---------|-----------------|--------|
| `/usr/bin/perf` (6.19.0-rc6, stripped) | none | **yes** | `exit 139` — **SIGSEGV (segfault)** |
| `~/linux-leo-yan/tools/perf/perf` (6.19.rc6.g759eafa92430) | 3 debug patches | no | `exit 234` — clean `EINVAL`, no core dump |

Command used (both binaries, against the existing file):
```
perf script --itrace=il64 --fields=comm,pid,tid,ip,brstack --ns -i ~/perf.data
```

### Unpatched (system) perf — crashes
```
raw exit code = 139  -> SIGSEGV
stderr tail:
  Warning:
  CS ETM Trace: Missing DSO. Use 'perf archive' or debuginfod to export data from the traced system.
                Enable CONFIG_PROC_KCORE or use option '-k /path/to/vmlinux' for kernel symbols.
  CS ETM Trace: Debug data not found for address 0x313fe784 in /packages/multifeed.raas/raas_server
  CS ETM Trace: Debug data not found for address 0xfffc0c13a458 in .../libevict-fbcode.so
```
Note: it emits real *decode-time* messages (per-address DSO lookups) → it segfaults **inside the decode path**, well past AUXTRACE_INFO.

### Debug-patched perf — does NOT crash
```
raw exit code = 234  (genuine exit(), not 128+signal; no signal 106 exists)
core files produced: 0   (even with `ulimit -c unlimited`)
stderr tail:
  CS ETM: no Trace ID mapping for non-empty queue=31 etmq=0x84a9d0 format=2 sink=0xffffffff traceid_list=0x84aa90 own=0x84aa90 decoder=(nil)
  0x1870 [0x1b28]: failed to process type: 70 [Invalid argument]
```
`type: 70` = `PERF_RECORD_AUXTRACE_INFO`; offset `0x1870 [0x1b28]` matches the single AUXTRACE_INFO record. `cs_etm__process_auxtrace_info` returns `-EINVAL`.

---

## Key observations from the debug logs

**1. Last decoder activity before abort — queue 31:**
```
CS ETM: iterating auxtrace queue index=31 etmq=0x84a9d0 empty=0 format=2 sink=0xffffffff traceid_list=0x84aa90 nr=0 decoder=(nil)
CS ETM: creating queue decoder queue=31 etmq=0x84a9d0 decoders=0 format=2 sink=0xffffffff traceid_list=0x84aa90 own=0x84aa90
CS ETM: no Trace ID mapping for non-empty queue=31 etmq=0x84a9d0 format=2 sink=0xffffffff traceid_list=0x84aa90 own=0x84aa90 decoder=(nil)
```
- `empty=0` → the queue has trace bytes.
- `nr=0` → no trace IDs mapped for it.
- `sink=0xffffffff` → invalid/uninitialized sink.
- `decoders=0` / `decoder=(nil)` → no decoder gets built → later deref crashes on unpatched perf.

**2. Suspicious sink IDs across queues** (per-CPU, `format=2`) look uninitialized/garbage:
```
sink=0xce6849dc, 0x16c202dc, 0x5f1bbbdc, 0xa77574dc, 0xefcf2ddc, 0x32f00695, ... 0xffffffff
```

**3. Only one AUXTRACE record type present** (`perf script -D`), because `-D` also dies at the same AUXTRACE_INFO record and never reaches the `PERF_RECORD_AUXTRACE` data records:
```
0 0 0x1870 [0x1b28]: PERF_RECORD_AUXTRACE_INFO type: 3
```

---

## Captured logs (filtered to common decoder flow; no product info)

Generated with the debug-patched build against the existing `~/perf.data`:

- `perf_debug.log` (226 KB, 3647 lines) —
  `perf --debug verbose=3 script --itrace=il64 --fields=comm,pid,tid,ip,brstack --ns -i ~/perf.data 2>&1 | grep -E "CS ETM|cs_etm"`
- `perf_auxtrace_record.log` (1 line) —
  `perf script -D -i ~/perf.data 2>&1 | grep PERF_RECORD_AUXTRACE`

---

## Build details

- Working tree already at `v6.19-rc6-dbg` (3 debug commits on top of `24d479d Linux 6.19-rc6`) — no patching needed:
  - `759eafa` Add debug logs for decoder
  - `9edd073` Enhance raw Coresight trace debug display
  - `6b0b17f` Fix print issue for Coresight debug in ETE/TRBE trace
- Built with `make CORESIGHT=1` on aarch64; linked against system `libopencsd_c_api.so.1`.
- `perf version 6.19.rc6.g759eafa92430`.

---

## Suggested next steps

1. **Get the exact crash line:** build a *symboled, unpatched* perf at `24d479d` (`make CORESIGHT=1`) and run under `gdb` — the system binary is stripped, so it won't give source lines. This pins the nil-decoder deref.
2. **Re-scope the debug guard:** the `"no Trace ID mapping for non-empty queue"` check currently returns `EINVAL` from AUXTRACE_INFO processing, aborting before decode. Consider making it warn/skip the offending queue instead, so the debug build can actually reach and log the real crash path.
3. **Investigate queue 31 / trace-ID mapping:** why is a non-empty per-CPU queue left with `nr=0` trace IDs and `sink=0xffffffff`? The likely fix is to skip/guard decoder creation for queues with no trace-ID mapping instead of dereferencing a NULL decoder.
