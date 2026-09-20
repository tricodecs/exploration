Firmware Rehosting and Vulnerability Validation Lab

1. Purpose

This document describes a practical lab architecture for:

* Rehosting embedded-device firmware without the original physical hardware.
* Using Penguin, IGLOO, PANDA, and QEMU together.
* Hosting the analysis environment as a Linux VM under Proxmox.
* Identifying firmware services and exercising them with appropriate testing tools.
* Reproducing and validating reported vulnerabilities.
* Determining whether a reported CVE or vulnerability is:
    * Confirmed
    * Not affected
    * Mitigated
    * Not reproduced / inconclusive
    * A likely false positive

The intended use is controlled firmware analysis, vulnerability validation, and defensive security research.

⸻

2. Overall Architecture

For a Proxmox-based lab, the recommended structure is:

Physical Server
      │
      ▼
Proxmox VE
      │
      ▼
Ubuntu Linux Analysis VM
      │
      ├── Docker / Podman
      │
      ├── Penguin
      │    └── IGLOO guest/runtime components
      │
      ├── PANDA / QEMU
      │
      └── Analysis Tools
           ├── AFL++
           ├── boofuzz
           ├── Nmap
           ├── Burp Suite / OWASP ZAP
           └── debugging / tracing tools
                   │
                   ▼
             Firmware Target

Proxmox runs the Linux VM.

PANDA is not built into Proxmox.

Proxmox uses QEMU/KVM for general virtualization. PANDA is a separate dynamic-analysis platform derived from QEMU and adds functionality designed for program and security analysis.

A useful mental model is:

Proxmox
   │
   └── hosts the analysis VM
           │
           ▼
        Penguin
           │
           ├── orchestrates firmware rehosting
           │
           └── IGLOO helps satisfy hardware expectations
                       │
                       ▼
                  PANDA / QEMU
                       │
                       ▼
                    Firmware

⸻

3. What Firmware Rehosting Means

Firmware rehosting means taking firmware designed for a physical embedded device and running it in an emulated or reconstructed environment.

Examples include firmware from:

* Routers
* IoT devices
* Cameras
* Gateways
* Industrial devices
* Network appliances
* Embedded controllers

On the real device:

┌───────────────────────────┐
│ Firmware                  │
│      ↓                    │
│ CPU + RAM + peripherals   │
│      ↓                    │
│ Physical hardware         │
└───────────────────────────┘

When rehosted:

┌───────────────────────────┐
│ Same firmware             │
│      ↓                    │
│ QEMU / PANDA              │
│      ↓                    │
│ IGLOO                     │
│      ↓                    │
│ Penguin orchestration     │
└───────────────────────────┘

QEMU/PANDA provides CPU and system emulation.

IGLOO helps deal with firmware assumptions about hardware that does not physically exist in the analysis VM.

Penguin orchestrates the overall rehosting environment.

Therefore:

Emulation is one mechanism used by rehosting. Rehosting is the larger process of making firmware execute sufficiently like it does on the original device.

⸻

4. Penguin and IGLOO

Assuming the MIT Lincoln Laboratory rehosting/igloo_driver and rehosting/penguin stack:

You generally do not need separate VMs for Penguin and IGLOO.

IGLOO is part of the guest-side/runtime mechanism used during firmware rehosting, while Penguin provides the host-side orchestration environment.

Conceptually:

Penguin
   │
   ├── manages execution
   ├── configures firmware environment
   ├── integrates analysis components
   │
   └── IGLOO runtime
           │
           ▼
     Firmware execution

The result is one primary analysis environment rather than an IGLOO VM plus a Penguin VM.

⸻

5. PANDA

PANDA stands for:

Platform for Architecture-Neutral Dynamic Analysis

PANDA is based on QEMU.

The simplest relationship is:

QEMU
 │
 └── CPU / machine emulation
          │
          ▼
PANDA
 │
 └── QEMU + recording + replay + instrumentation + analysis
          │
          ▼
Penguin + IGLOO
 │
 └── firmware rehosting and analysis environment

QEMU primarily answers:

Can this ARM, MIPS, x86, or other machine execute?

PANDA adds capabilities useful for detailed investigation.

Examples include:

* Record/replay
* Execution tracing
* Memory analysis
* Process observation
* Plugin-based instrumentation
* Coverage collection
* Taint analysis
* Debugging
* Repeatable execution analysis

