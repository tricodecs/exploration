Promox two-server architecture — Shared air-gapped network

Recommendation: Allow both servers to connect directly to the wider air-gapped network, but separate management traffic, dependency downloads, and vulnerability-testing traffic.

This design is simpler for managing offline dependencies than the previous architecture because Server 2 can obtain software directly from approved internal repositories without transferring everything through Server 1.

The main security consideration is ensuring that vulnerable test VMs cannot freely access the wider network.

1. Recommended network architecture

Wider air-gapped network

Internal package mirrors • ISO repository • Vulnerability feeds • DNS / NTP • Security services

Firewalls + network segmentation

Server 1

Orchestration

Central controller

Test workers

Database / evidence

Ludus client

Server 2

Proxmox + Ludus

Hypervisor / Ludus

VM templates

Virtual switches

Isolated test VMs

Server 1 remotely controls Server 2 through WireGuard / Ludus API

Controlled VM dependency access

Test VMs → Ludus range router → Firewall / approved proxy → Internal repositories only

All systems remain air-gapped from the internet, but Server 1, Server 2, and approved VMs can reach selected services within the organization’s internal network.

2. Server 1 — Central orchestration

Component	Purpose
Debian Linux	Host operating system
Orchestration application	Manages vulnerability-validation workflows
Ludus CLI / API client	Remotely controls Ludus on Server 2
WireGuard client	Secure access to Ludus and test VMs
Job scheduler / queue	Organizes and runs concurrent tests
Isolated testing workers	Run Nuclei, Metasploit, and PoCs
Database	Stores vulnerability records, job status, and results
Evidence storage	Keeps logs, test results, and reports
Vulnerability analysis	Correlates findings and identifies potential attack chains
Security and operations	Firewall, access controls, secrets, logging, monitoring, backups

Server 1 obtains vulnerability records, approved PoCs, tool packages, and updates from internal services on the air-gapped network.

3. Server 2 — Proxmox, Ludus, and virtual machines

Component	Purpose
Debian 13 / Proxmox VE 9	Physical server operating system and hypervisor
QEMU / KVM	Runs virtual machines
Ludus server and API	Receives commands from Server 1 and manages ranges
Ansible	Installs and configures software inside test VMs
Packer	Builds reusable VM templates from operating-system ISOs
WireGuard server	Provides secure access from Server 1
Linux bridges / VLANs	Create separate virtual test networks
Range router / firewall VMs	Control network access between VMs and internal resources
dnsmasq	Provides DHCP and DNS for Ludus provisioning
VM templates and ISO storage	Supplies operating systems for test environments
VM applications and test targets	Software and systems used to reproduce reported vulnerabilities
VM snapshots / storage	Supports restoring environments and repeatable tests
Security and operations	Proxmox firewall, audit logs, monitoring, backups

Ludus installs and configures several of these dependencies itself, including Ansible, Packer, WireGuard, dnsmasq, and its networking components.

Unlike the previous design, Server 2 can access internal package repositories directly, while its virtual machines obtain their dependencies through controlled network paths.

4. Network connections and security rules

I recommend three network segments, controlled through your existing switches and firewalls.

Network	Connected systems	Purpose
Management VLAN	Server 1 and Server 2	Ludus management, administrative access, monitoring
Repository / services VLAN	Both servers and approved VM traffic	Download packages, ISOs, updates, and dependencies
Isolated test networks	Ludus virtual machines	Run vulnerability tests without exposing the wider network

These can use existing physical network interfaces with VLANs, provided the network equipment supports them and the configuration is validated with Ludus. Ludus assumes a relatively simple host network configuration.

Important network ports

Connection	Port / Protocol	Purpose
Server 1 → Server 2	UDP 51820	WireGuard tunnel
Server 1 → Ludus	TCP 8080 inside tunnel	Ludus API
Authorized administrators → Server 2	TCP 8006	Proxmox management
Both servers / approved VMs → Repository	TCP 443	Download dependencies
Both servers / approved VMs → Internal DNS	TCP/UDP 53	Resolve internal service names
Both servers / approved VMs → Internal NTP	UDP 123	Time synchronization
Authorized administrators → Servers	TCP 22	SSH maintenance

