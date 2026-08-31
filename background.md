Ludus and Proxmox Overview and Deployment Requirements

1. Purpose and Scope

This page provides a high-level introduction to Ludus and the Proxmox Virtual Environment (Proxmox VE) platform that Ludus runs on.

The intent is to give stakeholders, infrastructure teams, developers, and security personnel a common understanding of:

* what Proxmox is
* what Ludus is
* how Ludus and Proxmox work together
* what Ludus can be used for
* the operating system and infrastructure requirements
* Ludus’s preferred bare-metal deployment
* the initial nested virtualization deployment planned for this environment
* the limitations of nested virtualization compared with bare metal
* what changes when Proxmox and Ludus are deployed in an air-gapped environment
* the expected complexity of standing up and maintaining the environment offline

This is an introductory page, not the detailed implementation runbook.

Separate Confluence pages should cover the actual installation commands, offline dependency manifests, Proxmox configuration, network design, Ludus installation, template builds, software import process, range configuration, updates, backup, and troubleshooting.

For this environment, there are two deployment models to keep in mind:

Ludus preferred / normal deployment

Dedicated system
→ Debian / Proxmox
→ Ludus
→ Range VMs

Current implementation

Existing virtualization platform
→ Debian / Proxmox / Ludus VM
→ Nested range VMs

Bare metal is not currently available, so nested virtualization is the first implementation. Bare metal should still be considered the preferred future architecture because Ludus explicitly recommends it. (Ludus)

⸻

2. What is Proxmox VE?

Proxmox VE is the virtualization platform underneath Ludus.

It provides the infrastructure needed to create and run virtual machines.

Proxmox includes capabilities such as:

* KVM/QEMU virtual machines
* Linux containers using LXC
* virtual networking and bridges
* VM storage
* snapshots and cloning
* VM templates
* backup and restore
* web-based administration
* command-line administration
* APIs for automation
* clustering and high-availability capabilities

For Ludus, the important part is KVM/QEMU. This is what actually runs the Windows and Linux virtual machines that make up a cyber range. (Proxmox)

Proxmox VE is based on Debian GNU/Linux.

The standard Proxmox ISO contains a complete Debian operating system along with the Proxmox kernel and packages. Proxmox recommends the ISO installation method for normal new installations. It is also possible to install Proxmox VE on top of an existing supported Debian system, but Proxmox considers that the more advanced installation path. (Proxmox VE)

For this project, Proxmox should be viewed as the virtualization foundation.

It provides the VMs, CPU, memory, storage, and virtual networking. Ludus then automates what Proxmox does.

⸻

3. What is Ludus?

Ludus is a platform for creating and managing cyber environments, which Ludus calls ranges.

A Ludus range can contain systems such as:

* Windows Server
* Windows workstations
* Linux servers
* Kali
* Active Directory domain controllers
* security tools
* analyst workstations
* intentionally vulnerable systems
* multiple isolated networks

Ludus is built on Proxmox. It is not a replacement for Proxmox.

Bad Sector Labs describes Ludus as an automation overlay on top of Proxmox. Administrators can still manually create or modify VMs and networks through Proxmox when needed. (Ludus)

Ludus mainly uses:

Packer to create reusable VM templates.

Ansible to configure machines after they have been created.

A simple example is:

Windows Server ISO
→ Packer builds a clean Windows template
→ Ludus tells Proxmox to clone the template
→ Ansible configures the new VM as a domain controller

The same model can be applied to several systems and networks at once.

The main advantage is repeatability. Instead of relying on a collection of manually configured VMs, the environment can be rebuilt from known templates, configuration, and automation.

Ludus is intended for cyber testing and development environments rather than production application hosting. Its own security guidance also recommends limiting access to trusted users because Ludus intentionally allows powerful automation and customization. (Ludus)

⸻

4. How Ludus and Proxmox Work Together

The responsibilities are separate but closely connected.

Proxmox provides the virtualization. Ludus turns that virtualization into a repeatable cyber range.

The basic architecture is:

Proxmox / Ludus Environment
        │
        ├── Debian
        ├── Proxmox VE
        ├── Ludus
        ├── Packer
        └── Ansible
                │
                ▼
           Ludus Range
           ├── Router
           ├── Windows Server
           ├── Windows workstation
           ├── Kali
           └── Linux systems

