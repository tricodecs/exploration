Think of them as four layers with different jobs:

* QEMU = the virtual hardware. It pretends to be the ARM, MIPS, x86, etc. computer that the firmware expects.
* PANDA = the observation layer. It is built on QEMU and adds tools to watch what the firmware is doing while it runs—execution, memory, system behavior, and so on.
* IGLOO = the helper inside the guest Linux system. It runs as a kernel module and gives Penguin deeper access to processes, memory, kernel functions, devices, and other guest activity.
* Penguin = the main rehosting controller. It takes the firmware filesystem, configures the environment, starts the PANDA/QEMU system, works with IGLOO, and helps make the firmware’s applications and network services runnable.

A simple way to remember it:

QEMU = runs it
PANDA = watches it
IGLOO = looks inside it
Penguin = puts the whole rehosting process together

For vulnerability validation:

Firmware → Penguin → PANDA/QEMU → Linux + IGLOO → firmware service runs → vulnerability test/PoC is executed → results are observed.

One important detail: Penguin already uses its own PANDA/QEMU-based environment, so you do not necessarily install all four independently.




QEMU and an LLM are not enough by themselves to validate a list of vulnerabilities because they solve only part of the problem.

* QEMU only runs or emulates the target system. It gives you a place where the software or firmware can execute.
* An LLM can reason about the vulnerability and decide what to try, but it does not automatically have a reliable way to prove that a vulnerability is real.
* Each vulnerability may require a different test. One may need a specific network request, another a malformed file, another a memory trigger, and another a login bypass.
* You need a real validation method such as a PoC, Nuclei template, Metasploit module, vendor test, or custom test.
* You also need evidence. A test result should show something concrete: a crash, unauthorized access, code execution, changed system state, specific logs, or other measurable behavior.
* For firmware, there is an extra problem: the firmware may not run correctly in QEMU without the right CPU architecture, kernel, devices, networking, and rehosting setup.

So the practical model is:

QEMU = runs the target

LLM = decides what to test and helps coordinate

PoC / test tool = actually tests the vulnerability

Logs / PANDA / IGLOO / debugger = provides evidence

Validation result = based on that evidence

The main issue is that QEMU + LLM gives you an environment and a decision-maker, but not the actual test logic and proof needed to confirm or reject each vulnerability.



The main difficulty is that firmware rehosting is not always “one setup works for every firmware.”

The key reason is IGLOO runs as a Linux kernel module inside the rehosted guest, and that module must be built for the exact guest kernel build it loads into. The kernel version, configuration, CPU architecture, compiler, and kernel symbols can all affect compatibility.

In plain terms:

Firmware A
→ ARM
→ guest kernel build A
→ needs matching IGLOO A

Firmware B
→ MIPS
→ guest kernel build B
→ needs matching IGLOO B

You cannot assume one igloo.ko will work for both.

There is one important correction to the way you phrased it: the IGLOO module does not necessarily have to match the original kernel that came with the physical firmware device. It must match the guest kernel that Penguin actually uses to rehost that firmware. Penguin starts from the firmware root filesystem and creates a target-specific rehosting configuration around it.

The other firmware-specific difficulty is that different firmware expects different hardware and system resources. One firmware may expect flash storage, NVRAM, special /dev devices, network interfaces, or vendor kernel drivers. Another may expect completely different resources. Penguin and IGLOO can model many of these, but the configuration may need adjustment for each target.

For an internal network with no Internet, this becomes harder because you need to have the required pieces already available internally:

correct architecture + suitable guest kernel + matching IGLOO build + matching compiler/toolchain + Penguin/PANDA/QEMU dependencies.

If one is missing, the environment cannot simply download it.

So the core challenge is:

You can standardize the platform, but the rehosting environment may still need to be tailored to the firmware being tested.

That is why having an internal collection of kernels, matching IGLOO modules, cross-compilers, and Penguin dependencies is important if you want to test many different firmware images.



Yes. That is another major difficulty.