Suppose a router firmware web server receives an input that reportedly triggers a vulnerability.

With ordinary QEMU, you might observe:

request
   ↓
firmware
   ↓
crash

With PANDA, you can investigate:

request
   ↓
which process received it?
   ↓
which code executed?
   ↓
which function processed it?
   ↓
where did the input propagate?
   ↓
what memory/state changed?
   ↓
what happened immediately before the crash?

This is why PANDA is particularly useful when determining whether a vulnerability report describes a real reachable condition.

⸻

6. Recommended VM Resources

There are no universally published minimum CPU and RAM requirements for all Penguin workloads, so CPU/RAM sizing should be treated as practical recommendations.

Resource	Practical Minimum	Recommended
vCPU	4	8
RAM	8 GB	16–32 GB
Disk	50 GB SSD	100–150 GB SSD/NVMe
OS	Linux x86-64	Ubuntu 22.04/24.04 x86-64
Container engine	Docker or Podman	Docker
Nested virtualization	Optional	Enabled
GPU	None	None
Network	Needed initially for dependencies	Internet or internal mirror

For a permanent single-user analysis VM:

8 vCPU
16 GB RAM
120 GB SSD/NVMe
Ubuntu 24.04 x86-64
Nested virtualization enabled
Docker

For development, multiple simultaneous firmware targets, frequent image rebuilding, or larger analysis workloads:

12–16 vCPU
32 GB RAM
200 GB+ SSD/NVMe

⸻

7. Why Disk Space Matters

Firmware analysis environments accumulate considerably more storage than the firmware image itself.

Storage may be consumed by:

* Container images
* Penguin images
* Nix store generations
* Firmware images
* Extracted filesystems
* PANDA recordings
* QCOW images
* Snapshots
* Debugging artifacts
* Coverage data
* Traces
* Fuzzing corpora
* Crash samples
* Analysis reports

Therefore, a 20–30 GB analysis VM is unnecessarily restrictive.

A useful practical baseline is:

50 GB = functional lower bound
100–150 GB = comfortable analysis VM
200 GB+ = development / multiple targets / long-term lab

⸻

8. KVM and Nested Virtualization

KVM acceleration can substantially improve execution for compatible x86/x86-64 workloads.

If the Linux analysis VM itself runs under Proxmox:

Physical CPU virtualization extensions
             │
             ▼
          Proxmox
             │
             ▼
     nested virtualization
             │
             ▼
      Ubuntu analysis VM
             │
             ▼
             KVM

Enable nested virtualization when possible.

However, KVM is not required for every workload.

Firmware for architectures such as:

* ARM
* MIPS
* PowerPC
* other embedded architectures

will often rely on software translation/emulation anyway.

Without KVM, QEMU/PANDA can normally fall back to software emulation such as TCG, although performance is lower.

⸻

9. Firmware Analysis Workflow

A typical process is:

Obtain firmware
      ↓
Extract filesystem/images
      ↓
Determine architecture
      ↓
Identify kernel/userspace/services
      ↓
Configure Penguin/IGLOO
      ↓
Boot with PANDA/QEMU
      ↓
Verify normal operation
      ↓
Identify reachable services
      ↓
Test reported condition
      ↓
Record/replay execution
      ↓
Analyze behavior
      ↓
Determine vulnerability status

⸻

10. Tools and Their Roles

Different tools answer different questions.

Nmap

Use Nmap to determine what the rehosted firmware exposes.

For example:

Firmware
   ↓
Network stack
   ↓
80/tcp HTTP
443/tcp HTTPS
22/tcp SSH
custom service

This helps identify which externally reachable components are worth investigating.

⸻

Burp Suite / OWASP ZAP

Use these when the firmware exposes an HTTP/HTTPS management interface or API.

Typical uses include analyzing:

* Request handling
* Authentication
* Sessions
* Parameters
* API behavior
* Input validation
* Server responses

⸻

boofuzz

Useful when investigating network protocol implementations.

Examples:

TCP
UDP
HTTP-like protocols
binary protocols
proprietary device protocols

boofuzz manipulates protocol messages and observes the target’s behavior.

⸻

AFL++

AFL++ is useful for fuzz testing parsers and binaries.

It is particularly useful when source code is unavailable because binary instrumentation/emulation modes can be used.

Conceptually:

Seed input
   ↓