When a range is deployed, Ludus uses Proxmox to create the required virtual infrastructure and then runs its automation against those VMs.

Ludus can define a complex environment from a single configuration file while still allowing manual changes through Proxmox. (Ludus)

This is why Proxmox is a dependency for Ludus. Ludus does not provide its own hypervisor.

⸻

5. Ludus Features and Common Use Cases

Ludus is useful when environments need to be created, tested, changed, broken, reset, and rebuilt without manually rebuilding every machine.

Range deployment

A Ludus range can describe several VMs and networks as one environment.

The range configuration defines the systems that should exist and how they should be configured.

Templates

Templates are the starting point for the VMs.

Ludus normally builds them from operating-system ISOs using Packer. The built-in templates intentionally contain only the basic components required for remote management and automation rather than a large amount of preinstalled customization. (Ludus)

Ansible automation

Ansible configures systems after deployment.

It can be used for things such as:

* Active Directory configuration
* domain joins
* software installation
* security configuration
* monitoring agents
* development tooling
* security tooling
* custom applications

Ludus can use Ansible roles and collections from external sources or local directories. (Ludus)

Blueprints

Blueprints package reusable range designs.

A Blueprint can describe the range and identify the templates, roles, and collections required to build it.

Sources

Ludus Sources package related content together, including:

* Blueprints
* Packer templates
* Ansible roles
* Ansible collections

Sources are useful in an air-gapped environment because the required content can be approved and staged internally rather than retrieved from public sources at deployment time. (Ludus)

Typical use cases

Common uses include:

* Active Directory labs
* security research
* security tool evaluation
* adversary and defensive technique testing
* development and integration testing
* malware-analysis environments
* cyber exercises and training
* proof-of-concept environments
* testing changes against representative Windows and Linux systems

The main benefit is the ability to consistently recreate known environments.

⸻

6. Deployment Model: Preferred Bare Metal vs. Current Nested Implementation

Preferred / Normal Ludus Deployment: Bare Metal

Ludus explicitly labels Bare Metal as recommended and states that Ludus works best on a bare-metal system. (Ludus)

The preferred architecture is:

Dedicated System
       │
       ▼
Debian / Proxmox
       │
       ▼
     Ludus
       │
       ▼
   Range VMs

There is only one virtualization layer.

This gives Proxmox more direct access to the available CPU, memory, storage, and networking and avoids the additional overhead introduced by another hypervisor.

Bare-Metal Requirements

For a Ludus system, the Ludus requirements are more important than Proxmox’s much smaller evaluation minimums.

Requirement	Bare-metal Ludus requirement
Architecture	x86-64 / AMD64
Operating system	Debian 12/13 or Proxmox VE 8/9
CPU	PassMark score greater than 6,000
Virtualization	VMX or SVM available
RAM	At least 32 GB
Range RAM planning	Roughly 32 GB per deployed range
Initial storage	At least 200 GB
Additional storage	At least 50 GB per range
Preferred storage	Fast NVMe
Network	Standard wired interface
Administrative access	Root shell access
Standard Ludus installation	Internet access

Ludus states that systems below its published specifications may work but are not tested or supported. (Ludus)

For bare metal, Ludus recommends prioritizing RAM first, followed by fast CPU cores and then fast NVMe storage. (Ludus)

Why bare metal is preferred

Bare metal generally provides:

* less virtualization overhead
* more predictable CPU performance
* direct control over available RAM
* a simpler storage path
* simpler networking
* fewer infrastructure layers to troubleshoot
* greater flexibility for device passthrough and advanced virtualization
* more room for large ranges

That does not mean nested Ludus loses its normal features. It means bare metal gives those features a less constrained infrastructure to run on.

⸻

Current Implementation: Nested Virtualization

Bare metal is not currently available for this environment.

The first implementation will therefore be:

Existing Hypervisor
        │
        ▼
Proxmox / Ludus VM
        │
        ├── Debian
        ├── Proxmox VE
        └── Ludus
                │
                ▼
           Nested Range VMs

Ludus explicitly supports nested virtualization but notes that there is a performance penalty. (Ludus)

Proxmox can also operate as a guest when the outer virtualization platform supports nested virtualization. Proxmox documents this under its testing/evaluation guidance. (Proxmox)

Nested VM Requirements

