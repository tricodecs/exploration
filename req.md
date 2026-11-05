Air-gapped Ludus — Two-server staging setup

Objective: Prepare and validate a working Ludus installation in a staging environment, then package the necessary components for deployment into the air-gapped environment.

We will use:

* Server 1 — Proxmox server: Proxmox VE 9 is already installed. Ludus will run directly on this server.
* Server 2 — Control server: A separate Linux server used to download and stage components, administer Ludus, deploy ranges, and verify the installation.
* Initial range: One Debian 13 test VM and one Debian 13 router VM.
* Ludus version: 2.3.10, the latest stable release confirmed in the official release listing. GitLab

Architecture

Server 2 — Control server

Download staging • Ludus CLI • WireGuard client • Range YAML

Staging management network

API / VPN / file transfer

Server 1 — Proxmox VE 9 + Ludus

Ludus server • Ansible • Packer • VM storage • Virtual networking

Router VM

Debian 13

Test VM

Debian 13

The important distinction is that Ludus server, Ansible, Packer, and VM virtualization run on Server 1, while Server 2 normally needs only the Ludus CLI, secure connectivity, and administration tools. Ludus

Group 1 — Server 1: Proxmox + Ludus

Ludus installed here

1. Ludus application files

Component	Exact file	Download
Ludus server	ludus-server-2.3.10	Official binary
Checksums	ludus_2.3.10_checksums.txt	Official checksums
Installer reference	install.sh	Official installer

The Ludus server binary includes its installation playbooks and built-in VM template definitions. You do not need the full Ludus source-code repository or Go compiler when using the precompiled binary. GitHub +1

2. Proxmox host packages

These packages must be available on Server 1, either already installed or supplied through an offline Debian package repository.

Best source: Official Debian 13 APT repository⁠￼.

Category	Exact packages
Networking	bridge-utils, dnsmasq, iptables-persistent, wireguard-tools
Utilities	git, curl, jq, vim, sshpass
System	gpg-agent, dbus, dbus-user-session
Python support	python3-pip, python3-debian

All dependent Debian packages must be included as well. These 14 named packages are identified in Ludus’s existing-Proxmox installation workflow and WireGuard setup. GitHub +1

3. Python and Ansible packages

Best source: Official Python Package Index⁠￼.

Component	Version or requirement
ansible	9.3.0 for the selected bootstrap workflow
netaddr	1.2.1 for bootstrap
proxmoxer	Version compatible with installation
requests	Version compatible with installation
pywinrm	Version compatible with installation
dnspython	Version compatible with installation
jmespath	Version compatible with installation

Include the Python dependency closure for the selected versions.

These libraries are installed on the Proxmox/Ludus server, not on the control server. The Proxmox playbook explicitly requests the six runtime Python libraries. GitHub

4. Packer and plugins

Component	Version	Download
HashiCorp Packer	1.9.4	HashiCorp
Proxmox Packer plugin	1.2.4 for Proxmox 9.1+	GitHub release
Ansible Packer plugin	1.1.1	GitHub release

Compatibility warning: Ludus 2.3.10’s installation playbook requests Proxmox plugin 1.2.3. If using 1.2.4, the offline installation process must be adjusted accordingly. GitHub

Packer and its plugins are included because the normal Ludus installation configures them, even if the first range uses a prebuilt VM template.

5. Ansible Galaxy dependencies

Best source: Download using the version-pinned Ludus 2.3.10 requirements file⁠￼.

3 ANSIBLE ROLES

Role	Version
lae.proxmox	v1.10.0
geerlingguy.packer	1.0.2
ansible-thoteam.nexus3-oss	v2.5.2

8 ANSIBLE COLLECTIONS

Collection	Version
ansible.windows	2.7.0
ansible.posix	1.6.2
ansible.utils	5.1.2
community.general	10.3.0
community.windows	2.3.0
chocolatey.chocolatey	1.5.3
microsoft.ad	1.8.0
vladgh.samba	3.4.0

All 11 are explicitly listed in the official requirements file and requested by the standard installation process. GitHub +1

The Nexus role does not mean you must deploy a Nexus repository server.

6. Debian 13 VM image and guest dependencies

Recommended: Use a prebuilt Debian 13 Proxmox VM template instead of building from an ISO.

Component	Requirement
Debian 13 VM backup	Prepared outside the air gap, for example .vma.zst
SSH	openssh-server, enabled
Python	python3
Privilege escalation	sudo, with expected Ludus user permissions
Guest management	qemu-guest-agent, enabled
Guest networking	DHCP-enabled interface
VM template storage	Proxmox template registered in the SHARED pool
Guest package dependencies	Preinstalled or available from an internal Debian package repository

