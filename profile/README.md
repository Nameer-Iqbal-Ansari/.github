## MultiCore SoC For SMP Linux

### Description
This repository contains the code and resources for this project — implementing a custom quad-core RISC-V processor on AWS FPGA and porting SMP Linux on it. The design is built on **OpenPiton**, an open-source, tiled manycore framework from Princeton University, paired with the **Ariane** (CVA6) 64-bit RISC-V core. Additionally, the project demonstrates customization of hardware and software for specific applications, and includes a forward-looking proposal for migrating the coherence layer to industry-standard AMBA CHI.

### Project Overview

**AWS FPGA with Linux Port:** This project has successfully ported a custom quad-core processor on AWS FPGA and booted SMP Linux on it. This achievement allows us to explore the interaction between hardware and software, showcasing the power of multicore systems on a cloud-based FPGA platform.

**Arty A7 FPGA with Zephyr Boot:** In addition to the AWS FPGA implementation, this project has also worked on the Arty A7 FPGA board (via the Litex `wex_riscv` core) and ported the Zephyr RTOS on it. This step provides an opportunity to understand the workings of different FPGAs and operating systems.

**Custom Hardware-Software Application:** This project takes the customization further by adding GPIOs, LEDs, a Seven Segment display, and a VGA controller to the hardware design. These hardware components are integrated into the software application, enabling seamless interaction and control through GPIOs and displaying relevant information on the Seven Segment display.

**VGA Controller:** This project also implements a VGA controller to enable video output from the FPGA, allowing graphical information to be displayed and interacted with through a VGA-compatible monitor.

### Documents

- [Final Year Project Report — MultiCore SoC For SMP Linux](docs/MULTICORE_SoC_For_SMP_Linux.pdf) — full write-up of the OpenPiton+Ariane architecture, FPGA implementation, software flow, and results.
- [OpenPiton CHI Migration Proposal](docs/OpenPiton_CHI_Migration_Proposal.pdf) — phased RTL design proposal for migrating OpenPiton's native coherence protocol to AMBA CHI.

*(Links assume both PDFs are placed in a `docs/` folder in this repo; update the paths if you store them elsewhere.)*

---

### Architecture: OpenPiton + Ariane

*Full details in the [FYP Project Report](docs/MULTICORE_SoC_For_SMP_Linux.pdf).*

The SoC is generated using **OpenPiton**, an open-source framework for building scalable, cache-coherent manycore chips, together with **Ariane**, a 6-stage, single-issue, 64-bit RISC-V core (RV64GC, implementing the I, M, and C extensions and M/S/U privilege levels).

- **Tiles:** Each tile contains an Ariane core, an L1.5 cache, an L1.5 arbiter, an L2 cache slice, and three NoC routers (NoC1–NoC3).
- **Cache hierarchy:**
  - L1 I$/D$: 8 KB, 4-way set associative (per-core, inside Ariane).
  - L1.5: 8 KB, 4-way set associative, inclusive of L1-D, acts as the bridge between the Ariane core's CCX bus and the NoC/L2 protocol via the Transaction Response Interface (TRI).
  - L2: distributed, shared, write-back cache with directory-based MESI coherence, 64 KB per slice, 4-way set associative.
- **On-chip network:** Three physical, priority-ordered Networks-on-Chip (NoC1 > handles L1.5→L2 requests, NoC2 handles L2→L1.5 forwards/downgrades and L2→memory requests, NoC3 carries acknowledgements/writebacks and is never blocked). This strict priority ordering, plus point-to-point ordering per NoC, gives the protocol its deadlock freedom and its TSO memory-ordering guarantee.
- **Coherence protocol:** A directory-based MESI protocol, with sharer/state information tracked in the L2's co-located directory and state arrays (currently a 64-bit sharer vector, i.e. up to 64 cores per L2 slice).
- **Chipset/peripherals:** ROM, Debug Module, CLINT, PLIC, UART, Ethernet, SD card, and DRAM controller, connected via an I/O crossbar and packet filter (`IO_CTRL_TOP`) that bridges NoC traffic to AXI/Wishbone-based peripherals.
- **Scalability:** Intra-chip scaling via a 2D mesh (P-Mesh) topology, and inter-chip scaling via the chip bridge, which multiplexes the three NoCs over a single off-chip link.

### Toolchain

- **OpenPiton** – manycore generation, test infrastructure (unit, single-core, multicore regressions), and simulation scripts (`sims`, `protosyn`).
- **Verilator** – open-source Verilog/SystemVerilog simulation for functional verification and waveform debugging (GTKWave).
- **Xilinx Vivado** – synthesis, implementation, and HLS for FPGA bring-up (AWS F1, Genesys2, VCU118, VC707, Nexys Video, XUPP3R boards supported).
- **riscv-gnu-toolchain** – RISC-V cross-compilation for baremetal apps, diagnostics, and Linux.

### Results