Requirement	Current nested deployment
Architecture	x86-64 / AMD64
Operating system	Debian 12/13 or Proxmox VE 8/9
Nested virtualization	Required
VMX/SVM visible inside VM	Required
RAM	32 GB minimum
Storage	200 GB minimum
vCPU	Sized for the intended range
Virtual NIC	At least 1 standard vNIC
Root access	Required
Offline repositories/content	Required in this environment

For initial planning, a more practical VM allocation would be approximately:

* 8–12 vCPU
* 48–64 GB RAM
* 300–500 GB virtual disk
* one standard virtual NIC
* nested virtualization enabled

These larger values are planning targets, not Ludus-published minimums.

Nested virtualization requirement

The outer virtualization platform must expose virtualization extensions to the Proxmox/Ludus VM.

Inside Linux these normally appear as:

* vmx for Intel virtualization
* svm for AMD virtualization

Without this, the Debian VM itself may work, but Proxmox cannot properly provide the nested KVM VMs required by Ludus.

Nested virtualization limitations

The normal Ludus functions remain available when nesting works correctly.

There is no Ludus document that says features such as Blueprints, Ansible, Packer, Active Directory ranges, templates, or Sources are disabled simply because the platform is nested.

The limits come mainly from the infrastructure around Ludus.

CPU

A range VM receives vCPUs from Proxmox, while the Proxmox/Ludus VM itself receives vCPUs from the outer hypervisor.

That introduces another CPU scheduling layer.

Proxmox can also only expose CPU capabilities to its guests that it receives from the outer platform.

RAM

The range is limited by the RAM assigned to the Proxmox/Ludus VM.

If that VM receives 64 GB, then Debian, Proxmox, Ludus, the range router, and all range VMs have to fit inside that resource pool.

Large Windows-heavy ranges may hit that limit quickly.

Storage

The storage path also gains another layer:

Range VM disk
→ Proxmox storage
→ Proxmox/Ludus VM virtual disk
→ outer hypervisor storage

This can make template builds, VM cloning, operating-system installation, and large software deployments slower.

Networking

The network path becomes:

Range VM
→ Proxmox bridge
→ Proxmox/Ludus VM vNIC
→ outer virtual switch

Standard Ludus networking can still operate, but troubleshooting VLANs, MTU, filtering, routing, or bridge behavior may involve both Proxmox and the outer virtualization platform.

Advanced virtualization

A nested Proxmox system only sees the hardware and virtualization features that the outer hypervisor makes available.

Advanced PCIe passthrough or specialized device access may therefore be limited compared with bare metal. (Proxmox)

Range scale

This is probably the most important practical Ludus limitation.

Ludus can still deploy the same types of ranges, but the nested Proxmox/Ludus VM normally reaches its CPU, RAM, or storage ceiling sooner than a dedicated system.

For this reason, the initial nested deployment should first prove the platform using a smaller validation range before larger environments are introduced.

⸻

7. Operating System and Platform Requirements

The host operating system is a firm part of the platform requirement.

Ludus currently supports virtualization-capable:

* Debian 12
* Debian 13
* Proxmox VE 8
* Proxmox VE 9

Bad Sector Labs states that other host environments are not supported. (Ludus)

Why Debian

Proxmox VE is Debian-based.

Proxmox VE 8 is based on Debian 12, while Proxmox VE 9 is based on Debian 13. (Proxmox)

A Proxmox/Ludus system can therefore be created in two general ways:

Proxmox ISO

The ISO already contains the Debian base and Proxmox packages.

or

Clean Debian system

Start with supported Debian and install Proxmox/Ludus onto it.

Ludus specifically recommends starting from clean Debian 12/13 when building a new environment rather than installing into an existing Proxmox system with unknown customizations. (Ludus)

RHEL and Ubuntu

RHEL is not a supported host operating system for the standard Proxmox/Ludus stack.

Ubuntu is also not a supported Proxmox/Ludus host.

They can still be used as guest operating systems inside a Ludus range.

The requirement applies to the system hosting Proxmox and Ludus.

⸻

8. Resource, Storage, Network, and Configuration Requirements

The minimum system requirements should be treated as a starting point, not as the expected size of every environment.

Ludus’s broader capacity guidance includes:

* approximately 32 GB RAM per deployed range
* at least 200 GB for initial templates
* at least 50 GB additional storage per range
* fast NVMe storage where possible
* no more than 150 ranges on one Ludus instance