These are common/default ports; enable only services actually required.

5. Internal resources and dependency management

The wider air-gapped network should host an approved repository or mirror service that both servers and permitted VMs can access.

Resource	Used by
Debian and Proxmox APT mirrors	Server 2; Debian-based test VMs
Ludus binaries and installation dependencies	Server 2
Ansible Galaxy roles and collections	Ludus on Server 2
Packer plugins and VM templates	Ludus on Server 2
Operating-system ISOs	Server 2 for creating VMs
Windows updates / WSUS	Windows test VMs
Python, Go, and other package repositories	Both servers and applicable VMs
Container image registry	Server 1 test workers; VMs when needed
Nuclei templates, Metasploit modules, and PoCs	Server 1 testing workers
CVE records and security advisories	Server 1
Application installers, firmware, and drivers	Test VMs
DNS, NTP, internal certificates	Both servers and applicable VMs

These services can be shared with the organization’s existing offline infrastructure rather than duplicated on the two servers.

How resources enter the environment

* An approved external system gathers software, updates, vulnerability data, and dependencies.
* Resources undergo security review, signature/hash verification, and approved import into the air-gapped network.
* The internal repository stores the approved resources.
* Server 1 retrieves vulnerability information and testing tools.
* Server 2 retrieves Proxmox, Ludus, and VM-building dependencies.
* Test VMs access approved internal repositories to install applications or required packages.

Ludus can also build VM templates using locally staged ISOs rather than downloading them directly.

6. Critical security considerations

The biggest difference from the previous design is that Server 2 now has direct network access to the wider air-gapped environment.

I would require the following controls:

* Block test VM access to management networks. VMs should not reach Proxmox management, Ludus administration, or unrelated organizational servers.
* Allowlist dependency access. Permit VMs to contact specific approved repository addresses and ports, rather than the entire air-gapped network.
* Restrict Ludus network defaults. Ludus allows outbound traffic and inter-VLAN communication by default. Set restrictive policies, including external_default: REJECT and appropriate blocked networks, with explicit exceptions.
* Use a controlled proxy or egress gateway for VMs. Ludus uses NAT, which can make VM traffic appear to originate from Server 2. A separate controlled access path helps distinguish VM downloads from legitimate hypervisor traffic.
* Separate provisioning from testing. Allow dependency downloads while preparing a VM, then remove unnecessary access before running a vulnerability test.
* Protect critical services. Keep backups, audit logs, secrets, and management access protected. Test firewall rules after deployments and configuration changes.

Also verify that Ludus’s default test-network IP ranges do not overlap with existing organizational networks. Its documentation specifically warns about possible conflicts with internal 10.x.x.x networks.

7. Overall recommendation

I prefer this shared-network design over the previous gateway-only design when internal repository access is a major operational requirement, provided network segmentation can be enforced.

Area	Previous design	New design
Server 2 dependency access	Through Server 1	Direct to internal repositories
VM dependency access	Through tightly controlled Server 1 paths	Through dedicated firewall / proxy
Operational complexity	Higher	Lower
Security exposure	Smaller direct network footprint	Larger, requiring stronger controls
Dependency availability	Dependent on Server 1	Independent of Server 1
Ludus remote management	Server 1	Server 1

Final recommendation: Keep Server 1 responsible for orchestration, Server 2 responsible for virtualization, and the wider air-gapped network responsible for shared dependencies. Use existing firewalls, a dedicated management VLAN, and a tightly restricted repository-access path for VMs.

The objective is to give both servers the resources they need without allowing vulnerable test environments unrestricted access to the organization’s network.












I recommend the two-server architecture for your Promox project. It is a sensible design for production vulnerability validation, particularly if you want to support multiple testing tools, automate workflows, and operate in an air-gapped environment.

However, I would make one important adjustment: separate the orchestration software from the actual vulnerability-testing processes, even if they share the first physical server.

1. Recommended architecture

Server 1 — Orchestration and Testing

Controller • Job queue • Test workers • Nuclei / Metasploit / PoCs • Results database

Internal network • Ludus API / WireGuard