The stack has many dependency layers, and they must line up correctly:

Firmware
→ CPU architecture
→ suitable guest Linux kernel
→ matching IGLOO module
→ matching compiler/toolchain
→ PANDA/QEMU build
→ Penguin
→ Python libraries, Nix packages, runtime libraries, guest tools

The problem is that changing one layer can affect the others. For example, a different kernel may require a different IGLOO build and possibly a different compiler/toolchain.

In an Internet-connected environment, missing dependencies can often be downloaded when needed. In an internal network with no external access, every required dependency must already be imported, approved, and available internally.

So the main difficulties are:

* Firmware-specific setup: different firmware may need different CPU architecture, kernel, hardware models, and configuration.
* Exact kernel/IGLOO pairing: IGLOO must match the guest kernel build it loads into.
* Many dependency layers: kernels, compilers, libraries, PANDA/QEMU, Penguin, Python packages, and guest tools all depend on one another.
* Offline availability: if even one required package or source file is missing, the build or rehosting process can stop.
* Version management: updating one component can require rebuilding or revalidating several dependent components.

In simple terms: the hard part is not just installing four tools. It is keeping a large, interconnected software stack compatible and fully available inside the isolated network.



# QEMU, PANDA, IGLOO and Penguin for Firmware Vulnerability Validation on an Internal Network

**Purpose:** Explain what QEMU, PANDA, IGLOO and Penguin do, how they work together to rehost firmware for vulnerability validation, what must be brought into a network with no external Internet access, the four practical offline deployment options, and the main setup difficulties.

**Accuracy review:** Checked against the current official QEMU, PANDA, Penguin, IGLOO, `linux_builder`, `kernelsmith`, and Nix documentation on September 27, 2026.

---

## 1. Executive summary

For this use case, **Penguin is the main firmware-rehosting tool**. It brings together a Penguin-specific **PANDA/QEMU** build, supported guest Linux kernels, the **IGLOO** kernel driver, guest utilities, Python packages and firmware-extraction dependencies.

The simplest relationship is:

```text
Firmware image
      |
      v
   Penguin
      |
      v
PANDA / QEMU
      |
      v
Guest Linux kernel
      |
      +---- matching IGLOO module (`igloo.ko`)
      |
      v
Firmware filesystem, applications and services
      |
      v
Vulnerability-specific test / PoC
      |
      v
Runtime evidence
```

The four technologies create and observe the test environment. **They do not automatically prove that any arbitrary vulnerability exists.** A vulnerability-specific test is still needed, such as a PoC, Nuclei template, Metasploit module, vendor test or custom validation procedure.

For an isolated network, the four main deployment choices are:

| Option | Summary | Complexity | Best fit |
|---|---|---:|---|
| **1. Prebuilt Penguin image** | Transfer an approved Docker/OCI Penguin image into the internal network. | Low | Prototype / fastest start |
| **2. Nix closures** | Transfer Penguin and all of its exact Nix dependencies. | Medium | Controlled internal builds |
| **3. Internal Nix binary cache** | Maintain an internal repository of approved prebuilt Nix artifacts. | Medium-High | Long-term shared platform |
| **4. Full source/build mirror** | Mirror source repositories plus every external build input. | High | Maximum offline independence |

---

# 2. What the four components are

| Component | Easy description | Role in validation |
|---|---|---|
| **QEMU** | An open-source machine emulator and virtualizer. It can emulate CPUs, memory and devices for architectures such as ARM, MIPS and x86. | Provides the virtual machine/emulation layer in which firmware software can run. |
| **PANDA** | A dynamic-analysis platform built on QEMU. It adds instrumentation, plugins and analysis features. | Provides deeper visibility into what the guest is doing while a test runs. |
| **IGLOO** | A Linux kernel module (`igloo.ko`) that runs inside the guest used by Penguin. | Gives Penguin controlled access to guest kernel/user memory, processes, hooks, files/devices and other guest state. |
| **Penguin** | The main firmware-rehosting framework. | Extracts/uses the firmware filesystem, creates a rehosting configuration, starts the guest environment, exposes services and collects analysis output. |