AFL++ mutations
   ↓
Target parser/service
   ↓
interesting execution?
   │
   ├── no → continue
   │
   └── yes
        ↓
     save input
        ↓
 crash / abnormal state

⸻

PANDA

PANDA is not primarily a vulnerability scanner.

Instead, it helps determine why something happened.

Examples:

Was the vulnerable code reached?
Did user-controlled input reach that code?
Did it affect the dangerous operation?
Where did execution diverge?
What happened immediately before the crash?

PANDA is therefore particularly useful after you already have:

* a vulnerability report,
* suspected trigger,
* crashing input,
* unusual behavior,
* or interesting fuzzing result.

⸻

11. Discovery vs Validation

It is important to distinguish two different activities.

Vulnerability discovery

You do not yet know what bug exists.

Example:

AFL++
  ↓
thousands/millions of test inputs
  ↓
crash discovered

Vulnerability validation

Someone has already reported a specific vulnerability.

Example:

CVE / security report
       ↓
specific affected component
       ↓
specific trigger
       ↓
reproduce condition
       ↓
analyze with PANDA

For validating an existing vulnerability, do not begin with broad fuzzing unless the report itself requires fuzzing.

First reproduce the specific claim.

⸻

12. What Is a CVE?

CVE stands for:

Common Vulnerabilities and Exposures

A CVE provides a standardized identifier for a publicly disclosed cybersecurity vulnerability.

Example:

CVE-2026-12345

The identifier itself does not tell you how severe the vulnerability is.

It primarily provides a common reference so researchers, vendors, scanners, defenders, databases, and patch-management systems can refer to the same vulnerability.

⸻

13. Information Associated With a CVE

A vulnerability record may contain or reference several related concepts.

CVE
 │
 ├── affected product/version
 ├── vulnerability description
 ├── references/advisories
 ├── CWE
 ├── CVSS
 ├── vendor advisory
 ├── patches/fixed versions
 ├── exploit prerequisites
 └── sometimes additional threat information

These should not be confused with one another.

⸻

14. CVE vs CVSS

CVE identifies the vulnerability.

CVSS describes severity characteristics.

CVSS stands for:

Common Vulnerability Scoring System

A CVSS score may range from:

0.0 → 10.0

For example:

CVE-20XX-XXXXX
CVSS: 9.8

does not mean:

The vulnerability definitely affects your device.

It means that, assuming the vulnerable condition exists under the assessed circumstances, its technical severity was scored accordingly.

Therefore:

high CVSS ≠ your system is definitely vulnerable

You still need to establish applicability and reachability.

⸻

15. CWE

CWE stands for:

Common Weakness Enumeration

CWE describes the general class of software weakness.

Examples include concepts such as:

buffer overflow
use-after-free
improper input validation
path traversal
command injection
authentication weakness

Relationship:

CWE
 ↓
type of weakness
CVE
 ↓
specific vulnerability instance

Multiple CVEs can therefore belong to the same CWE category.

⸻

16. CPE

CPE stands for:

Common Platform Enumeration

CPE identifiers are used to describe affected products/platforms in a standardized format.

They can help automated vulnerability-management tools determine whether a particular product/version might match a CVE.

However, version matching can generate false positives.

For example:

Scanner sees:
Component X version 1.2.3
CVE database says:
Component X <= 1.2.3 affected

The scanner may report the CVE even when:

* the vulnerable feature was disabled,
* the vendor backported a fix without changing the apparent version,
* the vulnerable function was removed,
* the component was compiled differently,
* the vulnerable service is unreachable,
* or the scanner identified the wrong software.

This is why scanner output should be treated as a lead requiring validation rather than automatic proof of exploitability.

⸻

17. NVD

NVD stands for:

National Vulnerability Database

NVD enriches vulnerability information and commonly provides information such as:

* CVSS data
* CPE mappings
* CWE classifications
* references

The authoritative technical remediation information, however, may come from the affected vendor’s security advisory.

When investigating a CVE, useful sources typically include:

CVE record
   +
vendor advisory
   +
NVD metadata
   +
release notes / patch
   +
your actual firmware

⸻

18. Vendor Advisories

The vendor advisory is particularly important because it may identify:

* exact affected firmware versions
* corrected versions
* required configuration
* affected services
* workarounds
* whether a patch was backported
* whether only certain models are affected

For firmware validation, this can be more informative than simply comparing version numbers.

