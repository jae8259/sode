# SODE on BlueField-3: Viability & Modification Plan

Oct 12, 2026

## Goal and setup

The goal is to reproduce SODE's latency speedup over XRP on real hardware, starting with Fig. 5(c), 5(d) and 12. The device side moves from NVMeVirt on spare host cores to a BlueField-3. The BF3 presents an emulated NVMe device through the DOCA NVMe Emulation app and is backed by a remote SSD over NVMe-oF. The host keeps the SODE kernel, the applications and the selective-execution protocol.

| Role | Paper (NVMeVirt) | This port |
| --- | --- | --- |
| NVMe controller | Kernel module on host cores 12-16 | DOCA NVMe Emulation app on BF3 Arm cores plus DPA |
| On-device compute | 4 host cores fixed at 1.2 GHz | BF3 Arm Cortex-A78 cores |
| Storage | Reserved host DRAM with NVM-mode timing model | Remote SSD over NVMe-oF |
| BPF execution | Host x86 JIT image called by the emulator | uBPF on Arm; bytecode pushed by the host |
| Host-device memory | Shared kernel address space | DMA through IOVAs |

```
 Host (x86)                    BlueField-3 (Arm + DPA)                 Remote storage
+---------------------+       +-----------------------------------+   +---------------+
| Application         |       | DPA threads                       |   |               |
| BPF-KV, WiredTiger  |       | poll doorbells, raise MSI-X       |   |               |
+----------+----------+       +-----------------+-----------------+   |               |
           |                                    | doorbell            |               |
           v                                    v                     | +-----------+ |
+---------------------+  SQE  +-----------------------------------+   | | SSD       | |
| SODE kernel         | 0x82  | NVMe Emulation app (SPDK)         |   | +-----^-----+ |
| read_sode, NVMe drv +------>| fetch SQE; opcode 0x82 -> SODE    |   |       | read  |
+----------+----------+       +-----------------+-----------------+   |       v       |
           | allocates                          | SODE request        | +-----------+ |
           v                                    v                     | | NVMe-oF   | |
+---------------------+  DMA  +-----------------------------------+   | | target    | |
| Host memory         |<----->| SODE engine on Arm cores          |<----->| RDMA/TCP  | |
| descriptor, scratch,|       | read block into Arm memory,       |per| +-----------+ |
| final data page     |       | run uBPF, map next offset, repeat |hop|               |
+---------------------+       +-----------------------------------+   +---------------+
```

The host issues one SQE with opcode `0x82`. Every intermediate block then stays in BF3 memory, and only the final page, scratch and status are DMA'd back before the CQE.

## I/O path: XRP and SODE on NVMeVirt vs SODE on BF3

One lookup needs N block reads ("hops"). The three models below show who does what on each hop. The NVMeVirt models are taken from this repo's code.

### XRP on NVMeVirt (the paper's baseline)

The "device" is host cores pretending to be an SSD. All hop logic runs on the host.

```
Host CPU (app core)                          NVMeVirt (host cores 12-16, same kernel)
read_xrp() -> bio -> nvme_queue_rq
  write SQE into SQ (host RAM),
  write doorbell into fake BAR (host RAM) --> dispatcher kthread polls BAR, sees doorbell
                                              I/O worker: memcpy 4 KiB from reserved DRAM
                                                into the PRP page (same RAM, no DMA)
                                              wait until the timing model says "done"
                                              write CQE, apic->send_IPI = fake MSI-X
nvme_handle_cqe (IRQ context, pci.c)    <--+
  run BPF (x86 JIT) on the page
  ext4 extent lookup -> next LBA
  rewrite slba, nvme_submit_cmd ----------> same again ... N times
  last hop: complete the bio -> return to app
```

Per hop: modelled media latency, one interrupt, BPF on the fast host core, one resubmit.

### SODE on NVMeVirt (the paper's system)

The host submits once. The hop loop moves into the emulator.

```
Host CPU                                     NVMeVirt
read_sode() -> nvme_queue_rq
  alloc resubmit_data (extent copy, prog ptr,
  scratch ptr), opcode |= 0x80,
  rsvd2 = phys addr of resubmit_data ------> dispatcher fetches SQE
                                              worker kmap()s resubmit_data (shared RAM)
                                              +- memcpy block from DRAM + timing model
                                              |  if a wimpy core is idle:
                                              |    wimpy core (1.2 GHz) runs BPF_PROG_RUN
                                              |    on the host's data page (shared RAM)
                                              |    -> next LBA -> loop
                                              +- else: give up, status = -1
                                              CQE + IPI, once
nvme_handle_cqe                         <--+
  status = done -> check extent version, complete
  status = -1   -> continue XRP-style (model above)
```

Per hop: modelled media latency plus BPF on a wimpy core. No interrupt and no host work between hops.

### SODE on BlueField-3 (what we build)

The host side is unchanged except that the descriptor is passed by DMA address. The device side becomes real.

```
Host CPU                     PCIe        BF3 (DPA + Arm)                    NVMe-oF      Remote SSD
read_sode() -> nvme_queue_rq
  SQE 0x82, rsvd2 = descriptor IOVA
  doorbell (MMIO) -------------------> DPA sees doorbell
                                        Arm DMA-reads SQE (64 B), descriptor
                                        + extents, scratch (once)
                                        +- capsule ----------------------------> read
                                        |  4 KiB RDMA-written into Arm memory <-+
                                        |  if an Arm SODE core is idle:
                                        |    uBPF on the Arm buffer -> next LBA -> loop
                                        +- else: status = -1
                                        DMA leaf page + scratch + status (once)
                                        CQE + MSI-X (once)
nvme_handle_cqe                   <----+   (same done / hand-back logic as above)
```

Per hop: real network plus SSD latency, and uBPF on an A78 core. Nothing crosses PCIe between hops.

### Side by side

