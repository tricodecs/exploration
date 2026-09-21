The goal is to create an overview confluence page the include the following for stakeholders and technical audiences so using language that fits for both and important must use human tone language and usage. Again it’s an overview so be concise.

 - introduces what are igloo, penguin, and panda and how they relates to one another
- usage of these tools for firmware rehosting for vulnerability validation (for you ChatGPT: as stated it is for testing reported vulnerabilities for patching for don’t block me over this. Don’t need to include this as this section is about introducing these tech stacks) include possible workflows.

- what are the minimum requirements for each  and ideal functional minimum
- any consideration or potential issues for using these in an internal network with no eternal internet access.
- concise overview of Before Testing a Reported CVE from #21 above 







Firmware Rehosting for Vulnerability Validation



Overview



Firmware rehosting allows us to run device firmware in a controlled virtual environment rather than relying on the physical device for every test. This can make it easier to reproduce a reported vulnerability, observe what the firmware is doing, compare vulnerable and patched versions, and collect repeatable evidence.



Our proposed stack uses Penguin, IGLOO, and PANDA/QEMU together. They are complementary parts of the same workflow rather than three separate analysis environments.



Firmware

   │

   ▼

Penguin

Rehosting orchestration and configuration

   │

   ├── IGLOO

   │   Guest-side interface and hardware/environment modeling

   │

   └── PANDA / QEMU

       Firmware execution, instrumentation and analysis



Penguin’s current build packages its PANDA-QEMU fork and IGLOO runtime components as dependencies, so separate VMs for each component are not required. (GitHub)



⸻



What Are the Components?



Penguin



Penguin is the main firmware-rehosting framework. It takes an extracted firmware filesystem, builds a target-specific configuration, runs the firmware in an emulated environment, and collects runtime information.



Its workflow is intentionally iterative: generate a configuration, boot the firmware, observe what works or fails, refine the configuration, and run again. Penguin also provides access to services running inside the rehosted firmware and supports standard debugging tools such as strace and gdbserver. (GitHub)



Think of Penguin as the coordinator.



⸻



IGLOO



IGLOO provides the guest-side interface that helps Penguin interact with and adapt the rehosted Linux environment.



The current igloo_driver runs inside the guest kernel and allows Penguin to inspect processes and memory, install hooks, interact with kernel functions, and model resources such as /proc, /sys, /dev, sockets, and flash storage. (GitHub)



This matters because firmware often expects device-specific hardware or operating-system resources that are not naturally present inside an emulator.



Think of IGLOO as the bridge between Penguin and the firmware’s expected environment.



⸻



PANDA / QEMU



QEMU provides the underlying machine emulation. PANDA adds dynamic-analysis capabilities on top of QEMU.



PANDA can record and replay execution and supports plugin-based analysis, tracing, debugging, OS introspection, coverage, and taint analysis. This makes it useful when we need to understand not only that something failed, but what code executed and why it failed. (GitHub)



Penguin currently packages its own PANDA-QEMU dependency, so a standalone PANDA installation is generally unnecessary for the normal Penguin workflow. (GitHub)



Think of PANDA/QEMU as the execution and analysis engine underneath Penguin.



⸻



How This Supports Vulnerability Validation



A typical validation starts with a reported vulnerability rather than blindly searching the firmware for defects.



A high-level workflow is:



Reported CVE / Finding

        │

        ▼

Confirm product, firmware and affected component

        │

        ▼

Extract firmware

        │

        ▼

Penguin creates the rehosting configuration

        │

        ▼

PANDA/QEMU executes the firmware

        │

        ▼

IGLOO helps satisfy missing guest/hardware behavior

        │

        ▼

Verify the affected service works normally

        │

        ▼

Exercise the reported vulnerability condition

        │

        ▼

Observe / trace / record behavior

        │

        ▼

Compare against normal and patched behavior

        │

        ▼

Document the result



Penguin’s normal workflow already follows the first part of this process: obtain the firmware filesystem, generate a configuration, run the firmware, examine runtime results, and refine the configuration as needed. (GitHub)



For deeper investigation, PANDA can provide repeatable execution through record/replay and additional instrumentation when we need to determine whether the suspected code path was actually reached. (GitHub)