Actual requirements depend heavily on what is deployed. (Ludus)

A small Linux range and a multi-domain Windows Active Directory range will have very different requirements.

Networking

For the current nested deployment, the Proxmox/Ludus VM needs at least one normal virtual NIC.

Ludus does not require a second outer vNIC simply because it is installed in the VM.

Ludus creates additional networking inside Proxmox, including:

* WireGuard
* NAT networking
* Proxmox bridges
* range networks
* user/range routing

An existing Proxmox installation is modified fairly significantly by the Ludus installer. Ludus creates groups, pools, WireGuard networking, NAT interfaces, range interfaces, users, and system services. (Ludus)

This is one reason a dedicated clean Proxmox/Ludus system is preferred instead of installing Ludus on a heavily customized shared Proxmox node.

The default Ludus network ranges also need to be checked against existing organizational networks to avoid IP conflicts. (Ludus)

⸻

9. Air-Gapped Deployment: Requirements, Complexity, and Risks

The air gap is the biggest difference between this implementation and the normal Ludus deployment.

The standard Ludus installation explicitly requires Internet access. (Ludus)

There is currently no equivalent to a single official “Ludus offline installation bundle” containing every dependency needed to install Ludus and build arbitrary ranges.

Because of that, standing up Ludus in a completely air-gapped environment is more involved than installing it on an Internet-connected system.

It should be treated as a controlled offline software-integration effort rather than simply running the normal quick-start installer.

Proxmox itself is relatively straightforward to bring offline

The Proxmox ISO is a complete installation image containing Debian and the Proxmox VE packages needed for the base installation. (Proxmox VE)

The current Proxmox download site also publishes checksums and signatures so installation media can be verified before crossing the air-gap boundary. (Proxmox Download)

The harder part is package maintenance after installation.

Proxmox Offline Mirror

Proxmox provides an official tool called Proxmox Offline Mirror specifically for restricted and completely air-gapped systems.

It does not provide a prebuilt snapshot for download.

Instead, an Internet-connected staging system runs the Offline Mirror software and creates the snapshot.

The process is:

Internet-connected staging system
        │
        ▼
Proxmox Offline Mirror
        │
        ├── Downloads selected Debian repositories
        └── Downloads selected Proxmox repositories
                │
                ▼
       Creates point-in-time snapshot
                │
                ▼
    Copy snapshot to approved media
    or an approved network location
                │
                ▼
          Air-Gapped Network
                │
                ▼
      Offline Debian/Proxmox host

Proxmox defines a snapshot as a point-in-time view of the mirrored repository. The Offline Mirror tooling can then synchronize those snapshots onto external media for use by disconnected hosts. (Proxmox Offline Mirror)

This provides a supported way to handle Debian and Proxmox APT packages and updates.

It does not solve the complete Ludus offline problem.

⸻

Ludus is where most of the air-gap complexity appears

The normal Ludus installation uses an online installer and installs several supporting packages and Python modules.

When Ludus is added to Proxmox, its documented installation includes packages such as:

* Ansible
* Packer
* dnsmasq
* sshpass
* curl
* jq
* iptables-persistent
* supporting Python libraries including proxmoxer, pywinrm, netaddr, requests, dnspython, and jmespath

Ludus then creates its own services, groups, pools, bridges, NAT networking, WireGuard configuration, and range networking. (Ludus)

In an Internet-connected environment much of that is handled by the installer.

In the air gap, every required package needs to exist locally and the installer behavior needs to be tested against those offline repositories.

Template creation adds another dependency layer

Templates are the basis of every VM Ludus deploys.

Many Packer template definitions normally reference operating-system ISOs using Internet URLs.

Ludus documents an alternative where the ISO is pre-downloaded, uploaded to Proxmox storage, and the Packer template is changed from an iso_url to a local iso_file. (Ludus)

For the air gap, that means every template should be reviewed before deployment to identify:

* operating-system ISOs
* Packer resources
* installation scripts
* package repositories
* files downloaded during the template build

An ISO being available does not automatically mean the template can build completely offline.

A Linux installer, for example, may still expect access to an APT or RPM repository during installation.

Ansible roles and collections add more dependencies

Ludus supports Ansible roles and collections from Ansible Galaxy, URLs, or local content. (Ludus)