## QEMU

QEMU provides a virtual model of a machine: CPU, memory and emulated devices.

This matters because embedded firmware may be built for an architecture different from the host running the validation platform. For example, an x86-64 validation server may need to run ARM or MIPS firmware.

QEMU by itself does **not** validate a reported vulnerability. It supplies the execution environment.

## PANDA

PANDA is built on QEMU and adds dynamic analysis.

Useful capabilities include:

- whole-system instrumentation
- plugin-based analysis
- operating-system introspection
- memory and execution analysis
- record/replay on supported guest architectures
- optional LLVM-based analysis such as dynamic taint analysis
- a Python interface called PyPANDA

Standalone PANDA currently publishes normal and development Docker images. LLVM 14 is required for PANDA's LLVM-dependent functionality such as its taint system.

## IGLOO

IGLOO is the guest-side kernel component used by Penguin.

The current `igloo_driver` can support functions such as:

- reading and writing guest kernel and user memory
- inspecting processes, mappings, arguments, file descriptors and registers
- installing kernel, user-space and syscall hooks
- resolving and calling kernel functions
- synthesizing or modeling `/proc`, `/sys`, `/dev`, sockets and MTD flash behavior

IGLOO is controlled from the host side by Penguin.

## Penguin

Penguin is the main rehosting layer.

Its documented workflow is:

1. Obtain an archive of the firmware root filesystem.
2. Generate an initial Penguin configuration using static analysis.
3. Run the rehosting environment.
4. Observe what works or fails.
5. Adjust the configuration when necessary.

During a run, Penguin can provide:

- a root shell inside the guest
- access to guest network services
- dynamic-analysis output
- console logs
- configuration options for addressing missing hardware assumptions or startup problems

### Important packaging point

For the normal Penguin workflow, you generally **do not need to install standalone upstream QEMU and standalone PANDA separately**.

The current Penguin Nix build already includes a pinned **Penguin PANDA/QEMU fork** and an `/igloo_static` tree containing kernels, the IGLOO driver and guest utilities.

Standalone QEMU or PANDA is only needed if you also want to use those tools independently of Penguin.

---

# 3. What you need to obtain

## Simplest Penguin-based deployment

For a basic internal firmware-rehosting environment, the main package is:

**Penguin container/image + Penguin wrapper**

The current Penguin Nix build pins and packages the important runtime pieces, including:

- Penguin Python environment
- Penguin application
- `pengutils`
- `pyplugins`
- Penguin PANDA/QEMU fork
- supported guest kernels
- IGLOO driver
- guest utilities
- architecture-specific native helpers
- musl headers/sysroots used by guest helpers
- vhost/vsock support
- `fw2tar` firmware-extraction dependency closure
- required runtime libraries

This is why a prebuilt Penguin image is normally the easiest way to start on an isolated network.

## QEMU â only if used independently

Bring:

- approved QEMU binaries or source
- the required `qemu-system-*` targets
- `qemu-img`
- required runtime libraries
- build dependencies if compiling QEMU internally

Common QEMU runtime dependencies can include GLib, Pixman, libfdt and networking libraries such as libslirp.

## PANDA â only if used independently

Bring:

- PANDA runtime image/binaries
- required PANDA plugins
- PyPANDA/Python environment if Python-based analysis is required
- build dependencies if compiling internally
- LLVM 14 if using LLVM-dependent PANDA features such as taint analysis

PANDA provides official `pandare/panda` and `pandare/pandadev` Docker images.

## IGLOO

If you need to build or modify IGLOO internally, bring:

- `rehosting/igloo_driver`
- Nix with flakes enabled
- matching kernel derivations from `linux_builder`
- matching cross-compilers/toolchains
- prebuilt `igloo.ko` modules where appropriate
- `kernelsmith` when another/custom kernel version is required