| Per hop | XRP / NVMeVirt | SODE / NVMeVirt | SODE / BF3 |
| --- | --- | --- | --- |
| Who reads the block | Emulator `memcpy` from host DRAM | Same | SPDK over NVMe-oF to the SSD |
| Where the block lands | Host page (shared RAM) | Host page (shared RAM) | BF3 Arm memory |
| Media latency | Timing model | Timing model | Real |
| Who runs the BPF | Host core, x86 JIT, IRQ context | Wimpy core, host's x86 JIT image | Arm core, uBPF |
| Interrupt per hop | Yes (fake IPI) | No | No |
| Host work per hop | IRQ + BPF + extent lookup + resubmit | None | None |
| Bytes over PCIe per hop | None (fake device) | None | None |
| Busy device | n/a | Hand back to host (XRP-style) | Same |

In the paper, XRP and SODE share the same fake data path; they differ only in who runs the between-hop work and whether an interrupt happens. On BF3 we keep that difference and make the data path and the device real.

### XRP baseline on the real machine

XRP needs nothing from BF3: it only needs an NVMe device that completes reads and raises interrupts, and all its logic is in the host's NVMe completion handler (`pci.c`). We run it against BF3 acting as a plain pass-through NVMe device (stock DOCA app, none of our code). XRP and SODE then share the same per-hop media path (PCIe, BF3, NVMe-oF, SSD), and the measured gap is exactly the per-hop round trip SODE removes.

The alternative, the host reaching the SSD through its own NVMe-oF initiator, would need XRP's hook ported into `nvme-rdma` or `nvme-tcp` and would mix BF3's pass-through cost into the comparison.

## What NVMeVirt gets for free from shared RAM

NVMeVirt runs SODE's device side inside the host kernel, so three things that are hard on a real device are plain memory accesses there. These three, not the generic NVMe plumbing, are the porting work. Ordinary SQ/CQ fetch, PRP walking and data DMA for normal reads are already handled by the DOCA NVMe Emulation app.

### 1. Descriptor: read in place with `kmap`

**Today.** The host builds `struct resubmit_data` in its own memory (`core.c:951-1090`): pointers to the data page, scratch page, subtask scratches, the `nvme_command`, the `bpf_prog`, the inode's `i_op`, the namespace's `queuedata`, plus a copy of the file's extent tree. It puts the physical address in the SQE: `cmndp->rw.rsvd2 = virt_to_phys(...)` (`core.c:1091-1095`, `pci.c:967-969`). The emulator turns that back into a pointer with `kmap_atomic_pfn(PHYS_PFN(phys))` (`io.c:406-411`) and reads and writes the struct in place for the whole request. It follows the pointers inside it as if it were the host: `on_meta->xrp_bpf_prog`, `on_meta->xrp_scratch_page`, `on_meta->extents`, `nvme_sect_to_lba(on_meta->queuedata, ...)` (`io.c:1499`). The host later reads `on_meta->status` and `slba` that the emulator wrote (`pci.c:1222-1240`).

**On BF3.** The Arm cores cannot dereference any host address. At request start the engine DMA-reads the descriptor and its extent array into Arm memory once. Every field it needs must be a value or a DMA address, not a kernel pointer (Mod 1). Results (`status`, `slba`, counters) go back by DMA before the CQE (Mod 2).

### 2. BPF program: calls the host's x86 JIT image

**Today.** The SODE program is loaded on the host by libbpf (`bpf_prog_load(..., BPF_PROG_TYPE_SODE, ...)`, `benchmark/BPF-KV/helpers.c:147`, `wiredtiger-sode/src/conn/conn_api.c:2356`), verified, and JIT-compiled to x86 by the host kernel. The emulator receives the `struct bpf_prog *` through the descriptor and calls `BPF_PROG_RUN(ebpf_prog, &ctx)` (`io.c:1465, 1536, 1547`), which jumps into x86 machine code in host kernel memory. `ctx.data` and `ctx.scratch` point at host pages.

**On BF3.** There is no host JIT image and no kernel BPF runtime in the path. The bytecode must be shipped to BF3 and executed there by a userspace BPF engine, with `ctx.data` and `ctx.scratch` pointing at Arm buffers. The details are in Mod 3.

### 3. Blocks: land in the host page via `memcpy`

**Today.** Each hop's read is `memcpy(vaddr + mem_offs, ns.mapped + offset, io_size)` from the reserved DRAM into the host's PRP page (`io.c:180-181`, opcode `nvme_cmd_read | nvme_cmd_sode`), using `w->slba` updated by the previous hop (`io.c:131-133`). The same host page is then `ctx.data` for the BPF run (`on_meta->xrp_data_page = page_address(bio_page(req->bio))`, `core.c:1035`). Every intermediate block is written into host memory, at no cost, and media latency comes from the timing model.

**On BF3.** Each hop is a real asynchronous read: `spdk_bdev_read()` on the NVMe-oF bdev into an Arm-local buffer, with a completion callback that runs the BPF and issues the next read. Intermediate blocks never go to the host. Only the last block is DMA'd to PRP1 (Mod 5).

## Viability

Viable. The host kernel and the selective-execution protocol carry over; the device side is rebuilt around the three shared-RAM dependencies above.

| What the emulator does today | Where | Why it breaks on BF3 | Fixed in |
| --- | --- | --- | --- |
| Reads the descriptor in place and follows the host kernel pointers inside it (`xrp_scratch_page`, `extents`, `subtask_scratch[]`, `xrp_command`, `queuedata`, `i_op`, `xrp_bpf_prog`) | `core.c:1091`, `io.c:406`, `io.c:1499`, `nvmev.h:163` | Arm cores cannot dereference host memory | Mod 1, Mod 2 |
| Calls `BPF_PROG_RUN` on the host's x86 JIT image | `io.c:1465, 1536, 1547` | No host JIT image or kernel BPF runtime on BF3 | Mod 3 |
| Reads each hop's block with `memcpy` from host DRAM into the host page | `io.c:180-181` | Blocks come from a real SSD over NVMe-oF, asynchronously | Mod 5, Mod 6 |
| The "device" is a virtual PCI bus created by a host kernel module | `pci.c`, `main.c` | BF3 exposes a real PCIe function | Mod 4 (the DOCA app already does this) |