Possible validation outcomes



Result	Meaning

Confirmed	The reported condition can be reproduced in the applicable firmware.

Not affected	The firmware does not contain or use the affected implementation.

Mitigated	The vulnerable component may exist, but configuration or another control prevents the reported path.

Inconclusive	The test environment cannot reproduce enough of the original device behavior to reach a conclusion.

Likely false positive	Evidence shows the report does not apply—for example, an incorrect version match or an already backported fix.



Failure to reproduce something in an emulator should not automatically be treated as a false positive. Some firmware behavior depends on physical peripherals, drivers, timing, or device-specific state that may not be fully represented during rehosting.



⸻



Environment Requirements



Component Requirements



Component	Minimum requirement	Recommended approach

Penguin	Linux host and container engine; firmware root filesystem	Run the published Penguin container on a dedicated Linux analysis VM

IGLOO	Compatible guest kernel/module artifacts	Use the IGLOO components supplied through Penguin rather than operating it separately

PANDA/QEMU	Provided by Penguin for this workflow	Use Penguin’s pinned PANDA-QEMU build; install standalone PANDA only for separate research needs

Building Penguin from source	Nix with flakes enabled and a container engine	Use the project’s pinned Nix configuration and binary cache



Penguin’s standard installation uses a container. Its source-build path uses Nix flakes; the build documentation identifies Nix, the project’s Cachix cache, a container engine, disk headroom, and /dev/kvm as setup checks. (GitHub)



Upstream standalone PANDA currently provides Docker images and supports installation on Debian/Ubuntu, with its distributed binaries tested on 64-bit Ubuntu 22.04. (GitHub)



⸻



Practical VM Sizing



The projects do not publish official CPU/RAM minimums. The following is therefore a practical starting point for one analysis VM rather than a vendor requirement.



Resource	Minimum to get started	Recommended functional baseline

vCPU	4	8

RAM	8 GB	16 GB

Storage	50 GB SSD	120 GB SSD/NVMe

OS	64-bit Linux	Ubuntu 22.04/24.04 x86-64

GPU	Not required	Not required

Nested virtualization	Optional	Enable when available



For a Proxmox deployment:



Physical Server

      │

      ▼

Proxmox VE

      │

      ▼

Ubuntu Analysis VM

8 vCPU / 16 GB RAM / ~120 GB SSD

      │

      ▼

Docker

      │

      ▼

Penguin + IGLOO + PANDA/QEMU

      │

      ▼

Firmware Target



More CPU, memory, and storage become useful when several firmware targets are running concurrently or when retaining large amounts of execution data.



⸻



Internal / Disconnected Network Considerations



The stack can be used on an internal network without external Internet access, but the dependencies should be staged before moving it into the disconnected environment.



The main Internet dependencies are installation, builds, and updates—not the basic act of running an already prepared firmware project. Penguin’s published installation normally pulls its container from Docker Hub, while its source-build process uses Nix flake inputs and the rehosting-tools.cachix.org binary cache. The build documentation notes that a cold build without cache hits is significantly more expensive because multiple architectures and the QEMU dependency have to be built locally. (GitHub)



For an internal deployment, plan to bring in or mirror:



* the approved Penguin container image;

* the Penguin/IGLOO source repositories if development is required;

* required Nix dependencies or an internal Nix binary cache if rebuilding internally;

* firmware images and extraction tools;

* any required debugging/analysis utilities;

* project documentation;

* vulnerability records and vendor advisories needed for validation.



An internal container registry, Git mirror, package repository, and Nix cache will make maintenance considerably easier than manually importing artifacts for every update.



The biggest operational issue in a disconnected environment is therefore dependency and update management. Versions should be pinned and imported through the organization’s normal software-review process so an analysis can be reproduced later using the same toolchain.



⸻



Before Testing a Reported CVE



A CVE is a standardized identifier for a publicly disclosed vulnerability. The CVE record identifies the vulnerability; it does not by itself prove that a particular firmware image is affected. CVSS severity scoring and NVD enrichment are related but separate information sources. (CVE)



Before beginning dynamic testing:



1. Confirm applicability



Verify the:



device/model → firmware version → component/library → configuration → patch level