- Booted a bare-metal "Hello World" across all 4 harts on a 2×2 tile configuration.
- Passed full OpenPiton regression suite (unit, single-core, and multicore diagnostics).
- Generated and deployed an AWS FPGA Image (AFI) to an F1 instance; validated UART runtime driver and DMA-based memory loading.
- Successfully booted SMP Linux across all 4 cores on AWS FPGA.
- Post-synthesis utilization on the target FPGA: ~70% LUTs, ~23% FFs, ~26% BRAM, ~6.5% DSPs (see full report for details).
- Booted the Zephyr RTOS on the Litex `wex_riscv` core on the Arty A7 FPGA.

---

### Roadmap: Migrating Coherence to AMBA CHI

Beyond the current implementation, this repository also tracks a **phased proposal to migrate OpenPiton's native coherence protocol to AMBA CHI (Coherent Hub Interface)**, the industry-standard directory-based coherence protocol used across ARM and increasingly RISC-V multi-core/chiplet systems.

*Full detail — including the field-level packet mapping tables, RTL change inventory, and TSO proof — is in the [OpenPiton CHI Migration Proposal](docs/OpenPiton_CHI_Migration_Proposal.pdf).*

**Motivation.** OpenPiton's native protocol is functionally complete and well-tested, but it is bespoke: its packet formats, conflict handling, and node roles exist only inside OpenPiton. Notable gaps include:
- Coherence transaction conflicts are undocumented in the current specification.
- The directory's sharer vector is hard-limited to 64 bits (64 cores per L2 slice).
- Prefetch requests are no-ops, and block-store requests are decomposed into regular stores before reaching the L2.
- Coherence Domain Restriction (CDR) remains an experimental research feature.

**Proposed conversion (Phase 0 – base protocol conversion).**
- **Node mapping:** L1.5 → CHI Request Node (RN-F); L2 (cache + directory) → CHI Home Node (HN-F) with an explicit order point; memory controller → CHI Subordinate Node (SN-F).
- **Channel mapping:** the three priority-ordered NoCs are replaced by four independently-credited CHI channels — REQ, RSP, DAT, and SNP — carried as four physical link pairs (rather than virtual channels), preserving OpenPiton's existing deadlock-freedom argument at the cost of extra wiring/router area.
- **Conflict resolution:** CHI's standard `RetryAck` / `PCrdGrant` protocol-credit mechanism replaces OpenPiton's undocumented backpressure-only behavior, with a sized RN-F replay buffer (≥6 entries) and an arrival-order, starvation-free credit-grant arbiter at the HN-F.
- **Memory-model preservation (TSO):** Phase 0 explicitly reconstructs OpenPiton's TSO guarantees from CHI's own ordering primitives — in-order issue into the fabric, ordered per-pair REQ/DAT channels, a single per-address order point at the HN-F, and snoop-response-before-completion — plus new RN-F-side gating logic that withholds `CompData` from the core until transaction completion is signaled.
- **Simplifications:** the native `ReqWBGuard` message is retired, since the HN-F's single order point already enforces the ordering it existed to provide.
- **Non-coherent sideband:** IPI delivery and diagnostic/configuration accesses are preserved via a reserved opcode range on the RSP channel during the transition, rather than a fifth physical lane.

**Later phases (deferred, separately reviewable):**
- **Phase 1** – richer CHI snoop types (`SnpOnce`, `SnpUnique`, `SnpCleanInvalid`), CHI-native exclusive atomics, and reactivation of block-store (`WriteUnique` multi-beat bursts) and prefetch (`ReadNoSnp`/`ReadNotSharedDirty` non-blocking hints).
- **Phase 2** – directory scaling past the 64-sharer limit using CHI's node-ID space, and integration with Coherence Domain Restriction (CDR).
- **Phase 3** – replacing the IPI mechanism with CHI Distributed Virtual Memory (DVM) broadcast operations, and extending multi-chip coherence to run CHI natively rather than OpenPiton's native inter-chip routing protocol.

Each phase is intended to be independently mergeable, with a build-time toggle keeping the native protocol available (and the ~9,000-test OpenPiton regression suite passing) throughout the transition. See the full proposal document for the complete RTL change inventory, field-level packet mapping tables, and formal-verification requirements.

---

### References & External Links

- [OpenPiton (Princeton, GitHub)](https://github.com/PrincetonUniversity/openpiton)
- [OpenPiton Microarchitecture Specification](http://parallel.princeton.edu/openpiton/docs/micro_arch.pdf)
- [OpenPiton Simulation Manual](http://parallel.princeton.edu/openpiton/docs/sim_man.pdf)
- [OpenPiton FPGA Prototype Manual](http://parallel.princeton.edu/openpiton/docs/fpga_man.pdf)
- [OpenPiton: An Open Source Manycore Research Framework (ASPLOS '16)](http://parallel.princeton.edu/papers/openpiton-asplos16.pdf)
- [Ariane / CVA6 core documentation](https://cva6.readthedocs.io/en/latest/03_cva6_design/index.html)
- [RISC-V International](https://riscv.org/)
- [AWS FPGA HDK/SDK (aws-fpga, GitHub)](https://github.com/aws/aws-fpga/blob/master/hdk/README.md)
- [ARM AMBA 5 CHI Architecture Specification](https://www.arm.com/)

---

### Demonstration