⸻

19. CVSS vs Real-World Applicability

A CVSS score should not be treated as a direct measure of risk to your exact environment.

A firmware image might contain a vulnerable library but never expose the vulnerable functionality.

For example:

Vulnerable library exists
       ↓
Scanner detects version
       ↓
CVE reported

But:

affected function never invoked
       ↓
service disabled
       ↓
input cannot reach vulnerable parser
       ↓
no reachable vulnerable condition

In that situation, the CVE may technically be present in the software inventory while the reported attack path is not applicable to the deployed device configuration.

Your PANDA analysis can help distinguish those conditions.

⸻

20. Other Useful Vulnerability Metadata

Several additional data sources can help prioritize analysis.

EPSS

EPSS estimates the probability that a published vulnerability will be exploited in the wild over a defined near-term period.

It is useful for prioritization but does not prove your firmware is affected.

⸻

CISA Known Exploited Vulnerabilities Catalog

The KEV catalog identifies vulnerabilities known to have evidence of exploitation in the wild.

A vulnerability appearing there can increase urgency, but applicability still needs to be checked against:

your product
your version
your configuration
your firmware build

⸻

21. Before Testing a Reported CVE

Translate the report into a precise technical claim.

Determine:

CVE
 ↓
affected product?
 ↓
affected version?
 ↓
affected component?
 ↓
affected service?
 ↓
required configuration?
 ↓
required input?
 ↓
expected vulnerable behavior?
 ↓
claimed consequence?

Example:

Claim:
An externally supplied request reaches parser X.
A malformed field causes length calculation Y.
That length reaches memory operation Z.
The result is an out-of-bounds access.

That is testable.

A vague statement such as:

"Device is vulnerable to CVE-XXXX-YYYY"

is not enough by itself.

⸻

22. Step 1 — Confirm Version Applicability

Before running PANDA, verify:

firmware version
component version
library version
build date
vendor patch level
device model
configuration

Determine whether the reported CVE actually corresponds to the firmware being tested.

Possible outcome:

Report says:
affected through 4.2.1
Firmware contains:
4.2.4
→ Not affected by version

But also account for vendor backports.

A firmware package might report an older upstream version while already containing the security patch.

⸻

23. Step 2 — Verify Rehosting Fidelity

Before testing the vulnerability, establish that the firmware works normally in the rehosted environment.

Verify:

firmware boots
   ↓
expected processes start
   ↓
network interfaces work
   ↓
target service starts
   ↓
benign request succeeds

This baseline is essential.

If the relevant service never starts because the rehosting environment is incomplete, failure to reproduce the CVE tells you almost nothing.

⸻

24. Step 3 — Establish a Control

Send a normal input first.

Example:

Benign request
       ↓
target service
       ↓
normal response
       ↓
no crash

Record the expected behavior.

This becomes the baseline against which the vulnerability-triggering condition is compared.

⸻

25. Step 4 — Record the Vulnerability Test

Use PANDA to record execution around the suspected vulnerable condition.

Conceptually:

(qemu) begin_record vuln_test

Exercise the reported condition.

Then:

(qemu) end_record

This captures execution so it can be replayed.

Record/replay is extremely useful because you can analyze the exact same event repeatedly without continuously reproducing the original interaction.

⸻

26. Step 5 — Replay

A conceptual replay invocation is:

panda-system-<arch> -m <same-memory> -replay vuln_test

Replay should use settings compatible with those used during recording.

Now you can repeatedly investigate:

same input
same execution
same event

with different analysis tools.

⸻

27. Step 6 — Verify Code Reachability

One of the most important questions is:

Did execution actually reach the supposedly vulnerable code?

Use coverage and tracing to determine whether the relevant:

* process
* module
* function
* basic block
* instruction

was executed.

Possible result:

Reported request
       ↓
service receives request
       ↓
different parser used
       ↓
vulnerable function never executes

That evidence strongly changes the interpretation of the vulnerability report.

⸻

28. Step 7 — Observe the Vulnerable Condition

If the code is reached, determine whether the claimed failure actually occurs.

Possible evidence includes:

invalid memory access
crash
assertion failure
process termination
unexpected branch
memory corruption
state corruption
unauthorized state transition
abnormal restart

Do not stop simply because the function was reached.

The goal is to establish whether the specific vulnerability condition occurs.

⸻