Server 2 — Proxmox and Ludus

VM provisioning • Isolation • Snapshots • Range management

Target VM 1

Target VM 2

Target VM 3

Isolated vulnerability-validation environments

Server 1: Decides what to test, deploys tests, manages execution, collects evidence, and determines whether a reported vulnerability is confirmed, rejected, or inconclusive.

Server 2: Hosts the virtual environments containing the software or systems being tested. Ludus manages the deployment and configuration of these environments through Proxmox.

This separation is consistent with Ludus’s architecture: its CLI and API can be accessed remotely, so orchestration does not need to run directly on the Proxmox host.

2. Recommended production specifications

These are my engineering recommendations, not official minimum requirements. I’m sizing for a small-to-medium deployment with several simultaneous testing jobs and multiple virtual machines.

Component	Server 1: Orchestration + testing	Server 2: Proxmox + Ludus
CPU	12–16 physical cores	24–32 physical cores
RAM	128 GB ECC	256–512 GB ECC
Storage	2–4 TB usable NVMe SSD	4–8 TB usable NVMe SSD
Network	10 GbE preferred	10 GbE preferred
Operating system	Debian 13	Proxmox VE 9 / Debian 13
Virtualization	Optional for basic scanning; useful for isolated workers	Required, with Intel VT-x or AMD-V
Storage protection	Mirrored enterprise SSDs	Mirrored enterprise SSDs
Form factor	Enterprise server	Enterprise virtualization server

Usable storage figures assume redundancy has already been accounted for. CPU figures refer to physical cores, not hyper-threaded logical CPUs.

For Server 2, Ludus officially requires at least 32 GB RAM and 200 GB storage for installation, and its documentation recommends approximately 32 GB RAM per deployed range. Those are minimum planning figures, not sufficient specifications for a busy production lab.

My preferred starting purchase would therefore be 16 cores / 128 GB for Server 1 and 32 cores / 512 GB for Server 2, provided the expected workload warrants that capacity.

3. Important design considerations

Consideration	Assessment
Separation of responsibilities	Strong advantage. Orchestration and virtualization can be maintained independently.
Testing performance	Good, provided test workers have dedicated CPU and RAM resources.
Isolation	Better than running everything on the Proxmox host, but testing workers should remain isolated from the controller.
Air-gapped deployment	Practical, but software repositories, test dependencies, operating-system images, and updates must be staged internally.
Reliability	A major limitation: each server is a single point of failure.
Expansion	You can add testing workers or Proxmox nodes as demand increases.
Cost	Two physical servers may be excessive for a small number of occasional tests.

Three details are particularly important.

First — Don’t install Docker on the Ludus host. Ludus explicitly warns that Docker can interfere with host networking. Docker-based testing workers are appropriate on Server 1, but Server 2 should remain dedicated to Ludus and virtualization.

Second — Keep test execution isolated. On Server 1, run vulnerability-testing tools in disposable VMs or suitably isolated containers, while keeping the controller, database, and credentials protected. For more demanding or higher-risk tests, dedicated VMs offer stronger isolation. Certain tests may also need to execute inside a Ludus range.

Third — Build in recovery and backup. Two servers don’t provide high availability. A Server 2 failure stops the hosted ranges. A Server 1 failure stops orchestration. For production, I would require independent backups, tested restoration procedures, monitoring, and a recovery plan. Ludus supports larger Proxmox clusters if you eventually need to expand.

For an air-gapped network, I would also reserve storage and compute capacity for internal package mirrors and test artifacts. Ludus currently documents internet access as an installation requirement, so fully offline deployment requires additional preparation and validation.

4. How the workload affects specifications

The largest uncertainty is the number and type of concurrent tests.

Workload	Hardware implication
Nuclei, lightweight PoCs, API testing	Relatively modest CPU and RAM requirements
Metasploit and multiple concurrent test workers	More CPU, RAM, and isolation capacity on Server 1
Firmware rehosting with QEMU / Penguin	Can be CPU- and memory-intensive, especially without hardware acceleration
Large Windows/Linux test ranges	Substantially more RAM and SSD capacity on Server 2
Local LLM inference	Potentially requires additional RAM and a dedicated GPU on Server 1