Official requirements: Ludus template documentation⁠￼. Ludus

A compatible Debian 13 template can be used as the base for both the router and test VM. However, router provisioning may require additional Debian packages, so the template must be prepared and tested for fully offline deployment.

7. Ludus host configuration

These are required configurations created or managed by the Ludus installer, not separate software downloads.

Component	Purpose
/opt/ludus	Ludus application and configuration files
ludus and ludus-admin services	Ludus API and administration
SHARED and ADMIN Proxmox pools	VM template and resource management
Proxmox storage permissions	Allows Ludus to create and manage VMs
Virtual bridges and NAT	Connects and isolates range networks
WireGuard server	Secure remote access
Administrator account and API key	Authentication and management
Internal DNS, routing and firewall rules	Supports provisioning and containment

Ludus normally creates these during installation. Do not manually duplicate their setup unless implementing a validated offline modification. Ludus

Group 2 — Server 2: Control server

Ludus management and staging

This server is responsible for interacting with Ludus, preparing deployment configurations, and collecting installation packages.

For simplicity, assume Server 2 runs Debian 13 (AMD64).

1. Required control software

Component	Exact package/file	Download
Ludus CLI	ludus-client_linux-amd64-2.3.10	Official binary
WireGuard client	wireguard-tools	Debian package
SSH client	openssh-client	Debian package
Certificate verification	ca-certificates	Debian package

These are sufficient for the normal Linux CLI management workflow, provided basic operating-system utilities are already present.

Ludus documents using its CLI over WireGuard to manage ranges and access deployed VMs. Ludus

2. Installation preparation tools

Since Server 2 also prepares the offline bundle, it needs tools to collect and organize dependencies.

Component	Purpose	Source
curl or wget	Download binaries and installation files	Debian
tar, unzip	Extract and package downloads	Debian
coreutils	Verify SHA-256 checksums	Debian
APT download tools	Collect Debian .deb packages and dependencies	Debian repository
Python pip	Collect Python package wheels	PyPI
Ansible Galaxy CLI	Collect the required roles and collections	Ansible documentation

These tools are for preparing the offline installation, not for running the Ludus server on Server 2.

If Server 2 cannot access the internet, downloads must be obtained from another approved source and brought into staging.

3. Required configuration and credentials

Component	Purpose
Ludus API key	Authenticates CLI requests
WireGuard client configuration	Connects Server 2 to Ludus
range-config.yml	Defines the Debian 13 test range
SSH access to Server 1	Transfers installation files and performs initial administration
Optional Proxmox web browser access	Manages VM templates and examines VM consoles

The initial administrator setup must happen locally on Server 1 because Ludus’s privileged administration API listens only on localhost by default. After that, Server 2 can administer Ludus with its assigned API key. Ludus +1

4. Network connections between the servers

Connection	Port	Purpose
Server 2 → Server 1	TCP 22	SSH and file transfer, if used
Server 2 → Server 1	UDP 51820	WireGuard VPN
Server 2 → Ludus API	TCP 8080	CLI management, preferably through the VPN
Server 2 → Proxmox	TCP 8006	Optional Proxmox web management

Ludus’s privileged admin API is TCP 8081, bound to localhost by default; it should not be exposed to Server 2 for routine operation. Ludus +1

5. What is not required on Server 2?

Component	Required?
Ludus server binary	No
Proxmox VE	No
Packer and Packer plugins	No
Ansible runtime for deploying Ludus ranges	No
Ludus server Python dependencies	No
VM templates installed locally	No
Go compiler	No
Nexus Repository Manager	No

These components either belong on Server 1 or are unnecessary for this initial deployment.

Staging installation sequence

1

Server 2 — Collect downloads

Download the Ludus server and CLI binaries, host packages, Python dependencies, Packer components, and Ansible content.

Verify checksums, resolve transitive dependencies, and retain exact package versions.

2

Server 1 — Install Ludus

Transfer the dependencies from Server 2. Install Ludus directly on Proxmox and complete the networking, storage, service, and account setup.

For an offline rehearsal, use the locally staged packages and an adapted installation workflow.

3

Server 1 — Prepare the Debian template

Create and configure a Debian 13 VM on the staging Proxmox host, or import a compatible prebuilt template.

Convert it into a Proxmox template and place it in the SHARED pool. Export a backup for the eventual air-gap transfer.

4

Server 2 — Configure and deploy

