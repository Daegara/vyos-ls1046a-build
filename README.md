[![VyOS LS1046A build](https://github.com/mihakralj/vyos-ls1046a-build/actions/workflows/self-hosted-build.yml/badge.svg)](https://github.com/mihakralj/vyos-ls1046a-build/actions/workflows/self-hosted-build.yml)

# VyOS for Mono Gateway Development Kit (NXP LS1046A)

This repo builds VyOS for the aarch64 [Mono Gateway Development Kit](https://docs.mono.si/gateway-development-kit/hardware-description) based on latest [VyOS 'rolling' release](https://vyos.net/get/nightly-builds/). New [builds](https://github.com/mihakralj/vyos-ls1046a-build/releases) are released each Friday.

**This is the first and only VyOS build for bare-metal aarch64 networking hardware with support for both ASIC HW-offload *and* VPP.**

**The hardware earns the effort.** The Mono Gateway Development Kit is build around the [NXP LS1046A](https://www.nxp.com/docs/en/data-sheet/LS1046A.pdf) - four Cortex-A72 cores at 1.6 GHz, with a hardware ASIC alongside which chews through packets before the CPU even notices they've arrived. The Mono Gateway Development Kit further adds 8 GB of ECC DDR4, three RJ45 ports, and two SFP+ cages - an ideal package for HW-offloaded, wire-speed networking on aarch64.

**There is nothing in this class for consumer home routers** and that gap is an opportunity. Historically, NXP sold the LS1046A to telecoms carriers and switch vendors, and provides a generalised [Application Solutions Kit (ASK)](https://www.nxp.com/design/design-center/software/embedded-software/software-for-industrial-networking/gateway-ask:VORTIQA-ASK) to control the HW-offload network accelerator functions. In developing the Gateway Development Kit, Mono purchased and with permission [released](https://github.com/we-are-mono/ASK) the ASK source under GPL-2.0.

**VyOS enables pushing this HW to its full potential with aarch64 becoming a first-class citizen in 1.5.x.**  This repo documents the development of ASK HW-offloading rebuilt to modern standards as ASK2, and the compliment this provides to VPP on this HW.

## Getting Started

| **I want to...**                 | **Go to...**                                                                                                        |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Use VyOS** on the Mono Gateway | **[INSTALL.md](INSTALL.md)**: Start here.                                                                           |
| **Understand the HW**            | [HARDWARE.md](HARDWARE.md): Physical HW, and logical network architecture                                           |
| **Update Firmware**              | [FIRMWARE.md](FIRMWARE.md): A brief how-to                                                                          |
| **Control HW & diagnose issues** | [HWCTL.md](HWCTL.md):  Control the main LEDs with `led` & diagnose issues with the seven built-in `*-check` scripts |
| **See what changed**             | [plans/CHANGELOG.md](plans/CHANGELOG.md): per-build changelog                                                       |
| **Understand how this started**  | [STARTING-GATE.md](STARTING-GATE.md): Getting mainline VyOS to work (at all)                                        |
| Understand work towards ASK2     | [`plans/ASK-PLANS.md`](ASK-PLANS.md)Index and source-of-truth                                                       |
| Dig into the detail              | [RABBITHOLE.md](RABBITHOLE.md): Down you go...                                                                      |
|                                  |                                                                                                                     |
## Build and Release Assets

Automated weekly (Friday 01:00 UTC) via GitHub Actions.

| File                          | Description                                                                                                                                    |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `*-LS1046A-arm64.iso`         | **Hybrid ISO** — boot from USB (`dd if=...iso of=/dev/sdX bs=4M`) for live install, or `add system image <url>` to upgrade an installed system |
| `*-LS1046A-arm64.iso.minisig` | ISO signature ([verify key](data/vyos-ls1046a.minisign.pub))                                                                                   |
| `vyos-packages.tar`           | Built kernel + vyos-1x `.deb` packages                                                                                                         |

## What This Build Actually Delivers

This is, as far as anyone can tell, the only VyOS build targeting bare-metal ARM64 networking hardware with HW offload and VPP. Some highlights from the wreckage:

- **10G SFP+ with VPP kernel bypass.** AF_XDP on eth3/eth4 polls at 2.47M packets/sec. The stock kernel path grinds through ~3-5 Gbps on those same ports. Not a typo.
- **CAAM hardware crypto.** IPsec AES-GCM offload via 3 Job Rings at ~2-3 Gbps encrypted throughput. WireGuard (ChaCha20-Poly1305) cannot use CAAM: it runs on ARM64 NEON SIMD instead, topping out around 1 Gbps. Pick your poison.
- **DPAA1 Frame Manager.** Five-port hardware packet engine handles parsing, core distribution, and buffer management before the CPU sees a single byte. Jumbo frames at 9578 MTU on RJ45. SFP+ ports cap at 3290 MTU under AF_XDP (DPAA1 XDP hard limit on `fsl_dpaa_mac`).
- **PTP hardware timestamping.** Nanosecond precision via `ptp_qoriq` on `/dev/ptp0`.
- **U-Boot direct boot.** `vyos.env` on the ext4 partition selects the active image. No GRUB, no OOM, no overhead. Image upgrades write the file automatically.
- **~80s cold boot to login prompt.** Single boot, no kexec double-reboot. `CONFIG_DEBUG_PREEMPT` suppressed saves ~20s of cosmetic scheduler spam.
- **HW-accelerated AF_XDP datapath.** The DPAA1 driver is modernized into a single shared kernel binary with `ndo_xsk_wakeup`, XSK-backed BMan pools, per-CPU NAPI on dedicated QMan channels, and FMan HW offloads (CC / HM / Policer / CEETM). The earlier DPDK DPAA PMD path was abandoned (RC#31: `dpaa_bus` probe globally disrupts kernel FMan interfaces); AF_XDP is the production kernel+VPP coexistence path. Design and milestone status: [specs/dpaa1-afxdp-modernization-spec.md](specs/dpaa1-afxdp-modernization-spec.md).

## Why VyOS?

**Zone-based firewall with nftables.** Interfaces join zones; policy applies per zone-pair. `set firewall zone DMZ from LAN firewall name LAN-DMZ-v4` replaces dozens of per-interface rules that multiply combinatorially as you add ports. Anyone who has managed large iptables rule sets knows exactly what that cost feels like.

**VPP userspace dataplane.** VyOS 1.5 ships VPP (Vector Packet Processing): a kernel-bypass data plane that batches 256 packets per poll cycle, reads hardware queues directly, and treats the Frame Manager as a co-processor. Critically, VPP is not all-or-nothing: only interfaces explicitly assigned to VPP use the VPP forwarding path. eth3/eth4 (10G SFP+) go to VPP; eth0-eth2 (RJ45) stay with the kernel for management and routing. VPP is off by default. See [plans/VPP.md](plans/VPP.md). The CAAM crypto engine provides 128 hardware algorithms for IPsec AES-GCM offload at ~2-3 Gbps encrypted.

**Native containerization via Podman.** Containers are first-class config-tree citizens: image, network, environment variables, volumes, ports, memory limits, all defined with `set container name ...` and committed alongside routing and firewall config. Run AdGuard, a monitoring agent, or a DDNS updater directly on the router. Roll it back if you regret it.

**CLI-first, text-based config.** The entire router state lives in `config.boot`: hierarchical, human-readable, diffable. A diff between two VyOS configs reads like plain English. No GUI-only system can match that at 2 AM when something is on fire.

**Transactional atomic commits.** Stage multiple changes, review with `compare`, apply with `commit`. If commit fails, nothing changes. `commit-confirm` provides automatic rollback on a timer. The number of network outages this design has prevented is unquantifiable. Use it.

**Full FRRouting integration.** BGP, OSPF, IS-IS, BFD, MPLS, VXLAN, segment routing, PIM: all in the config tree with proper dependency resolution at commit time. Full stack, no glue scripts, no surprises.


### DPAA1 Driver Modernization

An ongoing effort modernizes the mainline DPAA1 driver into a single shared kernel binary (consumed in different runtime modes — kernel `default`, `vpp` AF_XDP, `ask` offload, all shipping in one image) with HW-accelerated AF_XDP and four FMan/QMan hardware offloads. Full design and per-milestone status: [specs/dpaa1-afxdp-modernization-spec.md](specs/dpaa1-afxdp-modernization-spec.md).

**Shipping and board-validated today:**

- **Flavor-ops abstraction (M0)** — per-`dpaa_priv` ops tables, RCU-NULL-safe; byte-identical to mainline when no flavor module is loaded.
- **AF_XDP zero-copy plumbing (M1–M3-3)** — `ndo_xsk_wakeup`, XSK-backed BMan pool, per-CPU NAPI + dedicated QMan channels per qband, cluster-aware pinning. Driver proven to drop **0%** at line rate; ~5.57 Gbit/s aggregate RX measured (bottleneck is the single userspace receiver, not the NIC).
- **HW capability layer** — FMan PCD caps live-probed (`0x17` = CC HM POL PARSER on ucode 210).
- **HM VLAN-strip offload (M3-3c)** — live on hardware (`ethtool -k` → `rx-vlan-offload: on`).
- **Policer + CEETM scaffolds (M3-3d/e)** — install/stub APIs compiled in and cap-probed, stable contracts for the VyOS CLI consumers.

**What remains for a feature-complete driver** (see the spec's "What remains for a complete DPAA1 driver" table):

- **Two real kernel forward-ports** — the FMan PCD subsystem (unblocks CC steering and the productive HM/Policer datapaths) and the QMan-CEETM driver (~4500 LOC, absent from mainline 6.18, needed for HW egress shaping).
- **Non-kernel glue** — vyos-1x CLI consumers for HM/Policer/CEETM, a traffic generator for the functional datapath gates, and a multi-core receiver to record the literal ≥7 Gbps figure.

No further *architectural* work is required — the ops abstraction and capability layer already accommodate every remaining consumer.

## License

VyOS sources are GPLv2. ARM64 builder image from [huihuimoe/vyos-arm64-build](https://github.com/huihuimoe/vyos-arm64-build). Hardware documentation from [mono-gateway-docs](https://github.com/ryneches/mono-gateway-docs).