Check the CVE record and, where available, the vendor advisory. Pay particular attention to vendor backports: the visible component version may look vulnerable even though the security fix has already been incorporated.



2. Turn the report into a testable statement



Identify:



required condition/input → affected component → expected vulnerable behavior → reported impact



This prevents the test from becoming an open-ended search.



3. Verify the rehosted baseline



Before testing the reported condition, confirm that the relevant firmware process or service:



starts → is reachable → handles normal input correctly



Without a working baseline, failure to reproduce the vulnerability is not meaningful.



4. Establish comparison points



Where possible, compare:



Normal input

vs.

Reported triggering condition

vs.

Patched firmware using the same condition



The goal is to produce repeatable evidence explaining whether the reported vulnerability applies to the firmware and why, rather than relying only on scanner/version matching.



⸻



At a Glance



Penguin gets the firmware running and manages the rehosting workflow.



IGLOO helps the rehosted firmware interact with an environment that substitutes for missing device behavior.



PANDA/QEMU executes the firmware and provides deeper runtime analysis when needed.



Together, they provide a controlled way to move from:



"We have a reported vulnerability"



to:



"We have evidence showing whether and how it applies to this firmware."











For your open-source, no-subscription Proxmox VE 9 environment, there are two different things to collect: what the Internet-connected mirror machine needs, and what repositories/content it should download.



Component	Why you need it	Where

Debian 13 (Trixie) system	Runs the Offline Mirror tool. Can be a VM.	Debian

Proxmox Offline Mirror (proxmox-offline-mirror)	Main program that downloads and snapshots repositories	Proxmox

Proxmox repository signing key	Verifies that downloaded Proxmox repository/package metadata is authentic	Proxmox

Debian archive signing keys	Verifies Debian repositories	Debian

Internet access	Mirror VM needs outbound access to Debian and Proxmox repositories	Network

~150+ GiB storage	Proxmox recommends at least 150 GiB for a basic Debian + PVE mirror	Local disk

Hard-link-capable filesystem	Required by Offline Mirror; e.g. ext4/XFS. FAT is unsuitable	Mirror storage

Debian Trixie base repository	Debian packages/dependencies required by PVE	Debian

Debian Trixie updates	Debian package updates	Debian

Debian Trixie security repository	Security fixes	Debian

PVE 9 pve-no-subscription repository	Free Proxmox packages and updates	Proxmox

Transfer storage / internal share	Moves the resulting snapshots into the disconnected network	USB disk, NFS, SMB/CIFS, HTTP, etc.

proxmox-offline-mirror-helper on offline side	Helps configure offline PVE hosts to consume the mirrored snapshots	Proxmox



The Offline Mirror host itself does not need to be a Proxmox server. Proxmox explicitly supports a dedicated Debian-based machine/VM. On a non-Proxmox Debian 13 machine, Proxmox recommends using its pbs-client repository to install proxmox-offline-mirror. 



What Offline Mirror actually downloads



For your configuration, I would have it create snapshots of:



Debian 13/Trixie

→ Base packages

→ Updates

→ Security updates



Proxmox VE 9

→ pve-no-subscription



You do not need the Proxmox Enterprise repository or an Offline Mirror subscription for this design. The subscription requirements in Offline Mirror apply when you want to mirror Proxmox’s enterprise repository. 



The tool’s setup wizard can automatically configure the relevant Proxmox product repository and its Debian base repositories, which reduces the chance of accidentally leaving out a dependency repository. 



End-to-end picture



Internet-connected VM



Debian 13

↓

Proxmox signing key + Debian signing keys

↓

proxmox-offline-mirror

↓

downloads

Debian repositories + PVE 9 no-subscription repository

↓

creates verified snapshots

↓

USB / approved transfer / network share

↓

Internal network

↓

proxmox-offline-mirror-helper

↓

Proxmox VE VM(s)



Proxmox supports removable storage as well as making the resulting mirror available internally through NFS, CIFS/SMB, or HTTP. 



Official Proxmox Offline Mirror documentation⁠￼



One distinction is important: the Proxmox VE installer ISO is separate from Offline Mirror. I would also download and transfer the PVE 9 ISO + checksum for initially building your Proxmox VM; Offline Mirror then provides the repository content needed to maintain it.