Install the CLI and WireGuard client, obtain the appropriate API key and VPN configuration, then submit the minimal range-config.yml to Ludus.

Deploy the Debian router and test VM.

5

Both servers — Verify offline operation

Disable external internet access and confirm that Ludus services, range creation, guest configuration, and VM connectivity succeed without external downloads.

Resolve missing package or repository dependencies before packaging the environment for the air gap.

Verification and completeness assessment

I cross-checked the component separation against the official Ludus 2.3.10 installation playbook, Galaxy requirements, Proxmox deployment documentation, template requirements, and CLI instructions.

The important findings are:

* Ludus must run on Server 1. Server 2 is a client and staging machine, not a second Ludus installation.
* Server 2 does not need Packer, Go, or the Ludus server’s Python dependencies unless you deliberately assign additional build tasks to it.
* The prebuilt Debian template avoids ISO-based template building, but does not automatically eliminate package downloads during guest provisioning.
* Offline installation remains a custom workflow. The official Ludus installer has no verified all-in-one offline mode. Its external package and plugin downloads must be resolved locally.
* The named component list is verified, but the exact complete download bundle is environment-dependent. The Debian, Python, and guest package dependency closures must be captured and tested against the target host versions.

Final recommendation: Use Server 2 to prepare and manage the staging installation, and Server 1 exclusively for Proxmox, Ludus, and the test VMs. Once a fully disconnected installation and range deployment work in staging, export the validated installation bundle and Debian VM template for transfer into the actual air-gapped environment.









Complete component list — Air-gapped Ludus range

Assumptions:

* Proxmox VE 9 with Debian 13 is already installed.
* The server has no internet access.
* You want the simplest Ludus range: one Debian 13 test VM and one router VM.
* We will use Ludus 2.3.10 and precompiled binaries rather than compiling from source.

Recommendation: Prepare one offline installation bundle and one prebuilt Debian 13 VM template outside the air gap, then transfer both into the air-gapped network.

This avoids building the Debian template inside the air gap. However, Ludus’s standard host installer still requires Packer and Ansible components.

1. Ludus application

3 RELEASE FILES + INSTALLATION SCRIPT

Component	Download
ludus-server-2.3.10	Official server binary
ludus-client_linux-amd64-2.3.10	Official CLI binary
ludus_2.3.10_checksums.txt	Checksums
install.sh	Installer reference

The server binary embeds the installation playbooks and built-in templates. The installation script is useful as a reference, but must be adapted for offline use. GitHub +1

2. Proxmox host software dependencies

REQUIRED FOR LUDUS INSTALLATION

A. Debian packages

Download from Official Debian repositories⁠￼ using a matching Debian 13 staging machine.

Category	Packages
Networking	bridge-utils, dnsmasq, iptables-persistent, wireguard-tools
System utilities	git, curl, jq, vim, sshpass
Services	gpg-agent, dbus, dbus-user-session
Python support	python3-pip, python3-debian

These are 14 explicitly identified host packages. Also collect their required .deb dependencies. Some packages may already be installed on Proxmox. GitHub

B. Python packages

Download from PyPI⁠￼.

Package	Version
ansible	9.3.0
netaddr	1.2.1
proxmoxer	Not pinned
requests	Not pinned
pywinrm	Not pinned
dnspython	Not pinned
jmespath	Not pinned

Include all transitive Python dependencies. Pin the unversioned packages to versions tested on the staging machine.

C. Packer and plugins

Component	Version	Download
HashiCorp Packer	1.9.4	ZIP
Proxmox Packer plugin	1.2.4	GitHub release
Ansible Packer plugin	1.1.1	GitHub release

Version detail: The Ludus 2.3.10 installer hardcodes Proxmox plugin 1.2.3. Using 1.2.4 requires adapting that installation task; simply placing 1.2.4 in the bundle is insufficient. GitHub

3. Ansible Galaxy dependencies

3 ROLES + 8 COLLECTIONS

Best download method: Use Ludus’s official requirements file on a connected staging machine to collect the exact versions.

Official Ludus 2.3.10 requirements.yml⁠￼

Ansible roles

Role	Version
lae.proxmox	v1.10.0
geerlingguy.packer	1.0.2
ansible-thoteam.nexus3-oss	v2.5.2

Ansible collections

Collection	Version
ansible.windows	2.7.0
ansible.posix	1.6.2
ansible.utils	5.1.2
community.general	10.3.0
community.windows	2.3.0
chocolatey.chocolatey	1.5.3
microsoft.ad	1.8.0
vladgh.samba	3.4.0