For a smaller installation, 8 cores / 64 GB on Server 1 and 16 cores / 128 GB on Server 2 may be sufficient. I would not buy the larger configuration without estimating the actual concurrent workload.

One more consideration: if Server 2 is being built from scratch, Ludus’s documentation recommends starting with bare-metal Debian 12/13 rather than an existing customized Proxmox installation. Ludus can install and configure the virtualization components itself.

5. Overall assessment

I would retain the two-server model, with the following priorities:

* Server 1: Reliable orchestration, isolated testing workers, evidence collection, and automation.
* Server 2: Maximum practical RAM, fast redundant NVMe storage, and sufficient CPU cores for the test VMs.
* Infrastructure: Separate management and testing networks, internal dependency mirrors, independent backups, and network restrictions.

The arrangement is a good starting point for production operations, but it should not be mistaken for a fully redundant production platform.

One question before finalizing the specifications: Approximately how many target VMs and simultaneous vulnerability tests do you expect the system to handle at peak — for example, 10, 25, 50, or more? That will determine whether 256 GB or 512 GB on the Proxmox host, and 64 GB or 128 GB on the testing server, are actually justified.









For Promox, GPUs can help with vulnerability validation, identifying attack chains, analyzing security data, and prioritizing patches. However, their practical value depends on the workload.

GPU use cases for vulnerability testing

Use case	How GPU helps	Practical value
1. Vulnerability chain analysis	Analyzes large numbers of connections between vulnerabilities, systems, and permissions to identify potential attack paths.	High for very large environments
2. Patch prioritization	Repeatedly calculates which combinations of patches could eliminate the most potential attack paths.	Medium–High at scale
3. Vulnerability pattern analysis	Compares large datasets to identify recurring weaknesses, affected components, and related findings.	Medium
4. Password security testing	Tests large numbers of password-hash candidates simultaneously using Hashcat.	High when password auditing is in scope
5. Large-scale vulnerability correlation	Matches millions of software records, affected versions, CVEs, and system inventories.	Medium–High at scale
6. Firmware and binary analysis	Accelerates specialized bulk pattern searches across large collections of firmware or binaries.	Low–Medium
7. Fuzz testing	Supports specialized GPU fuzzers, particularly for discovering bugs in GPU software.	High for GPU software; low otherwise
8. Network traffic analysis	Accelerates large-scale packet and signature matching with GPU-enabled software.	Medium for high-volume traffic
9. Cryptographic vulnerability testing	Performs large numbers of repetitive cryptographic calculations to assess certain weaknesses.	High for suitable cryptographic tests
10. Running many PoCs simultaneously	Usually offers little benefit because PoCs are generally CPU-, network-, or target-dependent.	Low

Examples of established GPU-capable technologies include NVIDIA cuGraph for graph analysis, NVIDIA cuDF for large datasets, Hashcat for hash auditing, and cuFuzz for GPU software fuzzing.

My priorities for Promox

Given your goal of validating reported vulnerabilities and determining what needs patching, I would prioritize these three possible GPU workloads:

* Vulnerability chain analysis: Find possible combinations of vulnerabilities that could allow attackers to move between systems.
* Patch impact analysis: Determine which vulnerabilities are most important to patch because fixing them could break multiple potential attack chains.
* Large-scale vulnerability correlation: Analyze relationships between reported vulnerabilities, installed software, firmware, system configurations, and test results.

These three can share GPU-accelerated graph and data-processing infrastructure. However, GPUs only accelerate the calculations; they do not independently determine whether an attack chain is actually exploitable.

Which server should have the GPU?

Server	Recommendation
Server 1 — Orchestration	GPU optional; useful for large-scale vulnerability analysis and correlation
Server 2 — Proxmox/Ludus	GPU generally unnecessary unless testing GPU-specific software

My recommendation: Make Server 1 GPU-ready, but begin with CPU-based vulnerability analysis. Add a GPU when the actual volume of data or graph calculations justifies it.

For Promox, accurate vulnerability relationships and reliable test evidence matter more than GPU performance initially.


