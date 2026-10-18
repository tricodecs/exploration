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