These are verified against Ludus 2.3.10’s installation requirements. The standard installer explicitly installs this Galaxy content, even though a simple Debian range does not use all of it. GitHub +1

Package the roles and collections as archives or preinstalled directories, preserving their expected names and locations.

4. Components needed to create the range

1 PREBUILT TEMPLATE + CONFIGURATION

Component	Required details	Source
Debian 13 VM template	Prebuilt Proxmox template, exported as a VM backup such as .vma.zst	Create on a staging Proxmox server
Guest packages	openssh-server, python3, sudo, qemu-guest-agent	Debian 13 packages
Guest configuration	DHCP, enabled SSH, working guest agent, and Ludus-compatible sudo credentials	Ludus template requirements
range-config.yml	Specifies the Debian test VM and router settings	Ludus range configuration
Guest package availability	Packages preinstalled or an internal APT repository for anything deployment needs to install	Debian repository

For the standard Debian credentials, Ludus documents debian:debian. Use these only in the isolated demonstration environment, and ensure the template has the correct sudo permissions. Ludus

One Debian 13 VM template

Router VM

Network routing, DHCP and DNS

Test VM

Debian 13

Standard simple range: 2 VMs, using the same Debian base template.

Ludus expects the VM template in the Proxmox SHARED pool. A manually created template is supported when it satisfies Ludus’s guest requirements. Ludus

5. Required configuration and access

These are necessary for the environment to function, but they are configuration items, not additional download packages.

Component	Purpose
Proxmox VM storage	Holds templates and deployed VMs
Ludus SHARED and ADMIN pools	Organizes templates and managed VMs
Ludus virtual bridges and NAT	Provides range networking
Ludus WireGuard server	Supports secure client access
Ludus systemd services	Runs the server and administration API
Ludus administrator account and API key	Authenticates management operations
WireGuard client on the administrator workstation	Required for the documented remote VPN access method
Internal network and firewall rules	Allows management while isolating vulnerable VMs from the wider air-gapped environment

The Ludus installation playbooks create most of these automatically. Your offline-adapted installer must retain that behavior. Ludus +2

The hardware should also meet Ludus’s documented baseline: virtualization-capable x86-64 CPU, at least 32 GB RAM, and at least 200 GB storage, with additional capacity for ranges. Ludus

6. Components you do not need for this initial range

Component	Required?	Reason
Ludus application source-code repository	No	Using precompiled binaries
Ludus Source BSL repository	No	Debian 13 template definition is built in
Go compiler and modules	No	No source compilation
Debian 13 installation ISO	No	Using prebuilt template
Nexus Repository Manager	No	Not necessary for this simple range
Docker	No	Not required; Ludus advises against Docker on the host
Additional Windows/Kali images	No	Only Debian 13 is used
Vulnerability-testing applications	No	Not needed just to stand up the range

Note: The Nexus Ansible role in section 3 is still part of the standard installation requirements. That does not require deploying a Nexus server.

7. Recommended deployment order

* Prepare a connected staging machine matching Proxmox VE 9 / Debian 13.
* Download Ludus binaries and verify their checksums.
* Collect all host dependencies, including the complete Debian package closure, Python wheels, Packer plugins, and Ansible Galaxy archives.
* Prepare one Debian 13 VM template with SSH, Python, sudo, DHCP, and QEMU Guest Agent. Export its backup.
* Build the offline installation bundle and validate it on an equivalent disconnected staging host.
* Transfer into the air gap, install Ludus using locally staged dependencies, and configure the host networking and storage.
* Restore the template, apply the simple range configuration, and deploy.
* Verify both VMs, their network access, and that deployment finishes without external downloads.

8. Accuracy and completeness assessment

Area	Assessment
Ludus application files	Exact release files identified
Named host packages	Verified against installation playbook
Python dependencies	Named packages identified; transitive versions require resolution
Packer components	Verified, including installer version mismatch
Ansible roles and collections	All 11 requirements verified
Range OS and configuration	Verified from Ludus documentation
Fully offline installation procedure	Requires customization and staging validation

Important limitation: Ludus’s official installation assumes internet access, and the installer includes download operations. There is no verified turnkey, air-gapped installation package. An exact complete set of individual .deb and .whl files cannot be determined without examining the target Proxmox host’s installed packages and resolving dependencies.

Final recommendation: Use the precompiled Ludus 2.3.10 binaries, collect the complete host dependency set, and transfer one ready-to-use Debian 13 VM template. This is the simplest practical approach, with the fewest additional dependencies for creating your first air-gapped Ludus range.