The current recommended IGLOO build path is Nix. The older Docker-based build path has been removed from the current project.

## Penguin

Bring:

- approved Penguin Docker/OCI image or portable image
- `penguin` host-side wrapper
- firmware images/root-filesystem archives to be tested
- vulnerability-specific test artifacts
- Nix environment only if you intend to rebuild or modify the stack internally

---

# 4. End-to-end firmware vulnerability-validation process

```text
1. Obtain the firmware image
        |
2. Extract the firmware root filesystem
        |
3. Penguin performs static analysis
        |
4. Penguin creates a rehosting configuration
        |
5. Penguin starts its PANDA/QEMU environment
        |
6. A supported guest Linux kernel boots
        |
7. The matching IGLOO module loads/runs in that guest
        |
8. Firmware applications and services start
        |
9. Connect to the target service or process
        |
10. Run the vulnerability-specific test
        |
11. Collect service, process, log and dynamic-analysis evidence
        |
12. Determine whether the reported condition is reproduced
```

## Firmware extraction

Penguin expects a tar archive of the firmware root filesystem with permissions and ownership preserved. Its documentation recommends `fw2tar` for creating this from a firmware image.

## Rehosting configuration

Penguin performs static analysis of the extracted filesystem and creates a project configuration describing how the firmware should be rehosted.

## Running the firmware

Penguin boots the guest environment and uses the firmware filesystem in that environment.

The analyst can then:

- open a guest root shell
- inspect running processes
- access reachable guest services
- review console and plugin output
- refine the rehosting configuration when the firmware assumes unavailable hardware or files/devices

## Running the actual vulnerability test

Rehosting creates the target environment, but a separate test is required to validate a reported vulnerability.

Possible test sources include:

- published PoC
- Nuclei template
- Metasploit module
- vendor validation procedure
- custom test
- protocol-specific client/test

Useful evidence can include:

- expected or unexpected service response
- process crash
- controlled code path reached
- memory behavior
- system-call activity
- console/application logs
- PANDA/Penguin/IGLOO dynamic-analysis output

If many reported vulnerabilities will be processed automatically, an **orchestration layer** becomes useful for selecting the appropriate test, running it, collecting the evidence and classifying the result. Penguin itself is primarily the rehosting and analysis environment, not a complete vulnerability-test orchestrator.

---

# 5. The Linux kernel and IGLOO relationship

`igloo.ko` is a Linux kernel module.

The current IGLOO project is explicit: **the module must be built against the exact guest kernel build it will be loaded into.**

This is more specific than matching only a version number such as "6.13".

Compatibility can depend on:

- kernel source/version
- kernel `.config`
- CPU architecture
- compiler/toolchain
- exported kernel symbols
- module-version CRCs / `Module.symvers`

The toolchain matters because some kernel configuration choices and ABI details depend on compiler capabilities. Therefore, even identical kernel source and configuration can produce an incompatible module if the relevant build environment changes.

## Which kernel has to match?

IGLOO must match the **guest kernel that Penguin actually boots**.

That kernel does **not necessarily have to be the original kernel shipped inside the physical device**.

Penguin primarily rehosts the firmware's root filesystem using a supported guest kernel. That is an important part of how it can run firmware applications without reproducing every detail of the original physical hardware.

## Current standard `linux_builder` kernels

As of September 27, 2026, the official `rehosting/linux_builder` repository advertises two standard IGLOO kernel versions:

- Linux **4.10**
- Linux **6.13**

across up to eleven targets / 19 buildable kernel-target combinations.

Each build can provide the boot image, `vmlinux`, a minimal kernel development tree and analysis artifacts Penguin needs.

## Other/custom kernel versions

For other kernels, the project provides **kernelsmith**.

`kernelsmith` is a Nix-based Linux kernel cross-compiler designed to cover approximately Linux 2.6 through current 6.x kernels across multiple architectures, automatically selecting a suitable toolchain family for the kernel generation and target architecture.

The relationship is:

```text
Standard Penguin-supported kernel
        |
   linux_builder
        |
 matching IGLOO build
```

or, for another kernel:

```text
Required kernel version
        |
    kernelsmith
        |
 build exact guest kernel
        |
 build IGLOO against that exact kernel derivation
```

### Common embedded kernel families vs. Penguin's built-in matrix

Embedded devices in the field can use many kernel generations, including older 3.x/4.x and newer 5.x/6.x families. Examples frequently seen in embedded Linux ecosystems include 3.18, 4.4, 4.9, 4.14, 5.4, 5.10, 5.15, 6.6 and 6.12.

This should **not** be confused with Penguin's current standard `linux_builder` matrix. The fact that a kernel family is common in embedded devices does not mean Penguin currently ships it as a standard prebuilt guest kernel.

---

# 6. What Nix is

**Nix** is a package manager and reproducible build system.

In simple terms, it records and builds the exact versions of software and dependencies needed for an environment instead of depending on whatever happens to be installed on the host.

For this stack, Nix can pin/manage:

```text
Penguin
  |
  +-- PANDA/QEMU fork
  +-- Python environment
  +-- Linux kernels
  +-- IGLOO
  +-- cross-compilers/toolchains
  +-- guest utilities
  +-- firmware-extraction tools
  +-- runtime libraries
```

The current Penguin image is built from a Nix flake. Its dependencies are pinned by `flake.lock`, including its PANDA/QEMU fork, IGLOO/static tree and firmware-extraction stack.

This is especially useful for isolated networks because the approved software set can be transferred as a known group rather than downloaded independently by each internal machine.

---

# 7. What an internal Nix binary cache is

An **internal Nix binary cache** is a private repository of already-built Nix store objects.

It serves a role similar to an internal software/package repository.

Without one:

```text
Internal build machine
      |
      X  No Internet access
```

With one:

```text
External staging/build environment
      |
 build / download / scan / approve
      |
 controlled transfer
      |
      v
Internal Nix binary cache
      |
      +--> Penguin systems
      +--> IGLOO builds
      +--> kernel builds
      +--> development VMs
```

The cache can hold approved copies of:

- Penguin
- Penguin PANDA/QEMU build
- Linux kernels
- IGLOO modules
- `linux_builder`
- `kernelsmith` outputs/toolchains
- Python dependencies
- libraries
- build tools

Nix supports both local file-based caches and caches served over HTTP/HTTPS. Internal caches can also be signed so clients can verify that the artifacts came from the approved cache.

---

# 8. Four deployment options for a network with no external Internet access

## Option 1 â Transfer a prebuilt Penguin container image

### How it works

```text
Official Penguin source/image
        |
 Internet-connected staging system
        |
 download or build
        |
 scan / verify / approve
        |
 export Docker/OCI image
        |
 controlled transfer
        |
        v
Internal registry or image archive
        |
        v
Penguin runtime
```

Penguin officially consists of a `rehosting/penguin` container and a host-side `penguin` wrapper.

The current project builds the container reproducibly from a Nix flake. The flake exposes normal Docker image outputs as well as a `portableImage` variant.

The image contains or pulls together the important runtime environment, including:

- Penguin Python environment
- Penguin PANDA/QEMU fork
- `/igloo_static` with kernels, driver and guest utilities
- firmware-extraction stack
- runtime libraries and guest helper components

### Is this a Docker image?

Yes. It is a Docker/OCI-style container image that can be:

- loaded directly into Docker/Podman
- exported as an image tar archive
- stored in an internal container registry

The Nix build exposes outputs such as `dockerImage`, streaming image variants and `portableImage`.

The portable image relocates Penguin's Nix closure away from `/nix/store`; it is mainly useful on hosts where something else owns or mounts `/nix`. The normal image is preferred otherwise.

### Existing images

The project documentation points to the official `rehosting/penguin` image, and current GitHub releases publish release assets including portable images.

