For firmware rehosting, think of them as layers:

* QEMU — the hardware emulator. It imitates CPUs and hardware such as ARM or MIPS so firmware can execute without the original physical device.
* Igloo — the firmware-analysis/rehosting framework. It helps determine what the firmware expects from its original hardware and environment, and helps bridge missing pieces so the firmware can run under emulation.
* Penguin — the higher-level rehosting system. It uses QEMU plus rehosting techniques to create a usable environment for running and testing firmware, with more automation around setup and hardware/environment dependencies.

In simplified form:

Firmware → Penguin → Igloo/rehosting support → QEMU → emulated hardware

QEMU provides the virtual hardware. Igloo helps deal with the gap between that virtual hardware and what the firmware expects. Penguin ties those capabilities together into a more practical firmware-rehosting workflow.

None of them guarantees that arbitrary firmware will run correctly; firmware that depends heavily on proprietary hardware, drivers, peripherals, or device-specific behavior can still fail or behave differently from the real device.






The firmware components most likely to cause false negatives or false positives are the ones tightly coupled to the original hardware or device state.

1. Hardware drivers and proprietary kernel modules — usually the hardest to reproduce. They depend on exact chips, registers, interrupts, DMA, or vendor hardware. A vulnerability here may simply never trigger in rehosting, causing a false negative.
2. Hardware peripherals — radios, sensors, USB devices, flash controllers, cellular modems, GPUs/ASICs, etc. If Penguin has to fake or omit them, behavior can differ substantially from the real device.
3. NVRAM / persistent device state — configuration values, calibration data, credentials, interface settings, boot flags, and device identifiers. Missing or synthetic values can prevent vulnerable code paths from being reached, or accidentally create behavior that would not occur on the real device.
4. Vendor-specific services and IPC — firmware often has multiple proprietary processes communicating through sockets, shared memory, message queues, or custom APIs. If one service fails, dependent services may behave differently.
5. Bootloader/kernel/device-tree interactions — firmware may expect a specific kernel, bootloader, memory map, or device tree. Replacing or modifying these can change how the firmware behaves.
6. Timing, interrupts, and concurrency — race conditions, timing bugs, watchdog behavior, and interrupt-related vulnerabilities are particularly difficult to reproduce accurately under emulation.
7. Hardware-backed security components — TPMs, secure elements, TrustZone, hardware crypto engines, or secure boot components. These are often difficult or impossible to reproduce faithfully without significant additional work.

For your vulnerability-validation use case, a useful rule is:

Application-level vulnerabilities — such as a vulnerable web server, parser, API, or command-injection bug — are generally easier to validate reliably.

Hardware/kernel/driver-dependent vulnerabilities — are much more likely to produce unreliable results.

And typically the biggest danger is a false negative: Penguin says the vulnerability did not reproduce, but only because the required hardware behavior or device state was missing. A false positive is also possible when a stub, replacement driver, or artificial configuration creates behavior that would not exist on the physical device.






Yes. Workarounds can affect how trustworthy the vulnerability test is, depending on what the vulnerability depends on.

* Hardware drivers — Software that lets Linux communicate with specific hardware, such as network chips or storage controllers. If the original driver/hardware is unavailable, Penguin may substitute or bypass it. This matters if the vulnerability involves that driver or hardware behavior.
* Proprietary kernel modules — Vendor-specific code loaded into the Linux kernel. If missing, Penguin may not be able to reproduce the original system behavior at all. Vulnerabilities involving that module would be difficult to validate reliably.
* NVRAM values — Persistent device settings such as configuration, interface names, serial values, or boot settings. Penguin may provide artificial values so applications can start. Wrong values can cause the application to behave differently from the real device.
* Device-specific files — Files the firmware expects to exist, such as configuration files, certificates, device nodes, or runtime data. Penguin may need to create or replace them. The closer the replacement is to the real device, the more reliable the test.
* Peripherals — Physical components such as sensors, radios, USB devices, LEDs, or special network interfaces. Penguin may emulate, ignore, or fake their responses. Vulnerabilities depending on those interactions may not reproduce correctly.
* Vendor services — Background programs supplied by the manufacturer that other applications depend on. If they cannot run, Penguin may need to stub or bypass them.

The key reliability point is:

The more Penguin has to fake or bypass something that the vulnerability depends on, the less confidence you should have in the result.

For example, a vulnerability entirely inside a web server running in the firmware may still be tested quite reliably even if some hardware is missing. But a vulnerability involving a proprietary network driver, NVRAM configuration, or hardware interaction could produce a false negative or false positive in the rehosted environment.

So for your validation workflow, you should treat Penguin rehosting as high confidence only when the vulnerable component and its important dependencies are faithfully reproduced.





For Penguin in an air-gapped network, the main challenge is that installing Penguin is only the beginning. The harder part is reproducing everything a particular firmware expects in order to run.

I’d present it this way:

1. Penguin and its software dependencies — Penguin relies on components such as QEMU, libraries, Python packages, build tools, and potentially supporting repositories. In an air gap, those must be pre-staged or mirrored internally.
2. Firmware images — Penguin does not create the vendor firmware for you. Each firmware image or bundle has to be obtained externally, approved, and transferred into the internal environment.
3. Firmware-specific dependencies — Different firmware may expect a particular CPU architecture, kernel, filesystem, drivers, startup scripts, networking behavior, storage, NVRAM, or other hardware components. Those dependencies vary significantly from one firmware to another.
4. Hardware emulation is the difficult part — QEMU can emulate common CPU and hardware components, but some firmware depends on proprietary or device-specific hardware that cannot be reproduced accurately. Penguin may work around some of this, but the further the rehosted environment differs from the real device, the greater the risk of false negatives or misleading test results.

So the presentation takeaway would be:

For Penguin, air-gap complexity is partly about staging software dependencies, but the larger challenge is firmware-specific rehosting. Even with complete internal mirrors, some firmware may still require hardware, drivers, or device behavior that cannot be reproduced accurately in QEMU.

That makes Penguin somewhat different from Ludus: Ludus’s air-gap problem is primarily dependency distribution; Penguin has that same problem plus the technical difficulty of accurately reproducing the target device.






For Ludus in an air-gapped network, the main challenge is that everything Ludus normally pulls from the internet has to be available internally instead.

I’d present it this way:

1. Proxmox, Debian, and Ludus dependencies — Ludus and its installer normally pull packages, tools, and supporting components from the internet. In an air gap, those dependencies have to be pre-staged and transferred in, or made available through internal mirrors.
2. VM templates and OS images — Ludus uses Packer to build VM templates from operating system ISOs and images. Those Windows, Linux, or other OS images therefore have to be obtained externally, approved, and transferred into the internal environment.
3. Configuration content and Ansible dependencies — Ludus uses Ansible to configure the VMs after they are created. Any required Ansible roles, collections, scripts, packages, or external dependencies also have to be mirrored or pre-staged internally.
4. Software inside the VMs — Even after the VM is created, the operating system and installed tools may normally reach the internet for updates, package repositories, installers, or security tooling. In an air-gapped environment, those resources also need internal mirrors or locally staged packages.

So the presentation takeaway would be:

For Ludus, the air-gap complexity is front-loaded in staging and mirroring all of the dependencies that would normally come from the internet. Once those dependencies are available internally, Ludus can still provide the same repeatable and automated deployment model.

The main challenge is therefore not Ludus itself, but making sure every dependency in the build and configuration chain is available inside the air-gapped environment.