29. Step 8 — Track Input With Taint Analysis

If the vulnerability depends on externally controlled data, PANDA’s taint-analysis capabilities can be especially valuable.

Conceptually:

External input
      ↓
mark as tainted
      ↓
parser
      ↓
length/value/pointer
      ↓
security-sensitive operation

You want to determine:

Does the reported input actually influence the dangerous operation?

This distinguishes:

Input reaches parser

from:

Input controls vulnerable memory operation

Those are significantly different findings.

⸻

30. Step 9 — Debug the Failure

PANDA’s replay model makes debugging particularly useful.

Instead of repeatedly trying to reproduce:

request → crash

you can replay:

recorded request
     ↓
pause execution
     ↓
inspect registers
     ↓
inspect memory
     ↓
inspect call path
     ↓
step through failure

This can help identify exactly where the vulnerable behavior occurs.

⸻

31. Step 10 — Run Controls

A strong validation test should include multiple conditions.

For example:

Test A
Normal input
→ no failure
Test B
Reported triggering input
→ failure
Test C
Nearby but non-triggering input
→ no failure
Test D
Patched firmware + triggering input
→ no failure

That is much stronger evidence than observing a single unexplained crash.

⸻

32. Strong Confirmation Pattern

One of the strongest validation patterns is:

Vulnerable firmware
+
reported input
        ↓
vulnerable condition occurs
Vulnerable firmware
+
normal input
        ↓
condition does not occur
Patched firmware
+
same reported input
        ↓
condition no longer occurs

This provides evidence connecting:

version
+
input
+
code path
+
failure
+
fix

⸻

33. Evidence Chain

For a reported vulnerability, the evidence chain should ideally establish:

Reported input
      ↓
input reaches affected service
      ↓
affected code executes
      ↓
input influences relevant data/state
      ↓
vulnerable condition occurs
      ↓
security consequence is demonstrated or technically supported

PANDA is particularly useful in the middle of this chain:

reachability
     ↓
data propagation
     ↓
execution behavior

⸻

34. Final Vulnerability Classification

Avoid treating every unsuccessful reproduction as a false positive.

A more useful classification system is:

Confirmed

Evidence demonstrates the vulnerable condition in the affected firmware.

trigger
→ code reached
→ vulnerable condition
→ expected consequence

⸻

Not affected

Evidence shows the system does not contain or execute the vulnerable implementation.

Examples:

fixed version
different component
vendor backport
affected function absent

⸻

Mitigated

The vulnerable software may exist, but a control prevents the reported attack path.

Examples could include:

affected service disabled
feature unavailable
access restricted
vendor mitigation applied

This should be documented as mitigation rather than incorrectly labeling the underlying vulnerability nonexistent.

⸻

Not reproduced / inconclusive

The test did not reproduce the vulnerability, but the analysis environment cannot rule it out.

Example:

firmware requires hardware peripheral
       ↓
rehost cannot fully model peripheral
       ↓
affected path never becomes reachable

This is not sufficient evidence for a false positive.

⸻

Likely false positive

This classification should require evidence.

Examples:

scanner matched incorrect version
vendor backport verified
affected component absent
reported function does not exist
claimed vulnerable path provably unreachable
scanner identified wrong product

⸻

35. Why Rehosting Failure Does Not Automatically Mean False Positive

Firmware often depends on:

* hardware registers
* custom peripherals
* device-specific drivers
* timing
* NVRAM
* secure elements
* hardware initialization
* watchdogs
* physical I/O
* proprietary kernel modules

If those dependencies are not modeled sufficiently, the rehosted environment may behave differently from the physical device.

Therefore:

Cannot reproduce in PANDA
            ≠
CVE is false

Instead, ask:

Did the affected code actually execute?
Was the required system state reproduced?
Were the necessary peripherals represented?
Was the same configuration used?
Was the service fully functional?

⸻

36. Recommended Tool Chain

For general firmware vulnerability analysis:

Firmware
   ↓
Penguin + IGLOO
   ↓
PANDA / QEMU
   ↓
firmware boots
   ↓
Nmap
   ↓
identify exposed services
   ↓
appropriate testing tool

For binary/parser investigation:

AFL++
   ↓
interesting behavior
   ↓
PANDA
   ↓
record + replay + trace + analyze

For network protocols:

boofuzz
   ↓
protocol input
   ↓