As of September 27, 2026, the GitHub release page lists **v3.1.20** as the latest release.

### Safe sourcing

No public container should automatically be considered safe simply because it is public or official.

A practical approval process is:

1. Obtain it from the official Penguin project/release or build it from the official source.
2. Pin an exact release/tag or digest rather than depending on `latest`.
3. Record and verify the digest/hash.
4. Run required vulnerability/malware scans.
5. Retain an SBOM when required by policy.
6. Approve/sign the artifact internally if your process supports that.
7. Promote only the approved image into the internal registry.

### Advantages

- lowest setup complexity
- fastest route to an initial environment
- most Penguin dependencies are already packaged together
- avoids separately assembling QEMU, PANDA, IGLOO and their dependency trees

### Limitations

- provides the kernel/IGLOO combinations included in that approved build
- a custom kernel or modified low-level dependency may require a new external build/import cycle
- prebuilt runtime alone is not enough if you want full internal source rebuilding

### Best fit

- prototype
- initial evaluation
- small number of systems
- standard Penguin-supported kernel combinations

---

## Option 2 â Transfer complete Nix closures

### What a Nix closure means

A closure is the requested Nix package **plus every Nix store object it depends on**.

Example:

```text
Penguin
 + PANDA/QEMU
 + kernels
 + IGLOO
 + compilers
 + Python packages
 + runtime libraries
 + support utilities
        |
        v
 Complete Nix closure
        |
 controlled transfer
        |
        v
 Internal Nix store
```

Nix provides commands for copying complete store-path closures between stores or machines.

### Advantages

- more flexible than transferring only one container image
- exact dependency versions move together
- reproducible build/runtime environment
- useful when individual components need to be inspected or rebuilt internally

### Limitations

- requires more Nix knowledge
- the complete closure must be moved, not only the top-level package
- closures can be large
- a new dependency still requires another approved import if it is not already present internally

### Best fit

- controlled internal development
- teams that need some build capability but do not yet want to operate an internal cache service
- environments that prefer controlled file-based transfer

---

## Option 3 â Maintain an internal Nix binary cache

### How it works

```text
External staging/build environment
        |
 build / download
 scan / test / approve
        |
 controlled transfer
        |
        v
Internal Nix binary cache
        |
        +--> Penguin systems
        +--> IGLOO builds
        +--> linux_builder
        +--> kernelsmith
        +--> development/build VMs
```

Internal systems use this cache instead of reaching external Nix/Cachix sources.

### What can be stored

- Penguin builds
- PANDA/QEMU build used by Penguin
- Linux kernels
- IGLOO builds
- cross-compilers/toolchains
- Nix packages
- Python environments
- libraries
- build utilities

### Advantages

- high flexibility
- one approved internal source for many systems
- previously built kernel/toolchain combinations can be reused
- reduces repeated rebuilding and repeated transfers
- supports repeatable internal development
- easier version control and artifact governance

### Limitations

- requires an internal cache service and storage
- requires lifecycle management
- requires a process for importing, signing and retiring artifacts
- new upstream dependencies still require an approved staging/import process

### Best fit

- maintained internal vulnerability-validation platform
- multiple developers or build systems
- repeated Penguin/IGLOO/kernel work
- long-term isolated environment

---

## Option 4 â Full internal source and build mirror

### How it works

Mirror both the source repositories **and every external input needed to rebuild them**.

Possible source repositories include:

- Penguin
- `igloo_driver`
- `linux_builder`
- `kernelsmith`
- Penguin's QEMU/PANDA fork
- standalone PANDA, if you need independent PANDA use
- upstream QEMU, if you need independent QEMU use

Depending on the versions being built, external source/build inputs can include:

- Linux kernel source archives
- GCC
- binutils
- musl
- LLVM/Clang
- Python packages
- Nixpkgs and other flake inputs
- runtime libraries
- firmware-extraction dependencies
- guest helper utilities
- other source archives referenced by the Nix builds

### Important point

**Cloning the Git repositories alone is not enough.**