In an air-gapped system, anything normally obtained from Ansible Galaxy or another public repository has to be:

* downloaded beforehand
* version-pinned
* reviewed
* transferred
* installed from local content or an internal repository

The role itself may also download additional software when it runs.

That means reviewing only the Ansible role package is not enough. Its tasks need to be checked for additional Internet dependencies.

Ludus Sources need to be staged

Sources can contain Blueprints, templates, roles, and collections.

For the air gap, approved Source content should be mirrored or exported into an internal location so Ludus does not depend on public Git hosting.

Sources support locally packaged content, making them useful for this model. (Ludus)

Range VMs can have their own Internet dependencies

The final range introduces another dependency level.

For example, a range VM may expect to download:

* Debian or Ubuntu packages
* RPM packages
* Windows software
* browser installers
* Elastic components
* security tools
* development tools
* monitoring agents

Those dependencies need either an internal mirror or a local artifact.

The fact that Proxmox and Ludus themselves are installed successfully does not guarantee that a Blueprint or Ansible role will deploy successfully offline.

⸻

Why the air-gap setup is more complex

Compared with an Internet-connected deployment, the team needs to manage several additional concerns.

Dependency discovery

Every external dependency has to be found before the range is deployed.

Missing one package, ISO, Git repository, or software installer can cause a deployment to fail part way through.

Dependency versioning

The offline environment needs known compatible versions of:

* Proxmox
* Debian
* Ludus
* Packer
* Ansible
* Python libraries
* roles
* collections
* templates
* guest operating systems

Allowing these components to drift independently can create difficult troubleshooting problems.

Repository management

Debian and Proxmox repositories can be handled with Proxmox Offline Mirror.

Other ecosystems may require separate internal repositories or artifact storage.

Software transfer

The organization needs an approved process to move new software across the air-gap boundary.

That process should include:

* source validation
* version pinning
* malware scanning
* checksum/signature verification
* approval
* transfer
* internal storage
* audit trail

Updating

Updating an air-gapped environment is not simply running an Internet update.

New repository snapshots and software artifacts have to be prepared externally, validated, transferred, and tested before being introduced.

Troubleshooting

An online deployment can often resolve a missing package by downloading it.

An offline deployment cannot.

Failures caused by a missing dependency may require another software-transfer cycle before testing can continue.

Networking

Ludus normally expects Internet-connected networking and creates NAT, WireGuard, bridges, DNS/DHCP functions, and range routing.

In the air gap, these need to be reviewed so Ludus does not assume external connectivity that does not exist.

Internal DNS, NTP, package repositories, and management routes should also be defined before deployment.

Testing becomes important

The offline package set should be tested in a representative staging environment before being moved into the production air gap.

The goal is to prove that:

* Proxmox installs
* Proxmox updates from the offline mirror
* Ludus installs
* Ludus services start
* networking works
* templates build
* roles install
* the selected range deploys

before relying on that artifact set inside the isolated environment.

⸻

10. High-Level Stand-Up Process for the Air-Gapped Environment

The detailed commands should live on separate implementation pages, but the overall process should follow roughly this sequence.

1. Define and approve the architecture

Current:

Existing Hypervisor
→ Debian / Proxmox / Ludus VM
→ Nested range VMs

Future preferred option:

Bare Metal
→ Debian / Proxmox / Ludus
→ Range VMs

2. Provision the nested VM

For the current implementation, request sufficient:

* vCPU
* RAM
* virtual storage
* standard vNIC
* nested virtualization capability

Nested virtualization should be confirmed before doing the rest of the build.

3. Prepare the Internet-connected staging environment

This environment is used to gather and validate content before it enters the air gap.

It may be used for:

* downloading and verifying the Proxmox ISO
* creating Proxmox Offline Mirror snapshots
* collecting Ludus binaries
* collecting packages
* downloading operating-system ISOs
* collecting Ansible roles and collections
* collecting Sources and Blueprints
* collecting range-specific applications

4. Create the Proxmox Offline Mirror

Configure the required Debian and Proxmox repositories.

Synchronize them and create a known point-in-time snapshot.

Transfer that repository snapshot into the air gap using the approved process. (Proxmox Offline Mirror)

5. Install Debian / Proxmox

Install the supported Debian/Proxmox environment.