What makes it workable:

- The DOCA NVMe Emulation app already provides the NVMe plumbing: SQE fetch, PRP walk, data DMA, CQE, MSI-X.
- The SODE programs only compute over `data` and `scratch`; the one helper they reference (`bpf_printk`, WiredTiger error paths only) can be stubbed. A userspace BPF engine with an arm64 JIT can run them (Mod 3).
- SODE's benefit becomes more faithful: intermediate hops land in BF3 DRAM, the BPF runs there, and only the final page is DMA'd to the host.
- Parallel mode needs 4 worker cores; BF3 has 16 Arm Cortex-A78 cores.

## DOCA NVMe device emulation: what we build on

Use the [DOCA NVMe Emulation application](https://networking-docs.nvidia.com/doca/sdk/doca+nvme+emulation+application+guide) (DOCA 3.5.0), not the SNAP-4 service. It ships as source, fetches every SQE on the Arm cores, and hands it to SPDK. That gives us the one hook SODE needs: see opcode `0x82` and `rsvd2` before SPDK does.

How it works, per the guide:

- **Frontend:** [DevEmu PCI Generic](https://docs.nvidia.com/doca/archive/3-2-0/DOCA+DevEmu+PCI+Generic) exposes an NVMe-compliant PCIe function on BlueField-3. The host uses its normal NVMe driver, which for us is the patched SODE driver.
- **Transport:** PCIe is modelled as an SPDK NVMe-oF transport (`spdk_nvmf_transport_ops`, `nvmf_doca_*`). The SPDK nvmf target runs on the Arm cores as a black box.
- **Doorbells and interrupts:** DPA threads poll doorbells, forward them to the Arm side over message queues, and raise MSI-X when asked. One DPA thread per CQ.
- **SQE fetch:** Arm DMA-reads new SQEs (`nvmf_doca_sq_update_pi()`). I/O SQEs land in `nvmf_doca_on_fetch_nvm_sqe_complete()`, are wrapped in `nvmf_doca_request`, then go to `spdk_nvmf_request_exec()`.
- **Data:** each SQ has a pool of Arm-local DMA buffers. `nvmf_doca_sq_copy_data()` moves data asynchronously; `nvme_cmd_map_prps()` walks PRP lists. Host memory is reached through a per-device mmap held by each poll group.
- **Threading:** SPDK reactors, one per core, polling; cross-thread work by `spdk_thread_send_msg`, no locks.
- **Source and build:** `/opt/mellanox/doca/applications/nvme_emulation/` (`nvme_emulation.c`, `rpc_nvmf_doca.py`, `device/`, `host/`, `meson.build`). Built with meson and ninja; a custom SPDK is set via `spdk_lib_path` and related options.

What this means for SODE:

| Need | Fit | Note |
| --- | --- | --- |
| Intercept the SODE opcode and read `rsvd2` | Good | Branch in `nvmf_doca_on_fetch_nvm_sqe_complete()` on the raw SQE before `spdk_nvmf_request_exec()`. Opcode `0x82` is in the NVMe vendor-specific I/O range and its low bits mark controller-to-host data, so the spec allows it |
| DMA descriptor, scratch, final page to/from host | Good | Reuse the per-poll-group host mmap and `nvmf_doca_sq_copy_data()` |
| Data-dependent hops stay on BF3 | Good | Read hops with `spdk_bdev_read()` into Arm-local buffers; run BPF there; nothing crosses PCIe until the last hop |
| NVMe-oF SSD backend | Good | Attach the remote SSD as an SPDK `bdev_nvme` bdev (RDMA or TCP) and add it as the subsystem namespace |
| Only Read, Write, Flush supported | Must extend | `0x82` and a vendor admin command for BPF upload are new code paths we own |
| SPDK version | Constraint | Guide states SPDK 23.01; our engine and uBPF integrate against that |

The SNAP-4 service is the wrong base here: the container has no source, and the [SNAP-4 guide](https://networking-docs.nvidia.com/doca/sdk/doca-snap-4-service-guide) documents no custom-opcode hook. Its RDMA zero-copy path also moves data host-to-remote without staging on Arm, which is the opposite of what SODE needs.

## Risks and open questions

Real NVMe-oF media latency per hop is expected, not a risk: SODE's saving per hop is the same, it is just a smaller fraction of a slower hop.

| Risk | Effect | Mitigation |
| --- | --- | --- |
| Frontend overhead not in NVMeVirt | Every I/O pays DPA doorbell forwarding and Arm SQE DMA | Run the XRP baseline on the same emulated device; it is also the realistic cost |
| A78 cores are not "wimpy" | On-device BPF runs faster than the paper's 1.2 GHz Xeon emulation | Report as-is; optionally pin to fewer cores. Changes absolute numbers, not the trend |
| SPDK 23.01 pin in the NVMe Emulation app | uBPF and our code must build against it | Vendor uBPF as a static library inside the app build |
| Vendor opcode `0x82` passes the DPA and transport untouched | If the DPA or nvmf layer filters opcodes, the hook fails | Checked in the installed DOCA 3.2.1 source: `nvmf_doca_on_fetch_nvm_sqe_complete()` (`/opt/mellanox/doca/applications/nvme_emulation/host/doca_transport.c:3310`) switches on the opcode and answers anything but Flush, Write and Read with an error CQE. We add a `0x82` case there. M1 confirms the DPA side passes it through |
| Host kernel 5.12 on current server hardware | May not boot or lack drivers | Test boot before any other work |
| Extent table size per request | Descriptor DMA grows with file fragmentation | Cache the extent table per inode on BF3, keyed by `xrp_extent_version` |

Open questions:

- What NVMe-oF transport and media sit behind BF3: RDMA or TCP, and which SSD?
- Which DOCA and BF3 firmware versions are installed? DevEmu PCI Generic needs DOCA 2.7.0+ and firmware 32.41.1000+.
- Is the host IOMMU on? The plan assumes it is and uses proper DMA mapping.

# Modification plan

## Mod 1: Pointer-free, DMA-addressable SODE descriptor

**Purpose.** Make `struct resubmit_data` usable by a device that can only DMA it. Keep every existing field so the host code and the NVMeVirt build are unchanged; mark the host pointers as host-only with a macro; add the few values the BF3 device needs under `#ifdef SODE_BF3`. The SQE's `rsvd2` carries the descriptor's IOVA from `dma_map_single`, not `virt_to_phys`, which also removes the `intel_iommu=off` requirement.

**Why nothing has to move.** x86-64 and arm64 are both LP64 little-endian with the same alignment rules: `int` is 4 bytes, `u64` and pointers are 8 bytes and 8-aligned. The struct as written today has the same byte layout on both sides. A pointer field is just 8 opaque bytes to the device. The same holds for `struct xrp_extent` (`include/linux/fs.h:742`): its `rb_node` is 24 bytes of host pointers the device skips, followed by `lblk`, `len`, `pblk`, `version`, 48 bytes in total.

**Shared header sketch** (one file included by the host kernel, NVMeVirt and the BF3 app):

```c
#ifdef SODE_DEVICE                 /* BF3 build: host pointers are opaque 8 bytes */
#define SODE_HOST_PTR(type, name)  __u64 name
#else                              /* host kernel and NVMeVirt builds */
#define SODE_HOST_PTR(type, name)  type name
#endif

struct resubmit_data {
	SODE_HOST_PTR(char *, xrp_data_page);              /* host only */
	SODE_HOST_PTR(char *, xrp_scratch_page);           /* host only */
	u64 slba;                                          /* device writes */
	int status;                                        /* device writes */
	SODE_HOST_PTR(struct nvme_command *, xrp_command); /* host only */
	SODE_HOST_PTR(struct bpf_prog *, xrp_bpf_prog);    /* host only (NVMeVirt calls it) */
	u64 xrp_partition_start_sector;                    /* device reads */
	SODE_HOST_PTR(struct inode_operations *, i_op);    /* host only */
	SODE_HOST_PTR(void *, queuedata);                  /* host only (NVMeVirt derefs it) */
	SODE_HOST_PTR(struct xrp_extent *, extents);       /* host only; array follows the header */
	int num_extents;                                   /* device reads */
	u64 first_slba;
	int size;                                          /* device reads: bytes to DMA */
	u64 cpu_arival;
	u64 prev_cpu_arival;
	int previous_complete_counter;
	u64 xrp_ebpf_time;                                 /* device writes */
	SODE_HOST_PTR(u64 *, subtask_scratch[4]);          /* host only */
	u64 is_parallel;                                   /* device reads */
	u64 xrp_extent_version;                            /* device reads */
#ifdef SODE_BF3                    /* new: what a DMA-only device needs */
	u64 scratch_iova;                                  /* replaces xrp_scratch_page for the device */
	u64 subtask_scratch_iova[4];                       /* replaces subtask_scratch[] */
	u32 prog_id;                                       /* replaces xrp_bpf_prog (Mod 3) */
	u8  lba_shift;                                     /* replaces queuedata */
	u8  is_ext4;                                       /* replaces i_op */
#endif
};
```

`static_assert` on `sizeof` and on `offsetof` for every device-read field, in both builds, catches any layout drift.

**What each build sees.**

| Build | `SODE_BF3` | `SODE_DEVICE` | Effect |
| --- | --- | --- | --- |
| Host kernel for NVMeVirt | off | off | Exactly today's struct |
| Host kernel for BF3 | on | off | Today's struct plus the new fields at the end |
| BF3 app | on | on | Same bytes; host pointers typed as `__u64` so they cannot be dereferenced by mistake |

The host keeps using its own pointers (`xrp_scratch_page`, `subtask_scratch[]`, `extents`) in the completion path, so `pci.c` logic is unchanged apart from DMA sync (Mod 2).

**Files.**

- `src/kernel/linux/drivers/nvme/host/nvme.h:861`: struct moves to the shared header with the macro and the `SODE_BF3` block.
- `src/kernel/linux/drivers/nvme/host/core.c:951-1110`: `__alloc_resubmit_req` fills the new fields (`lba_shift` from the namespace, `is_ext4` from `i_op`, `prog_id`); `alloc_resubmit_req` returns a `dma_map_single` IOVA; the scratch and subtask scratch pages are DMA-mapped; `free_resubmit_req` unmaps. The per-CPU backup caches stay but hold mapped buffers.
- `src/emulator/nvmev.h:163`: includes the shared header instead of its own copy.

## Mod 2: Host completion path reads device-written state safely

**Purpose.** The host's SODE completion branch reads state that BF3 now writes by DMA, so it needs a defined handoff. Today the emulator writes host memory directly from the same kernel, so ordering and cache coherency come for free.

What the host branch does today (`drivers/nvme/host/pci.c:1215-1385`):

1. Reads `data->status`. A value above 0 means the device finished the traversal. The host then compares `data->xrp_extent_version` against the inode's extent tree and, if stale, restarts the whole request from `first_slba` with a fresh descriptor.
2. Otherwise the device handed the request back. The host runs the BPF program on the bio's data page and scratch, maps the next offset through the descriptor's extent copy (`driver_retrieve_mapping`), and resubmits XRP-style. In parallel mode it also copies `level` and `nr_page` into each subtask scratch.

**Required changes.**

- **Device-side ordering contract:** before posting the CQE, BF3 must have DMA-written the last hop's block to PRP1, the scratch page, the subtask scratches (parallel mode), and the descriptor's `status`, `slba` and counters. The CQE is written last.
- **Host-side sync:** `dma_sync_single_for_cpu` on the descriptor and scratch mappings before reading them; `dma_sync_single_for_device` before resubmitting. On x86 with coherent DMA these are cheap, but they keep the code correct.
- **Resubmit path:** where the host rewrites `rsvd2` with a fresh descriptor (the stale-extent restart and the fallback resubmit), use the IOVA from Mod 1, not a physical address.
- **Extent lookup:** unchanged. `driver_retrieve_mapping` keeps reading the host's own `extents` copy.

**Files.**

- `src/kernel/linux/drivers/nvme/host/pci.c`: `nvme_queue_rq` (descriptor attach, around line 967), SODE completion branch (1215-1385), `driver_retrieve_mapping` (1036).
- `src/kernel/linux/drivers/nvme/host/core.c`: `free_resubmit_req` unmaps.

The timeout override in `nvme_timeout` (`pci.c:1622`, returns `BLK_EH_RESET_TIMER` for SODE requests) stays. Long on-device traversals over NVMe-oF make it more necessary, not less.

## Mod 3: Ship BPF programs to the device

**Purpose.** BF3 cannot use the host's `struct bpf_prog` or its x86 JIT image. The device needs the bytecode, an engine to run it on Arm, the same calling convention NVMeVirt used, and a key in the descriptor to find the right program. Compiling the C to BPF is unchanged; everything after that is new.

### How a SODE program runs today

1. **Build:** clang compiles the C to a BPF ELF (`benchmark/BPF-KV/sode-bpf/Makefile`, `benchmark/wiredtiger-sode/bpf_prog/Makefile`). Programs: `get.c` (section `oliver_agg`), `range.c` (`oliver_range`), WiredTiger `wt_bpf.c` (`prog`; the parallel and no-parallel variants are different files).
2. **Load:** the application calls libbpf `bpf_prog_load(path, BPF_PROG_TYPE_SODE, ...)` (`benchmark/BPF-KV/helpers.c:147`, `wiredtiger-sode/src/conn/conn_api.c:2356`). libbpf links the `__noinline` subprograms into one flat instruction array and calls `bpf()`.
3. **Verify:** the host kernel checks the program with the XRP/SODE rules in `fs/ioctl.c:911-961`. The context is `struct bpf_xrp` (`include/uapi/linux/bpf.h:5262`): `done`, `next_addr[16]`, `size[16]` are read-write; `data` and `scratch` are read-only pointers, each verified as 4 KiB of memory (`PTR_TO_MEM`, `mem_size = PAGE_SIZE`). Helpers allowed: `bpf_base_func_proto`. There is no `convert_ctx_access`, so the program reads the context struct exactly as laid out.
4. **JIT:** the host kernel compiles it to x86.
5. **Run:** NVMeVirt calls `BPF_PROG_RUN(prog, &ctx)` on that x86 image, per hop. On hand-back, the host's completion handler runs the same image on the next block (`pci.c`).

### What the programs need from an engine

| Program | Subprogram calls | Helpers | Context use |
| --- | --- | --- | --- |
| BPF-KV `get.c` | none (`__inline` only) | none | reads `data`, `scratch`; writes `done`, `next_addr[0]`, `size[0]` |
| BPF-KV `range.c` | none (`__inline` only) | none | same |
| WiredTiger `wt_bpf.c` (parallel) | yes, `__noinline` chain about 5 deep (`ebpf_search_int_page` → `ebpf_parse_cell_addr` → `ebpf_addr_to_offset` → `ebpf_vunpack_uint` → `ebpf_unpack_posint`) | `bpf_printk` = `bpf_trace_printk`, on error paths only | also reads `done` as input: the subtask number |

`bpf_printk` in the repo's `bpf_helpers.h` builds its format string on the stack, so there is no `.rodata` section to relocate. Check this again if the programs are rebuilt against a newer libbpf, where `bpf_printk` uses `.rodata`.

### Calling convention the device must reproduce

Taken from `src/emulator/io.c:1443-1580`:

- **Single mode:** zero the context; `data` = this hop's block, `scratch` = the request's scratch; run. Return value non-zero means error. Then `done` set means finished; otherwise `next_addr[0]` is the next **file offset** (not an LBA) and `size[0]` its length. The engine maps the offset through the extent copy to an LBA.
- **Parallel mode (WiredTiger):** each of the 4 subtasks runs with `done = subtask + 1` as input and its own subtask scratch over the same block. Output is packed into `done`: `done & 0x7` is the subtask's status and `done >> 4` is the finished flag (`io.c:1544-1566`; `wt_bpf.c:558-594`). The owner merges the 4 results (`io.c:1780-2040`).

### Engine choice on BF3

The engine runs inside the NVMe Emulation app, a userspace SPDK process on the Arm cores.

| Option | How | For | Against |
| --- | --- | --- | --- |
| uBPF, arm64 JIT (recommended) | Load the instruction array into a uBPF VM, JIT once per program, call the JIT'd function per hop | Same bytecode as the host; user-supplied programs, as in the paper | Must confirm uBPF's arm64 JIT supports local calls (WiredTiger) and registered helpers (`bpf_trace_printk` stub) |
| uBPF interpreter | Same VM, no JIT | Simplest; good first step and correctness reference | Slower per hop |
| BF3's own Linux kernel BPF | Load into the BF3 kernel, run via `BPF_PROG_TEST_RUN` | Mature arm64 JIT and verifier | A syscall and buffer copy per hop from the SPDK reactor; the SODE program type would have to be added to the BF3 kernel |
| Native arm64 build of the same C | Compile `get.c` / `wt_bpf.c` with gcc for arm64 behind shim macros | Fastest; `io.c:887-1440` already holds a native C copy of the WiredTiger lookup | No longer user-supplied eBPF; a different program from the one the host runs |

Use the native build only as a reference, to measure how much per-hop time the BPF engine costs.

uBPF details to handle:

- **Memory checks:** the interpreter by default only allows loads and stores inside the one buffer passed to it. The SODE context holds pointers to two other buffers (`data`, `scratch`), so that check must be turned off or extended to those two 4 KiB regions. The host verifier has already proved every access stays inside them.
- **Helpers:** register function 6 (`bpf_trace_printk`) as a stub that logs or does nothing. Any other helper ID means the program is rejected and the device hands requests back.
- **Stack:** the kernel gives 512 bytes per frame; the WiredTiger call chain needs a few frames. The engine's stack must be at least that deep.

### Two engines, one request

On hand-back the request moves between engines: the device may run k hops in uBPF, then the host runs the remaining hops in its x86 kernel JIT on the same scratch bytes (`pci.c:1270-1385`). Both must produce identical bytes for identical input. x86-64 and arm64 are both little-endian, and the BPF instruction semantics are fixed, so this should hold. It must still be tested: for each key, compare a SODE run that is forced to hand back after k hops against an XRP run, for every k.

### Delivering the program to BF3

What to ship: the flat instruction array libbpf passes to `bpf()`. Subprograms are already linked into it with call offsets fixed, so the device needs no ELF loader. The copy must be taken before `bpf_check()`, which rewrites instructions in place.

**Recommended design: kernel pushes, keyed by BPF prog ID.**

1. In `bpf_prog_load` for `BPF_PROG_TYPE_SODE` and `BPF_PROG_TYPE_SODE_PARALLEL`, keep a copy of the instructions before `bpf_check()` rewrites them in place.
2. After a successful load, call a hook the NVMe driver registers. It sends a vendor admin command (e.g. opcode `0xC1`, "SODE load program") carrying `prog->aux->id` and the original instructions to every SODE-capable controller. This runs in process context, so a synchronous admin command is fine.
3. The descriptor (Mod 1) carries the `u32` prog ID. On BF3 the app keeps an ID-to-uBPF-VM table; an unknown ID means the device hands the request back to the host.
4. On program free, send an unload admin command.

This needs no change to the benchmark loaders (`benchmark/BPF-KV/helpers.c:139`, `benchmark/wiredtiger-sode/src/conn/conn_api.c:2356`).

**Fallback for the first milestone.** Hard-code one program on BF3, loaded from its ELF at app start, and ignore the ID. BPF-KV `get.o` has no subprogram calls or helpers, so a minimal ELF load is enough. This unblocks end-to-end tests of Mods 4 and 5 before the admin path exists.

**Files.**

- `src/kernel/linux/kernel/bpf/syscall.c`: `bpf_prog_load`, instruction copy and load or unload hook for the SODE types (types already handled near line 2968 and 3032).
- `src/kernel/linux/drivers/nvme/host/core.c`: hook implementation, vendor admin command.
- BF3 app (new): admin handler for `0xC1` and its unload counterpart, uBPF VM table, helper stub, the two calling conventions above.

**To verify before M3:** uBPF arm64 JIT with local calls and a registered helper, using `wt_bpf.c`; the interpreter memory-check workaround; the handoff test between uBPF and the kernel JIT.

## Mod 4: NVMe frontend on the DOCA NVMe Emulation app

**Purpose.** Replace NVMeVirt's host-side fake PCI device with the DOCA NVMe Emulation app on BF3. The app already does what `pci.c`, `admin.c` and the queue half of `io.c` do: BAR and doorbells (DevEmu plus DPA), SQE fetch, PRP walk, CQE post, MSI-X. We keep it as is and add one branch.

**The hook.** In `nvmf_doca_on_fetch_nvm_sqe_complete()`:

- If `opcode == 0x82` (read with the SODE bit), do not call `spdk_nvmf_request_exec()`. Hand the raw SQE and its `nvmf_doca_request` to the SODE engine (Mod 5) on the same reactor.
- Everything else goes down the existing path unchanged, so the XRP baseline hits the same frontend and backend as SODE. That keeps the comparison fair.
- When the engine finishes, it completes the request through the app's normal CQE path, so the DPA raises MSI-X as for any other read.

**Admin.** Add the vendor admin opcodes from Mod 3 to the admin handler (`nvmf_doca_on_fetch_sqe_complete()` path). Identify should still report the namespace from the NVMe-oF bdev.

**Queue and core layout.** NVMeVirt used 1 dispatcher + 4 I/O workers + 4 "wimpy" cores. On BF3 the mapping becomes SPDK reactors: some own the emulated queue pairs (poll groups), others run SODE subtasks for parallel mode. Pin with the SPDK core mask; start with 4 I/O reactors and 4 SODE reactors to mirror the paper.

**Host-visible behaviour to preserve.**

- The data transfer for `0x82` goes into PRP1, exactly one 4 KiB block (the host checks 4 KiB alignment in `read_sode`).
- `cqe.status` success; the outcome travels in the descriptor's `status` (Mod 2).
- No latency injection: completion time is real.

**Files.**

- Replaced, not ported: `src/emulator/pci.c`, `pci.h`, `main.c`, `admin.c`, `dma.c`, and the SQ/CQ parts of `io.c` (`nvmev_proc_io_sq`, `__enqueue_io_req`, `__fill_cq_result*`, `__reclaim_completed_reqs`).
- New: patch to `/opt/mellanox/doca/applications/nvme_emulation/` transport source (SQE hook, admin opcodes) and `meson.build` (link uBPF and the SODE engine).
- Setup: `rpc_nvmf_doca.py` script that creates the transport, attaches the NVMe-oF bdev, creates the subsystem, and adds the listener (hotplug).

## Mod 5: Device-side SODE engine on the Arm cores

**Purpose.** Port the SODE core logic, meaning the hop loop, the selective hand-back policy and parallel subtasks, from NVMeVirt kthreads into an SPDK-native module inside the NVMe Emulation app. All intermediate state lives in Arm memory; the host sees only the final result.

**Per-request flow (single mode).**

1. **Start:** from the SQE take `rsvd2` (descriptor IOVA), `slba`, PRP1. DMA-read the descriptor header plus extent array and the scratch page into Arm buffers. Look up the uBPF VM by `prog_id`.
2. **Hop read:** `spdk_bdev_read()` one block at `slba` from the NVMe-oF bdev into an Arm-local 4 KiB buffer. Async; the reactor keeps polling.
3. **Selective check:** run on-device only if this request's SODE core is idle. That mirrors `resubmit_waiting[worker->id] == 0` in `io.c:2240`. Otherwise go to step 6 with `status = -1` (host continues XRP-style).
4. **Compute:** run the BPF program with `ctx.data` = hop buffer, `ctx.scratch` = Arm copy of scratch.
5. **Next hop:** if `ctx.done`, go to step 6 with `status = 2`. Else map `next_addr[0]` through the local extent copy (`driver_retrieve_mapping` logic), compute `slba` with `lba_shift` and partition start, and loop to step 2.
6. **Finish:** DMA-write the last hop's block to PRP1, the scratch page, then the descriptor's `status`, `slba` and counters. Post the CQE through the app's normal path.

**Parallel mode.** Same loop, but at step 4 the hop is fanned out to 4 SODE reactors with `spdk_thread_send_msg`. Each runs the program with `ctx.done = subtask + 1` on its own subtask scratch over the same hop buffer; the owner merges their results. Today's merge is in `io.c` around lines 1780-2040. The eligibility check stays: all 4 SODE cores idle, page `type != 7`, and `entries / 2 <= 16` (`io.c:2273`). The subtask scratches are DMA-read at start and written back at finish, because the host reads them on hand-back (Mod 2).

**What goes away.**

- `kmap_atomic_pfn` on host memory, `BPF_PROG_RUN`, every pointer from Mod 1.
- Spin-wait synchronization (`global_worker_lock`, `volatile` arrays): replaced by SPDK message passing between reactors.
- The work-queue sequencing in `__reclaim_completed_reqs` and friends: SPDK request objects own lifetime.
- Dead code: `wiredtiger_lookup` and the `ebpf_*` C copies in `io.c:887-1440` (their call is commented out at `io.c:1548`).

**Files.**

- Source of the logic: `src/emulator/io.c` (`resubmit_logic_single` 1443, `resubmit_logic` 1510, the resubmit worker loop through about 2330), `src/emulator/nvmev.h` (`nvmev_io_work`, `NUM_R_CPU`).
- New: `sode_engine.c` / `.h` inside the NVMe Emulation app, plus vendored uBPF.

## Mod 6: NVMe-oF storage backend, no timing model

**Purpose.** NVMeVirt kept data in reserved host DRAM (`memmap=192G$192G`) and faked SSD latency with a timing model. On BF3, data lives on a real SSD reached over NVMe-oF, and latency is whatever the media and fabric deliver. The timing model and the DRAM store go away completely.

**Backend setup.**

- Remote side: an NVMe-oF target exporting the SSD (SPDK `nvmf_tgt` or kernel `nvmet`), RDMA preferred for latency.
- BF3 side: `bdev_nvme_attach_controller` (RDMA or TCP) creates the bdev; the NVMe Emulation app's subsystem uses it as namespace 1. Both normal reads and SODE hops go through this bdev.
- LBA format must be the same end to end. The host computes `slba` with its `lba_shift`; the engine passes LBAs straight to the bdev, so the emulated namespace must report the remote namespace's block size. Use 4 KiB if possible to match XRP's 4 KiB blocks.
- Capacity: the remote SSD must hold the YCSB and BPF-KV databases. The README's cached-DB set is 4.9 TB, so plan the SSD size or the subset of figures accordingly.

**Control configuration for interpreting results.**

| Backend | Purpose |
| --- | --- |
| NVMe-oF to the real SSD | The headline numbers |
| NVMe-oF to a remote `malloc` bdev | Isolates fabric cost from media cost |
| Local `malloc` bdev on BF3 (small DB that fits in BF3 DRAM) | Closest to the paper's NVM-mode setting; upper bound on speedup |

**Files.**

- Removed: `src/emulator/ssd.c`, `ssd.h`, `ssd_config.h`, `simple_ftl.c`, `conv_ftl.c`, `channel_model.c`, `zns_*`, `kv_ftl.c`, `append_only.c`, `bitmap.c`, `pqueue/`; `nvmev_init_latency_emulation` and the `nsecs_target` gating in `src/emulator/io.c`.
- Removed setup: the `memmap` and `intel_iommu=off` GRUB steps in `README.md`.
- New: backend section of the BF3 RPC setup script (Mod 4).

## Mod 7: Experiment harness and target figures

**Purpose.** Keep the benchmarks, parsers and plotting scripts; replace only what assumes an emulator on the host. Target the latency figures first, because the goal is the latency speedup.

**Target figures, in order.**

| Priority | Figure | Measures | Script |
| --- | --- | --- | --- |
| 1 | Fig. 5(d) | BPF-KV average latency vs index depth, 1 thread | `eval/bpfkv_single_thread/` |
| 1 | Fig. 5(c) | BPF-KV p99.9 latency vs index depth, 1 thread | `eval/bpfkv_single_thread/` |
| 1 | Fig. 12 | Normalized latency reduction of SODE | `eval/figure/fig12.py` (check its inputs) |
| 2 | Fig. 5(a), 5(b) | BPF-KV p99 and p99.9, depth 6 | `eval/bpfkv_multi_threads/` |
| 2 | Fig. 7(a) | BPF-KV range-query average latency | `eval/bpfkv_range_query/` |
| 3 | Fig. 8(b) | BPF-KV latency vs throughput, 12 threads | `eval/bpfkv_throughput_latency/` |
| 3 | Fig. 10 | WiredTiger p99 vs client threads | `eval/wiredtiger_tail_latency/` |

BPF-KV first: smaller DBs, one BPF program per query type, and no parallel mode, so fewer moving parts than WiredTiger. It still runs on an ext4 file (`utils/mount_disk.sh`, `O_DIRECT`), so the extent path from Mod 1 is exercised from day one. Each figure compares XRP and SODE on the same emulated device.

**Harness changes.**

- Drop `utils/disable_cpu_freq_scaling.sh` calls (e.g. `eval/bpfkv_single_thread/run_full_exp.sh:34`, `eval/test/test_bpfkv.sh:39`). Replace with a BF3 health check: app running, namespace visible, SODE program registered.
- `numactl --membind=0 --cpunodebind=0` in the run scripts can stay if the host has node 0; the second node is no longer reserved for emulation.
- Add per-hop instrumentation: the descriptor already has `xrp_ebpf_time` and arrival timestamps. Log device-side hop count, bdev read time and BPF time so speedup can be decomposed (see Risks).
- `eval/create_db.sh`: size DBs to the remote SSD; consider generating only the configs the target figures need.
- `eval/figure/*.py`: no change expected except axis labels or adding the control-backend series.

**Files.** `eval/run.sh`, `eval/test/test_bpfkv.sh`, `eval/test/test_wiredtiger.sh`, `eval/bpfkv_*/run_*_exp.sh`, `eval/wiredtiger_*/run_*_exp.sh`, `eval/create_db.sh`, `utils/disable_cpu_freq_scaling.sh` (removed).

# Porting plan

Port in six milestones, each ending in something measurable. The baseline comes first: XRP runs entirely on the host, so it works on the BF3 device before any SODE code exists. Every later milestone keeps a working fallback, because the host already handles a request the device hands back (`status <= 0`).

No code changes start until you approve this plan. Proposed layout: a new branch, a new `src/bf/` directory for the NVMe Emulation app patch, the SODE engine and the setup scripts. It starts as a verbatim copy of `src/emulator/io.c` and the headers it includes, so every BF3 change shows as a diff against the emulator. `src/emulator/` stays as the reference implementation.

## M0: Bring-up and baselines

**Goal.** A stock BF3 NVMe-oF-backed device under the SODE kernel, with baseline numbers.

- Boot the SODE kernel (5.12) on the host with IOMMU on. If it fails, decide on a kernel forward-port before anything else.
- On BF3: check DOCA ≥ 2.7.0 and firmware ≥ 32.41.1000; build the stock NVMe Emulation app with SPDK 23.01.
- Attach the remote SSD with `bdev_nvme_attach_controller`; create the subsystem and listener; confirm `/dev/nvme0n1` on the host with 4 KiB LBA format.
- Run `fio` 4 KiB random reads, queue depth 1, for a latency floor. Run the same against a remote `malloc` bdev.
- Run BPF-KV with XRP for Fig. 5(c) and 5(d).

**Exit.** Baseline Fig. 5(c,d) data on BF3, and a per-I/O latency floor for both backends.

## M1: Opcode probe

**Goal.** Prove `0x82` reaches our code with `rsvd2` intact.

- Add the hook in `nvmf_doca_on_fetch_nvm_sqe_complete()`: log opcode, `rsvd2`, `slba`, PRP1, then serve it as a plain read.
- Host kernel unchanged. The descriptor keeps `status = 0`, so the host runs the whole traversal itself, XRP-style.

**Exit.** `eval/test/test_bpfkv.sh` passes with `--use-sode` on BF3. Logged `rsvd2` values are non-zero and stable.

## M2: Descriptor round trip (Mods 1 and 2)

**Goal.** BF3 can read and write the host descriptor by DMA.

- Host: shared header, DMA-mapped descriptor and scratch, IOVA in `rsvd2`, sync calls in the completion path.
- Device: DMA-read descriptor, extents and scratch; check `version`, `num_extents`, `lba_shift` against host-side logs; write back `status = -1` and a magic value in a counter field.

**Exit.** 1 million SODE requests with every descriptor validated, no IOMMU faults, host still completing via fallback.

## M3: Single-mode engine for BPF-KV get (Mod 5, hard-coded program)

**Goal.** The first real on-device traversal and the first SODE latency numbers.

- Before hardware: build an offline test that runs uBPF with `benchmark/BPF-KV/sode-bpf/get.c` over a BPF-KV DB file using `pread`. It checks the hop logic and uBPF against XRP lookup results for the same keys, on x86 and on BF3 Arm.
- Port the single-mode hop loop and selective check into the app; load `get.o` at startup (Mod 3 fallback).
- Add per-hop counters: hops on device, bdev read time, BPF time.

**Exit.** BPF-KV get results identical to XRP; Fig. 5(c,d) for XRP and SODE; latency breakdown per hop.

## M4: Program registration and range queries (Mod 3)

**Goal.** Any SODE program, keyed by prog ID, with no hard-coding.

- Kernel: keep pre-verifier instructions; push and unload admin commands from the NVMe driver hook.
- Device: admin handler and ID-to-VM table; unknown ID hands the request back.
- Add `range.c`; check multi-threaded behaviour and the selective policy under load.

**Exit.** Fig. 7(a), Fig. 5(a,b), and Fig. 12 for BPF-KV.

## M5: Parallel mode and WiredTiger

**Goal.** WiredTiger SODE, including the 4-way parallel subtasks.

- Confirm uBPF handles `wt_bpf.c`'s BPF-to-BPF calls in the offline test first.
- Port the subtask fan-out and merge with `spdk_thread_send_msg`; DMA subtask scratches.
- Size DBs to the remote SSD.

**Exit.** Fig. 10 and the WiredTiger part of Fig. 12, results matching XRP.

## M6: Controls and full runs

**Goal.** Numbers that can be explained.

- Repeat the target figures on the remote `malloc` and local `malloc` backends (Mod 6).
- Fig. 8(b) if time allows.
- Write up the gap to the paper using the per-hop breakdown.

**Exit.** All priority 1 and 2 figures on three backends.

## Decisions needed from you before M0

- Approve the `src/bf/` layout and the branch.
- The NVMe-oF transport and SSD model behind BF3.
- Whether the host IOMMU can be on. M2 assumes it is.
- Whether a remote `malloc` target is available for the control runs.

## Sources

- [DOCA NVMe Emulation Application Guide](https://networking-docs.nvidia.com/doca/sdk/doca+nvme+emulation+application+guide), DOCA 3.5.0
- [DOCA DevEmu PCI Generic](https://docs.nvidia.com/doca/archive/3-2-0/DOCA+DevEmu+PCI+Generic), DOCA 3.2.0
- [DOCA SNAP-4 Service Guide](https://networking-docs.nvidia.com/doca/sdk/doca-snap-4-service-guide) and [Advanced Features](https://networking-docs.nvidia.com/doca/archive/3-5-0/snap-4-service-advanced-features/), DOCA 3.5.0
- Repository code at commit `9366538ed`