A repository may still reference:

- another Git repository
- another Nix flake
- a kernel.org tarball
- a compiler/toolchain archive
- a package registry
- a binary cache

All required external inputs must already exist internally or be pre-populated into the internal Nix store/cache.

### Advantages

- maximum external-Internet independence
- highest source/build control
- supports custom kernels and deeper development
- suitable when rebuilding from source internally is a requirement

### Limitations

- highest setup complexity
- largest storage requirement
- requires dependency inventory and update management
- may require internal Git, package, artifact and Nix services
- requires ongoing maintenance as upstream dependencies change
- the best way to prove completeness is to perform a clean/cold build without external network access

### Best fit

- long-lived disconnected environments
- full internal development
- teams modifying Penguin, PANDA/QEMU, IGLOO or kernels
- environments requiring source-level rebuild capability

---

# 9. Comparison of the four options

| Option | Setup complexity | Internal flexibility | Offline independence | Typical use |
|---|---:|---:|---:|---|
| **1. Prebuilt Penguin image** | Low | Low-Medium | High for the packaged runtime | Initial deployment / prototype |
| **2. Nix closures** | Medium | Medium-High | High for transferred packages | Controlled internal builds |
| **3. Internal Nix cache** | Medium-High | High | Very high for cached packages | Maintained shared platform |
| **4. Full source/build mirror** | High | Highest | Highest | Full offline development/rebuild |

A practical rollout is:

```text
Initial proof of concept:
Option 1

Long-term shared environment:
Option 3

Need full source rebuild/custom-kernel independence:
Add Option 4
```

Option 2 is a useful middle ground when you want reproducible Nix artifacts but do not yet want to operate a permanent cache service.

---

# 10. Main difficulties on an isolated internal network

## 10.1 The dependency chain is larger than four repositories

A realistic dependency chain looks like:

```text
Penguin
  |
  +--> Python packages
  +--> PANDA/QEMU fork
  |       +--> glibc / Pixman / libfdt / GLib / slirp / other runtime deps
  |
  +--> Linux guest kernels
  |       +--> architecture-specific toolchains
  |
  +--> IGLOO
  +--> guest helper programs
  +--> firmware-extraction tools
  +--> Nix / Nixpkgs / flake inputs
```

If a required dependency was not transferred, a clean internal build can fail because there is nowhere external to retrieve it from.

## 10.2 External binary caches normally make the build easier

Penguin's current Nix configuration declares the `rehosting-tools.cachix.org` substituter.

The official documentation notes that a cold build with no cache hits is expensive because it may need to build/cross-compile many guest helpers and pull/build the QEMU closure and toolchains.

An isolated network cannot use that external cache directly, which is why complete Nix closures or an internal Nix cache are valuable.

## 10.3 Kernel and IGLOO builds must stay paired

IGLOO cannot be built once and assumed to work with every guest kernel.

The kernel, configuration, target architecture and compiler/toolchain must stay aligned with the IGLOO module.

The current Nix design helps by building IGLOO directly against a kernel derivation and checking module-version CRCs against that kernel's `Module.symvers`.

## 10.4 Firmware architectures vary

Firmware may target ARM, ARM64, MIPS little/big endian, PowerPC, x86 and other architectures.

The correct emulator target, guest kernel and cross-toolchain must exist for the intended target.

Current Penguin Nix documentation lists 12 bootable system targets across ARM, AArch64, MIPS variants, PowerPC variants, RISC-V, LoongArch and x86-64.

## 10.5 Firmware rehosting is not always automatic

Firmware applications may expect physical-device resources such as:

- flash/MTD devices
- NVRAM
- GPIO
- custom `/proc` or `/sys` entries
- proprietary drivers
- special sockets or network interfaces
- device-specific startup scripts

Penguin/IGLOO can model or synthesize some missing resources, but the rehosting configuration may still require iteration.

Extracting a filesystem successfully does **not** guarantee every firmware service will immediately start.