firmware
   ↓
PANDA analysis

For a web administration interface:

Burp / ZAP
   ↓
HTTP/API request
   ↓
firmware web service
   ↓
PANDA

For an already reported CVE:

CVE / vendor advisory
        ↓
determine affected component/version
        ↓
reproduce reported condition
        ↓
PANDA record
        ↓
PANDA replay
        ↓
coverage / tracing
        ↓
taint if appropriate
        ↓
debugging
        ↓
controls
        ↓
classification

⸻

37. Recommended Proxmox Deployment

For this lab, a clean deployment is:

Physical Server
      ↓
Proxmox VE
      ↓
Dedicated Ubuntu 24.04 VM
      ↓
8 vCPU
16 GB RAM
120 GB SSD
nested virtualization
      ↓
Docker
      ↓
Penguin + IGLOO
      ↓
PANDA / QEMU
      ↓
Firmware

Keep firmware analysis inside the dedicated VM instead of installing the research stack directly onto the Proxmox host.

Benefits include:

* isolation
* easier snapshots
* easier rollback
* clean dependency management
* disposable/rebuildable analysis environment
* reduced impact on the hypervisor

⸻

38. Recommended Snapshot Strategy

Because Proxmox hosts the analysis VM, use snapshots at major milestones.

Example:

Snapshot 1
Clean Ubuntu
Snapshot 2
Docker + dependencies installed
Snapshot 3
Penguin/IGLOO/PANDA working
Snapshot 4
Target firmware configured
Snapshot 5
Known-good boot state

If an experiment breaks the environment:

rollback
   ↓
known-good analysis environment

This is one major advantage of putting the toolchain inside a Proxmox VM.

⸻

39. Suggested Workflow for Each Reported Vulnerability

For every vulnerability report, create a small validation record containing:

CVE / finding ID:
Device:
Firmware version:
Affected component:
Reported affected versions:
Reported prerequisite:
Reported trigger:
Expected behavior:
Expected vulnerable behavior:
Rehost status:
Affected service reachable:
Affected code reached:
Input reaches vulnerable operation:
Observed result:
Patched version comparison:
Final classification:
Supporting evidence:
Limitations:

This makes vulnerability validation repeatable and auditable.

⸻

40. Example Decision Flow

Reported CVE
     │
     ▼
Does product/version match?
 ┌───┴────┐
 No       Yes
 │         │
 ▼         ▼
Not      Is affected
affected component present?
         │
     ┌───┴────┐
     No       Yes
     │         │
     ▼         ▼
   Not       Can affected
 affected    path execute?
              │
         ┌────┴────┐
         No        Yes
         │          │
         ▼          ▼
   investigate    Trigger test
   why unreachable      │
                        ▼
                Condition occurs?
                   │          │
                  No         Yes
                   │          │
                   ▼          ▼
             Controls /     Confirmed
             fidelity check
                   │
                   ▼
         Not affected / mitigated /
         inconclusive / likely false
         positive depending on evidence

⸻

41. Core Principle

The objective is not simply:

“Does a scanner list this CVE?”

The objective is to establish:

Does this exact firmware contain the affected implementation?
Can the relevant code execute?
Can the reported input reach it?
Does that input influence the vulnerable operation?
Does the vulnerable condition actually occur?
Does a fixed version eliminate the condition?

That turns a vulnerability report into evidence-backed validation.

⸻

42. Summary

The lab can be understood as four layers:

Layer 1
Proxmox
→ virtualization and isolation
Layer 2
Ubuntu analysis VM
→ operating environment
Layer 3
Penguin + IGLOO + PANDA/QEMU
→ firmware rehosting and execution analysis
Layer 4
Nmap / AFL++ / boofuzz / Burp / ZAP / PANDA plugins
→ vulnerability discovery, reproduction, and validation

And for a reported CVE:

CVE information
      ↓
version applicability
      ↓
firmware rehosting
      ↓
baseline verification
      ↓
reported trigger
      ↓
PANDA record/replay
      ↓
reachability
      ↓
taint / tracing / debugging
      ↓
control tests
      ↓
evidence
      ↓
Confirmed
Not affected
Mitigated
Not reproduced / inconclusive
or evidence-supported false positive

The central idea is:

PANDA is not primarily there to tell you that a CVE exists. It helps you establish what the firmware actually did when the reported vulnerability condition was exercised.