Inside the air gap, configure package management to use the approved offline repository rather than public Internet repositories.

6. Verify nested virtualization

Confirm that VMX or SVM is visible inside the VM.

This should be an acceptance criterion before moving forward.

7. Configure Proxmox

Configure:

* management addressing
* storage
* virtual networking
* DNS
* NTP
* administrative access
* offline APT repositories

8. Stage Ludus dependencies

Bring in the approved:

* Ludus release
* installation dependencies
* Packer
* Ansible content
* Python dependencies
* Sources
* templates
* ISOs
* required application installers

9. Install Ludus

Run the Ludus installation using the locally available dependencies.

The offline procedure needs to account for the fact that Ludus’s documented installer normally assumes Internet access. (Ludus)

10. Review Ludus configuration and networking

Validate:

* Proxmox storage pools
* Ludus bridges
* NAT network
* WireGuard
* range address space
* DNS/DHCP behavior
* routes
* potential conflicts with existing air-gap networks

11. Build only the initial required templates

For the first validation range, avoid importing every possible template.

Start with the minimum set needed for the proof of functionality.

Pre-downloaded ISOs can be placed in Proxmox storage and referenced locally by Packer. (Ludus)

12. Deploy a small validation range

A reasonable first range might contain:

* one router
* one Windows Server
* one Windows workstation
* one Kali or Linux system

This proves the complete path:

Nested virtualization
→ Proxmox
→ Ludus
→ networking
→ Packer
→ templates
→ Ansible
→ operational range

Only after this works consistently should larger Active Directory or specialized ranges be introduced.

⸻

11. Key Takeaways and Follow-On Documentation

The important stakeholder points are:

Proxmox is the virtualization platform.
It provides the VMs, storage, networking, and underlying management capabilities.

Ludus is the cyber-range automation layer.
It uses Proxmox, Packer, Ansible, templates, Blueprints, Sources, and range configuration to create repeatable cyber environments. (Ludus)

Bare metal is Ludus’s preferred and normal deployment model.
Bad Sector Labs explicitly labels bare metal as recommended. (Ludus)

The current implementation will use nested virtualization.
Bare metal is not currently available, so Proxmox and Ludus will initially run inside a VM.

Nested virtualization does not inherently remove normal Ludus features.
The primary limitations are performance, available resources, storage/network layering, and dependence on what the outer hypervisor exposes.

The nested VM needs hardware virtualization exposed to it.
VMX or SVM must be visible inside the VM for Proxmox/KVM to run the nested range machines. (Ludus)

Debian is part of the required platform.
Ludus supports Debian 12/13 or Proxmox VE 8/9. Proxmox itself is Debian-based. RHEL and Ubuntu are not supported as the standard Proxmox/Ludus host. (Ludus)

Air-gapped Proxmox is manageable through an official Proxmox mechanism.
Proxmox Offline Mirror can create point-in-time Debian/Proxmox repository snapshots on an Internet-connected staging system and transfer them to disconnected systems. (Proxmox Offline Mirror)

Air-gapped Ludus is more complicated.
Ludus’s normal installation expects Internet access, and there is no single documented offline Ludus bundle containing every possible dependency. The required software has to be identified, staged, validated, and maintained internally. (Ludus)

Standing up the base platform is only part of the problem.
Templates, Ansible roles, Sources, Blueprints, and the software installed inside the range can introduce their own Internet dependencies.

For this reason, the initial air-gapped deployment should start with a small, known range and a tightly controlled dependency set.

Recommended follow-on Confluence pages

1. Nested Virtualization VM Requirements and Validation
2. Air-Gapped Proxmox Installation
3. Proxmox Offline Mirror Architecture and Operations
4. Air-Gapped Ludus Installation
5. Ludus Offline Dependency Manifest
6. Proxmox and Ludus Network Design
7. ISO and Template Management
8. Ludus Sources, Blueprints, Roles, and Collections
9. Air-Gap Software Import and Approval Process
10. Range Deployment and Validation
11. Updating Proxmox and Ludus Offline
12. Backup, Recovery, Operations, and Troubleshooting

Official References

Ludus Introduction

Ludus Installation Requirements

Ludus Bare Metal Deployment

Ludus on Existing Proxmox

Ludus Templates

Proxmox VE Requirements

Proxmox VE Administration Guide

Proxmox Offline Mirror Documentation