## 10.6 Emulation and instrumentation have performance cost

QEMU software emulation is slower than native execution, especially across CPU architectures.

PANDA instrumentation can add more overhead because it is observing and analyzing execution.

KVM acceleration can help when the host and guest architecture/setup are compatible, but cross-architecture firmware rehosting generally depends on QEMU's software emulation rather than assuming KVM acceleration.

## 10.7 Container and host requirements still matter

The internal host needs an approved container runtime such as Docker or Podman and sufficient:

- CPU
- RAM
- disk space
- networking
- runtime permissions/device access

Penguin's current Nix development checks include the Nix setup, cache reachability, container engine, disk headroom and `/dev/kvm` availability.

## 10.8 Imported software needs supply-chain controls

For an internal security environment, imported artifacts should normally be tracked with information such as:

- source
- exact version
- hash/digest
- license information
- vulnerability scan status
- malware scan status where required
- SBOM where required
- signature/approval status
- update/retirement process

This applies to container images, Nix closures, source archives and testing artifacts.

## 10.9 Updates require a deliberate import process

With no external Internet access, a new:

- Penguin release
- PANDA/QEMU build
- IGLOO version
- guest kernel
- compiler/toolchain
- Python dependency
- vulnerability fix

must go through the staging, scanning, approval and transfer process.

## 10.10 The vulnerability test itself must also be available internally

Rehosting only creates the test environment.

The specific PoC/test/template/module needed to validate a reported vulnerability must also be imported or developed internally.

---

# 11. Recommended internal architecture

For a maintained environment:

```text
                EXTERNAL / STAGING NETWORK
                --------------------------
Official repositories and release sources
Nix/Cachix/upstream source inputs
             |
      build / download
             |
       scan / verify
             |
     record digest / SBOM
             |
          approve
             |
     controlled transfer
             |
             v

                   INTERNAL NETWORK
                   ----------------

     Internal container/artifact registry
                  +
        Internal Nix binary cache
                  +
        Internal Git/source mirror
           (only when required)
                  |
                  v
         Penguin build/run VM
                  |
          PANDA/QEMU environment
                  |
           supported guest kernel
                  |
          matching IGLOO module
                  |
             firmware services
                  |
        vulnerability-specific test
                  |
                  v
             evidence/results
```

## Suggested rollout

### Initial deployment

Use **Option 1**:

> Approved official Penguin container/portable image + Penguin wrapper + firmware + required validation test artifacts.

This is the least complex way to prove the workflow inside the internal environment.

### Permanent shared platform

Move toward **Option 3**:

> Internal Nix binary cache + internal container/artifact registry.

This allows systems to reuse approved Penguin builds, kernels, IGLOO modules, compilers and dependencies without Internet access.

### Full internal development / custom kernel support

Add **Option 4** when needed:

> Full source/dependency mirror, including all Nix flake inputs and external source archives required for clean builds.

---

# 12. Key points to remember

- **QEMU** provides the emulated machine.
- **PANDA** adds deep dynamic analysis on top of QEMU.
- **IGLOO** runs inside the guest kernel and gives Penguin guest introspection/control capabilities.
- **Penguin** is the main firmware-rehosting workflow.
- Normal Penguin use already includes a pinned **PANDA/QEMU fork**, so standalone QEMU and PANDA are not mandatory.
- The **IGLOO module must match the exact guest kernel build** it was built against.
- Current `linux_builder` standard kernels are **4.10 and 6.13**; `kernelsmith` provides a path for other/custom kernel versions.
- The kernel Penguin boots does **not necessarily have to be the original device kernel**.
- Rehosting alone does **not** validate a vulnerability; a vulnerability-specific test is still required.
- **Option 1** is the easiest way to start in an isolated network.
- **Option 3** is the strongest practical model for a maintained shared internal platform.
- **Option 4** is appropriate when full source-level rebuild independence is required.
- Git repositories alone are not enough for a full offline build; every external source/build input must also be available internally.
