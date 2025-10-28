
# **An In-Depth Analysis of Hyper-V-VmSwitch Event ID 285: Causes, Impacts, and Resolution Strategies**

## **I. Executive Summary**

This report provides an expert-level analysis of Hyper-V-VmSwitch Event ID 285, a warning that signifies a critical timeout within the Hyper-V virtual network stack. The event, particularly when referencing UNSPECIFIED\_OPERATION\_CODE (4) with an Operation Type: RNDIS, points to a failure in the communication protocol between a guest virtual machine and the host's virtual switch. While logged as a warning, this event often presages significant performance degradation and can be an indicator of deeper system instability.

The investigation reveals that the root cause is rarely a single misconfiguration but rather a complex interplay of factors. The most common source is **network interface card (NIC) driver and firmware incompatibilities** with advanced Hyper-V networking features, primarily Virtual Machine Queue (VMQ) and Switch Embedded Teaming (SET). The issue is prevalent with modern high-speed NICs from vendors like Intel (E610/E810 series) and has a long, well-documented history with Broadcom adapters.

Key findings indicate that Event ID 285 is frequently correlated with Event ID 280, which signals a failure in negotiating vPort Queue Pairs between the hypervisor and the physical NIC driver. This correlation provides a crucial diagnostic clue, pointing directly to a driver-level fault. The impact of these events ranges from severe network performance degradation and VM sluggishness to host instability, including authentication failures that necessitate reboots. Historically, the underlying RNDIS protocol has also been associated with critical security vulnerabilities, elevating the importance of resolving these warnings beyond mere performance tuning.

The primary resolution path involves a meticulous process of updating NIC drivers and firmware to versions specifically certified for the host's Windows Server operating system. As an immediate workaround, disabling VMQ on the physical NIC or within the affected VM's network adapter settings is often effective at restoring stability. For long-term resilience, a strategic approach to network configuration, proactive driver management, and careful hardware selection is essential for any enterprise Hyper-V deployment.

## **II. Deconstructing Hyper-V-VmSwitch Event ID 285**

To effectively address Event ID 285, a thorough understanding of the information contained within the event log entry is paramount. Each component provides a piece of the diagnostic puzzle, revealing the nature and origin of the failure within the Hyper-V networking stack.

### **Anatomy of the Warning**

The event log entry, as seen in the provided image, contains several key fields that define the issue:

* **Log Name:** Microsoft-Windows-Hyper-V-VmSwitch/Operational. This specifies that the event is recorded in the operational log for the Hyper-V virtual switch, a critical component for all virtual machine networking.1 This log tracks the day-to-day activities and health of the vSwitch.  
* **Source:** Hyper-V-VmSwitch. The event originates directly from the virtual switch driver, vmswitch.sys, which runs in the host operating system and manages network traffic for all VMs.3  
* **Event ID:** 285\. This is the unique identifier assigned by Microsoft to the specific condition where a virtual switch operation "took too long to complete."  
* **Level:** Warning. This severity level indicates a non-critical but significant issue. It signifies that a function did not complete as expected, which could lead to performance degradation or escalate to more severe errors if the underlying cause is not addressed.  
* **Message:** V-Switch operation UNSPECIFIED\_OPERATION\_CODE (4) took too long to complete. Operation Type: RNDIS. This is the core diagnostic message. It states that the virtual switch initiated an operation using the RNDIS protocol, but the operation failed to complete within its allotted timeframe.  
* **Execution Details:** Execution time 0 ms. Queued time 0 ms. Expected execution time less than 0 ms. The timing values in this specific instance can be misleading. An "expected execution time less than 0 ms" often points to an immediate timeout or an internal logic error within the driver stack, rather than a measured delay that exceeded a threshold.4

### **The Critical Role of RNDIS (Remote Network Driver Interface Specification)**

The Operation Type: RNDIS is a vital clue. RNDIS is a Microsoft-proprietary protocol that defines a message-based communication channel between a host and a network device over a generic bus. In the context of Hyper-V, the guest VM acts as the "host," and the vmswitch.sys driver on the physical host acts as the "device," with communication occurring over Hyper-V's high-speed VMBus.3

The guest VM's network driver (netvsc.sys) uses RNDIS to send control messages—not data packets—to the host's virtual switch provider (vmswitch.sys).3 These messages manage the configuration of the virtual network adapter, such as setting hardware filters, updating offload capabilities, or querying status. A timeout during an RNDIS operation, as logged in Event 285, signifies a breakdown in this crucial control plane communication. This failure disrupts the ability of the guest and host to properly manage the virtual network connection, which can lead to a misconfigured or unstable state.

### **Analysis of Other Common Operation Types for Event 285**

While the user's image shows an RNDIS timeout, Event 285 is a generic warning that can be triggered by other types of operations, each pointing to a different facet of the same underlying problem.

* **OID (Object Identifier) Requests:** OIDs are a fundamental part of the Windows Network Driver Interface Specification (NDIS), used by drivers to query or set parameters for network adapters. Event 285 frequently appears with OID timeouts, such as OID\_GEN\_STATISTICS or OID\_RECEIVE\_FILTER\_MOVE\_FILTER.5  
  * An OID\_GEN\_STATISTICS timeout indicates the physical NIC driver failed to respond to a request for performance statistics from the vSwitch in a timely manner.6  
  * An OID\_RECEIVE\_FILTER\_MOVE\_FILTER timeout suggests the driver is struggling to reconfigure hardware filters used for technologies like VMQ.5  
    These timeouts demonstrate that the physical NIC driver is becoming unresponsive to standard management requests from the Hyper-V stack.7  
* **IOCTL (I/O Control) Requests:** In some reported cases, Event 285 is triggered by an IOCTL timeout, such as IOCTL\_ENUM\_NIC\_COUNT\_EX.4 IOCTLs are a lower-level mechanism for applications and drivers to communicate directly with a device driver. A timeout at this level suggests a more fundamental communication breakdown between the Hyper-V services and the network driver stack.

The variety of operation types (RNDIS, OID, IOCTL) that can trigger Event 285 is highly significant. It demonstrates that the problem is not a flaw within a single protocol like RNDIS, but rather a symptom of systemic stress or incompatibility within the vmswitch.sys driver and, more specifically, its interaction with the underlying physical NIC driver. The specific operation that times out acts as a clue to where the stress is most pronounced. An RNDIS timeout points to the guest-to-host control plane, while an OID timeout points to the host's management of the physical NIC. When different operations time out under similar conditions (e.g., on hosts with specific high-speed Intel NICs), it strongly implies the root cause is a NIC driver that is becoming unresponsive or deadlocked, causing whichever operation is currently in flight to fail.

## **III. Core Technologies and Interdependencies**

Event ID 285 does not occur in a vacuum. It is almost always a consequence of interactions between the Hyper-V Virtual Switch and advanced networking features designed to enhance performance. Understanding these technologies is essential to diagnosing the root cause.

### **The Hyper-V Virtual Switch (VmSwitch) Architecture**

The Hyper-V Virtual Switch is a software-based Layer 2 Ethernet switch that provides the foundational connectivity for virtual machines.8 It is implemented as a driver (vmswitch.sys) in the management operating system. The architecture is extensible, allowing third-party NDIS filter drivers to be inserted into the stack to provide services like monitoring, firewalling, or forwarding.3 The communication path for a VM involves its network driver (netvsc.sys) sending traffic and control messages over the VMBus to the host's vmswitch.sys driver. The vSwitch then processes this traffic and passes it to the physical NIC's NDIS miniport driver for transmission on the physical network.3 A failure or delay at any point in this software and hardware chain can result in the timeouts logged as Event 285\.

### **Virtual Machine Queue (VMQ): A Double-Edged Sword**

Virtual Machine Queue (VMQ) is a hardware acceleration feature that significantly improves network performance in virtualized environments. Its primary function is to offload the task of sorting incoming network packets to the physical NIC. The NIC can then use Direct Memory Access (DMA) to place packets directly into the memory buffers of their intended destination VM, bypassing the Hyper-V host's processing stack. This reduces CPU overhead on the host and lowers network latency for the VM.10

However, this performance benefit comes at the cost of complexity. VMQ requires a robust and well-written NIC driver that can flawlessly integrate with the Hyper-V vSwitch. Historically, faulty VMQ implementations have been a notorious source of network problems, especially with Broadcom NICs.11 When a NIC driver's VMQ implementation is buggy or incompatible with the version of Hyper-V being used, it can lead to packet loss, high latency, and general network sluggishness. These performance issues create conditions where control operations can easily time out, manifesting as Event 285\. For this reason, disabling VMQ is one of the most common and effective workarounds for a wide range of unexplained Hyper-V network performance issues.13

### **Switch Embedded Teaming (SET): Exposing Latent Driver Flaws**

Switch Embedded Teaming (SET) is a NIC Teaming solution integrated directly into the Hyper-V Virtual Switch. It allows administrators to group between one and eight physical Ethernet adapters into a single software-based team, providing fault tolerance and bandwidth aggregation. As an alternative to traditional OS-level teaming, SET is designed to be more efficient and compatible with all Hyper-V networking features, including VMQ and SR-IOV.17

While powerful, SET places additional demands on the NIC drivers of the team members. The drivers must correctly handle the distribution of traffic and the allocation of hardware resources across multiple physical ports. This complexity often exposes latent bugs or design limitations in how a driver manages resources, particularly the number of available "queue pairs" (dedicated hardware channels for Rx/Tx traffic) for each virtual port (vPort). This precise issue is the direct cause of the related Event ID 280, where Hyper-V's request for a certain number of queue pairs is rejected by the driver.5

The progression from basic networking to advanced features like VMQ and SET represents a fundamental trade-off between performance and stability. Event 285 often arises when this trade-off fails—when the complexity introduced by a performance-enhancing feature exceeds the driver's ability to handle it reliably. A clear causal chain often emerges in modern deployments: an administrator enables SET on a team of high-speed Intel NICs for best-practice redundancy. This combination of SET and the driver's VMQ implementation creates a conflict over how many queue pairs can be allocated to each VM. This conflict triggers Event 280 as the driver forces a lower-than-requested number of queues. This misconfigured or contested hardware state leaves the driver struggling to manage network operations efficiently, causing subsequent control plane requests (like RNDIS or OID) to time out, thus triggering Event 285\.

## **IV. Symptomatology and System Impact**

Event ID 285 is more than a benign entry in an event log; it is a symptom of an underlying condition that can have wide-ranging and severe impacts on the performance, stability, and even security of a Hyper-V host and its guest virtual machines.

### **Network Performance Degradation**

The most immediate and common consequence of the conditions that cause Event 285 is a noticeable degradation in network performance for guest VMs. The timeout itself is a symptom of a bottleneck or instability in the vSwitch control plane, which invariably affects the data plane. This can manifest in several ways:

* **High Latency:** Increased ping times and slow application response.  
* **Low Throughput:** Slow file transfers and reduced bandwidth for network-intensive applications.12  
* **General Sluggishness:** VMs may feel unresponsive, particularly during network-heavy operations.19

### **Host and Guest Instability**

In more severe cases, the instability within the vmswitch.sys driver can cascade, leading to broader system-level problems. The network stack is a foundational component of the operating system, and its failure can have far-reaching effects.

* **Host Authentication Failures:** One user report detailed a scenario where the onset of VmSwitch events was consistently followed by Netlogon errors on the Hyper-V host. This prevented administrators from logging in to the server, either locally or remotely, forcing a hard reboot to restore access.20 This indicates the network instability can become severe enough to disrupt core OS services like domain authentication.  
* **VM Disconnections and Failures:** Other reports describe VMs randomly disconnecting from the network, rebooting unexpectedly, or encountering "generic failure" errors that require the entire virtual switch to be deleted and recreated to restore connectivity.21

### **Security Implications of RNDIS Vulnerabilities**

While Event 285 is logged as a performance warning, the underlying RNDIS technology has been the subject of critical security research. Flaws in how vmswitch.sys parses and handles RNDIS packets sent from a guest VM can be exploited to compromise the host itself.

* **Denial of Service (DoS):** Research has shown that malformed RNDIS packets could trigger an Out-of-Bounds Read (OOBR) in the vmswitch.sys driver, leading to a host crash (a BugCheck, or "blue screen of death").24  
* **Remote Code Execution (RCE):** A critical vulnerability, **CVE-2021-28476**, with a CVSS score of 9.9, was discovered in vmswitch.sys. This flaw allowed an attacker within a guest VM to send a specially crafted packet to the Hyper-V host, potentially leading to a DoS or achieving arbitrary code execution on the host machine. This vulnerability affected multiple versions of Windows and Windows Server and was present in production for over a year and a half before being patched.3

This history of vulnerabilities means Event ID 285 should be treated not just as a performance indicator, but as a potential indicator of a compromised or vulnerable attack surface. An unstable driver that fails to process legitimate requests in a timely manner may also be more susceptible to failing to properly validate malicious or malformed requests. The conditions that cause performance timeouts—such as driver bugs and firmware issues—create a non-standard, unstable operating environment for the network stack. It is plausible that this unstable state could lower the barrier for an attacker to trigger a known (but unpatched) or even an unknown vulnerability in the same RNDIS processing code path. Therefore, resolving Event 285 is not merely a performance tuning task; it is a critical step in **hardening the hypervisor host** by ensuring the network stack operates in a stable, predictable, and secure state.

## **V. Analysis of Correlated Events: The Link to Event ID 280**

In many troubleshooting scenarios, Event ID 285 does not appear in isolation. Its diagnostic value is magnified significantly when it appears alongside Event ID 280 in the Hyper-V-VmSwitch log. This correlation provides a clear narrative of cause and effect.

### **Decoding Event ID 280**

The message associated with Event ID 280 is explicit and highly informative:

VMS Utilization Plan Vport QueuePairs adjusted from requested number (X) to actual (Y). Reason: The number requested exceeds the maximum QPs per VPort supported by physical NIC. 6

This event logs a failed negotiation for hardware resources. It means that the Hyper-V hypervisor, as part of its resource allocation plan, requested a specific number of queue pairs (e.g., 16\) for a VM's virtual network adapter. However, the physical NIC's driver rejected this request and forced a lower number (e.g., 4 or 8), explicitly citing the NIC's supported maximum as the reason.

### **Diagnostic Synergy**

When Event 280 and 285 appear together, they tell a story. Event 280 is the *cause*—the initial, specific failure to configure the hardware as requested by the hypervisor. Event 285 is the subsequent *effect*—the generic timeouts and instability that result from the VM operating in this misconfigured or resource-constrained state.

This pattern is heavily documented in community forums discussing modern Intel E610-XT4 and E810-XXV NICs, particularly when used in a Switch Embedded Teaming (SET) configuration on Windows Server 2022 and 2025\.5 In these cases, administrators observe that when a VM is started, Hyper-V attempts to allocate 16 queue pairs to its vPort. The Intel driver rejects this and forces it down to 4, logging Event 280\. Shortly thereafter, as the VM begins passing network traffic, Event 285 warnings begin to populate the log, indicating that operations like RNDIS or OID requests are timing out.

The presence of Event 280 definitively shifts the focus of the investigation. It moves the likely culprit from a generic Hyper-V issue or a problem within the guest OS to a specific **NIC driver or firmware incompatibility**. It provides the "smoking gun" that proves the physical hardware layer is not complying with the hypervisor's resource requests. For an administrator who starts by seeing a vague Event 285 timeout, finding the corresponding Event 280 is a critical breakthrough. It allows them to rule out other potential causes and focus all troubleshooting efforts on the host's physical NICs, their drivers, firmware, and advanced settings. This transforms a broad performance problem into a targeted hardware and driver investigation.

## **VI. Vendor-Specific Analysis: Intel and Broadcom Case Studies**

The manifestation of VmSwitch issues, including Event 285, is often specific to the NIC vendor and even the particular model and driver version. Two vendors, Intel and Broadcom, are frequently discussed in relation to these problems.

### **Intel NICs (E810/E610 Series): The Modern Challenge**

With the rise of 25GbE, 40GbE, and 100GbE networking in virtualized environments, modern Intel adapters from the Ethernet 800 and 600 series are common choices. However, these advanced NICs have presented new challenges.

* **Problem:** Users deploying Intel E810-XXV and E610-XT4 adapters are consistently reporting the combination of Event 280 and 285, especially on Windows Server 2022 and the forthcoming Windows Server 2025 when using Switch Embedded Teaming (SET).5  
* **Root Cause Analysis:** The evidence strongly points to a driver-level policy or bug, rather than a hardware limitation. The datasheets for these NICs indicate that the hardware supports a high number of receive queues (e.g., 128), which should allow for more than the 4 queue pairs per vPort to which they are being limited.5 The problematic behavior is that the driver initially advertises a high capability (16 queue pairs) to Hyper-V, but then downgrades this to 4 at runtime when the VM is activated. This runtime change triggers Event 280 and creates the unstable condition that leads to Event 285\. An Intel support representative's statement that this is "expected behavior" has been challenged by users, as it contradicts both the hardware's capabilities and the driver's initial advertisement, suggesting it may be a hard-coded workaround for a deeper issue within the driver's interaction with SET.5  
* **Status:** As of the latest available community information, there is no official public fix from Intel for this specific queue pair limitation. The standing recommendation is to continuously monitor for new driver and NVM firmware releases and to open formal support tickets with both Intel and Microsoft to provide diagnostic data and press for a resolution.6

### **Broadcom NICs: A History of VMQ Issues**

Broadcom NICs have a long and well-documented history of causing severe network performance problems in Hyper-V environments, almost always tied to their implementation of Virtual Machine Queue (VMQ).

* **Problem:** For many years, administrators have reported that enabling VMQ on Broadcom NICs in a Hyper-V host leads to drastically reduced network throughput, high latency, and sluggish VM performance.11 These conditions are ripe for producing Event 285 timeouts.  
* **Root Cause Analysis:** The issue lies in flawed VMQ implementations in numerous older Broadcom driver versions. The problem was so widespread that Microsoft released a Knowledge Base article acknowledging it and recommending disabling the feature. The standard resolution for years has been to either perform a significant driver update or, more reliably, to disable VMQ entirely in the advanced properties of the physical network adapter.13  
* **Status:** While newer driver versions have reportedly corrected the most severe issues, many administrators still find that network performance and stability are better with VMQ disabled on Broadcom NICs.13 The default advice for any Hyper-V host equipped with Broadcom NICs remains consistent: first, ensure drivers are updated from the server OEM, and if any unexplained network issues persist, disable VMQ.

The following table provides a summary for quick reference.

| Vendor | Common Affected Models | Primary Associated Event(s) | Typical Root Cause | Primary Recommended Action |
| :---- | :---- | :---- | :---- | :---- |
| **Intel** | E810-XXV, E610-XT4 | Event 280 & 285 | Driver/firmware incompatibility with SET and vPort Queue Pair allocation. | Aggressively update drivers/firmware to the latest versions from the vendor; open support tickets with Intel and Microsoft. |
| **Broadcom** | NetXtreme Gigabit/10GbE series | Severe performance degradation that can lead to Event 285\. | Flawed or incompatible Virtual Machine Queue (VMQ) implementation in drivers. | Update drivers from the server OEM; if issues persist, disable VMQ on the physical NICs as a permanent solution. |

## **VII. Comprehensive Troubleshooting and Mitigation Strategies**

Resolving Event ID 285 requires a structured, methodical approach that moves from initial diagnosis to immediate stabilization and finally to a permanent resolution. The following framework outlines this process.

### **Phase 1: Diagnostics and Identification**

Before making any changes, it is crucial to gather all relevant data to accurately identify the scope and context of the problem.

1. **Event Log Correlation:** Using Windows Event Viewer on the Hyper-V host, navigate to Applications and Services Logs \> Microsoft \> Windows \> Hyper-V-VmSwitch. Filter the Operational log for Event ID 285\. When instances are found, expand the time window and look for corresponding instances of Event ID 280\.1 Note the timestamps, the specific VM names, and the "Friendly Name" of the NIC or vPort mentioned in the event details.  
2. **Hardware and Driver Verification:** Identify the exact model of the physical NICs involved and their current driver versions. This can be done via PowerShell:  
   PowerShell  
   Get-NetAdapter \-Name "\*" | Format-Table Name, InterfaceDescription, DriverVersion, DriverDate

3. **Firmware Verification:** The NIC's firmware version is as critical as its driver. This information can typically be found using vendor-specific utilities (e.g., Intel PROSet, Broadcom Advanced Control Suite) or through the server's out-of-band management interface (e.g., Dell iDRAC, HPE iLO).  
4. **OS and Integration Services Verification:** Confirm that the host operating system has all the latest Windows Updates installed. Inside each affected guest VM, ensure that the Hyper-V Integration Services are up to date and running. For modern Windows guests, this is handled by Windows Update, but for Linux or older Windows versions, it may require manual verification or installation.27

### **Phase 2: Immediate Workarounds and Stabilization**

If the issue is causing a significant operational impact, immediate workarounds can be employed to restore service while a permanent fix is pursued.

1. **Action 1: Disable VMQ (Most Common Fix):** This is the most frequently successful immediate workaround, as it removes the complexity of the hardware offload feature that is often the source of the driver conflict.  
   * **On the VM's vNIC (Least Disruptive):** In the VM's settings in Hyper-V Manager, navigate to the network adapter, expand the Hardware Acceleration section, and uncheck "Enable Virtual Machine Queue".11 This can often be done while the VM is running. Alternatively, use PowerShell:  
     PowerShell  
     Set-VMNetworkAdapter \-VMName "\<VM Name\>" \-VmqWeight 0

     15  
   * **On the Physical NIC (More Impactful):** In Device Manager on the host, find the physical network adapter, open its Properties, go to the Advanced tab, find the "Virtual Machine Queue" property, and set its value to Disabled.13 This will cause a brief network interruption for all VMs using that adapter. Via PowerShell:  
     PowerShell  
     Disable-NetAdapterVmq \-Name "\<NIC Name\>"

     29  
2. **Action 2: Recreate the Virtual Switch:** A vSwitch configuration can become corrupted. Deleting and recreating the vSwitch (and the associated SET team, if applicable) can resolve underlying state issues.20 This will cause a temporary but complete network outage for the host and all associated VMs and should be performed during a maintenance window.  
3. **Action 3: System File and Component Store Repair:** If broader OS instability is suspected, running system file checks on the host can resolve corruption that may be affecting Hyper-V services. Open an administrative Command Prompt or PowerShell and run:  
   DOS  
   sfc /scannow  
   DISM /Online /Cleanup-Image /RestoreHealth

   22

### **Phase 3: Long-Term Resolution**

Workarounds address symptoms; the following steps address the root cause.

1. **Action 1: Authoritative Driver and Firmware Updates:** This is the most critical step for a permanent solution. Do not rely on drivers provided by Windows Update for server-grade NICs.  
   * Go to the support website for your server manufacturer (e.g., Dell, HPE, Lenovo).  
   * Find the support page for your specific server model and select your operating system (e.g., Windows Server 2022).  
   * Download the latest validated network driver and firmware update packages. These versions have been tested by the OEM for compatibility.  
   * Carefully read the release notes for any mention of fixes related to Hyper-V, SET, VMQ, or network performance.6  
   * Apply the updates during a planned maintenance window, as they will require a host reboot.  
2. **Action 2: Advanced NIC Property Tuning:** If updated drivers do not resolve the issue, further tuning may be required.  
   * **RSS (Receive Side Scaling):** If VMQ is disabled, ensure RSS is enabled on the physical NICs. RSS is a software-based technology that distributes the network processing load across multiple CPU cores, providing an alternative to VMQ's hardware offload.31  
   * **Other Offloads:** As a troubleshooting measure, experiment with disabling other offload features, such as "Large Send Offload (LSO)" or various checksum offloads, in the NIC's advanced properties. Note that disabling these will increase CPU utilization on the host.33  
3. **Action 3: Engage Vendor Support:** For persistent issues, especially with modern NICs experiencing the Event 280/285 pattern, the problem is almost certainly a vendor-level bug that administrators cannot fix on their own. Open a support case with both the NIC vendor (e.g., Intel) and Microsoft. Provide comprehensive diagnostic information, including event logs, driver/firmware versions, and a detailed description of the environment.6

The table below summarizes this framework.

| Priority/Phase | Action | Implementation Method (GUI & PowerShell) | Impact/Considerations |
| :---- | :---- | :---- | :---- |
| **1\. Diagnostics** | Correlate Event Logs | Event Viewer: Filter Hyper-V-VmSwitch log for IDs 280, 285\. | No service impact. Establishes the link between hardware negotiation failure and timeouts. |
| **1\. Diagnostics** | Verify Driver/Firmware | Device Manager / OEM Tools. Get-NetAdapter | ft Name, DriverVersion | No service impact. Identifies if components are out of date. |
| **2\. Workaround** | Disable VMQ on vNIC | VM Settings \> Network Adapter \> Hardware Acceleration. Set-VMNetworkAdapter \-VMName "X" \-VmqWeight 0 | Low impact; can be done live in many cases. A brief network "blip" for the VM may occur. |
| **2\. Workaround** | Disable VMQ on Physical NIC | Device Manager \> NIC Properties \> Advanced. Disable-NetAdapterVmq \-Name "X" | Medium impact; causes a brief network interruption for all VMs on the adapter. |
| **2\. Workaround** | Recreate Virtual Switch | Hyper-V Manager \> Virtual Switch Manager. Remove-VMSwitch, New-VMSwitch | High impact; requires a maintenance window. Causes network outage for the host and VMs. |
| **3\. Long-Term Fix** | Update NIC Driver & Firmware | Run installer packages downloaded from server OEM/vendor website. | High impact; requires host reboot and a planned maintenance window. This is the most likely permanent fix. |
| **3\. Long-Term Fix** | Engage Vendor Support | Submit tickets via vendor support portals with detailed logs. | No direct service impact, but requires administrative time. Essential for un-resolvable bugs. |

## **VIII. Strategic Recommendations for Infrastructure Resilience**

Resolving an active incident is reactive; building an infrastructure that is resilient to such issues is proactive. The following strategic recommendations are designed to minimize the likelihood of encountering Event 285 and similar network-related problems in Hyper-V environments.

### **Hardware Selection and Validation**

The foundation of a stable virtualized environment is compatible and validated hardware.

* **Adhere to the Hardware Compatibility List (HCL):** Before purchasing servers or network adapters, consult the server vendor's official HCL for the specific operating system version you intend to deploy (e.g., Windows Server 2022). Prioritizing components that are explicitly tested and validated by the OEM significantly reduces the risk of incompatibility.17  
* **Conduct Pilot Testing:** For any new server model, NIC, or major OS version, establish a pilot or pre-production environment. Deploy a representative workload and run it for an extended period to uncover any latent driver, firmware, or hardware issues before they can impact the production fleet.

### **Proactive Driver and Firmware Management**

Network adapter drivers and firmware are not "set it and forget it" components. They are complex pieces of software that require active lifecycle management.

* **Establish a Regular Update Cadence:** Institute a formal policy to review and apply certified driver and firmware updates during scheduled maintenance windows. A quarterly or semi-annual review is a reasonable starting point.34  
* **Source from the OEM:** Always prioritize driver and firmware packages provided by the server manufacturer (OEM). These packages have been tested and validated for your specific hardware configuration. Avoid using generic drivers from the NIC vendor or those delivered via Windows Update for server-grade hardware, as they may lack OEM-specific customizations and validation.

### **Configuration Best Practices**

A thoughtful approach to network configuration can prevent many common problems.

* **Incremental Feature Enablement:** When deploying new hosts, begin with a minimal set of advanced networking features. Establish a stable performance baseline with basic connectivity first. Then, incrementally enable features like VMQ or configure SET teams, monitoring for any signs of instability after each change.  
* **NUMA-Aware Configuration:** In multi-socket servers, be mindful of Non-Uniform Memory Access (NUMA) topology. When configuring advanced features like VMQ or RSS, use PowerShell to set the processor affinity so that a NIC's network processing is handled by CPU cores on the same NUMA node as the NIC itself. This prevents performance-degrading cross-NUMA node memory access.17  
* **Traffic Isolation:** Whenever possible, use dedicated physical network adapters for different types of traffic. A common best practice is to separate Management, Live Migration, Storage (iSCSI/SMB), and VM traffic onto different physical NICs or teams. This isolation prevents contention, improves security, and contains the impact of a potential issue (like a driver failure) to a single traffic type.35

#### **Works cited**

1. Troubleshooting Hyper-V with Event Logs \- BDRShield, accessed October 15, 2025, [https://www.bdrshield.com/blog/hyper-v-event-logs-troubleshooting/](https://www.bdrshield.com/blog/hyper-v-event-logs-troubleshooting/)  
2. Hyper-V: my custom virtual switch disappears from time to time \- Super User, accessed October 15, 2025, [https://superuser.com/questions/1770564/hyper-v-my-custom-virtual-switch-disappears-from-time-to-time](https://superuser.com/questions/1770564/hyper-v-my-custom-virtual-switch-disappears-from-time-to-time)  
3. Critical 9.9 Vulnerability in Hyper-V Allowed Attackers to Exploit Azure | Akamai, accessed October 15, 2025, [https://www.akamai.com/blog/security/critical-vulnerability-in-hyper-v-allowed-attackers-to-exploit-azure](https://www.akamai.com/blog/security/critical-vulnerability-in-hyper-v-allowed-attackers-to-exploit-azure)  
4. V-Switch operation IOCTL\_ENUM\_NIC\_COUNT\_EX (2241616) took ..., accessed October 15, 2025, [https://www.reddit.com/r/HyperV/comments/147q2s2/vswitch\_operation\_ioctl\_enum\_nic\_count\_ex\_2241616/](https://www.reddit.com/r/HyperV/comments/147q2s2/vswitch_operation_ioctl_enum_nic_count_ex_2241616/)  
5. E610-XT4 Hyper-V 2025: Queue Pairs limited to 4 per vPort — Event 280/285, accessed October 15, 2025, [https://community.intel.com/t5/Ethernet-Products/E610-XT4-Hyper-V-2025-Queue-Pairs-limited-to-4-per-vPort-Event/td-p/1700370](https://community.intel.com/t5/Ethernet-Products/E610-XT4-Hyper-V-2025-Queue-Pairs-limited-to-4-per-vPort-Event/td-p/1700370)  
6. Hyper-V 2025 \+ SET Team: Intel E610-XT4 limited to 4 Queue Pairs ..., accessed October 15, 2025, [https://learn.microsoft.com/en-us/answers/questions/2288350/hyper-v-2025-set-team-intel-e610-xt4-limited-to-4](https://learn.microsoft.com/en-us/answers/questions/2288350/hyper-v-2025-set-team-intel-e610-xt4-limited-to-4)  
7. Receiving Hyper-V Extensible Switch config change OID requests ..., accessed October 15, 2025, [https://learn.microsoft.com/en-us/windows-hardware/drivers/network/receiving-oid-requests-about-hyper-v-extensible-switch-configuration-changes](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/receiving-oid-requests-about-hyper-v-extensible-switch-configuration-changes)  
8. Plan for Hyper-V networking in Windows Server \- Microsoft Learn, accessed October 15, 2025, [https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/plan/plan-hyper-v-networking-in-windows-server](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/plan/plan-hyper-v-networking-in-windows-server)  
9. How to configure Hyper-V virtual switch \- Veeam, accessed October 15, 2025, [https://www.veeam.com/blog/how-to-configure-hyper-v-virtual-switch.html](https://www.veeam.com/blog/how-to-configure-hyper-v-virtual-switch.html)  
10. Broadcom Ethernet NIC Controller Configuration Guide, accessed October 15, 2025, [https://cdn-reichelt.de/documents/datenblatt/E910/BROAD\_BCM5719-4P\_MAN-EN.pdf](https://cdn-reichelt.de/documents/datenblatt/E910/BROAD_BCM5719-4P_MAN-EN.pdf)  
11. Virtual Machines Slow and Sluggish – Broadcom Network Adapter VMQ Issue \- TechTripp, accessed October 15, 2025, [https://techtripp.com/trend/virtual-machines-slow-and-sluggish-broadcom-network-adapter-vmq-issue/](https://techtripp.com/trend/virtual-machines-slow-and-sluggish-broadcom-network-adapter-vmq-issue/)  
12. After 2 years, I have finally solved my "Slow Hyper-V Guest Network Performance" issue. I am ecstatic. : r/sysadmin \- Reddit, accessed October 15, 2025, [https://www.reddit.com/r/sysadmin/comments/2k7jn5/after\_2\_years\_i\_have\_finally\_solved\_my\_slow/](https://www.reddit.com/r/sysadmin/comments/2k7jn5/after_2_years_i_have_finally_solved_my_slow/)  
13. Slow copy performance to a Hyper-V guest on a host with a Broadcom adapter, accessed October 15, 2025, [https://support.softwarepursuits.com/slow-copy-performance-hyper-v-guest-with-a-broadcom-adapter](https://support.softwarepursuits.com/slow-copy-performance-hyper-v-guest-with-a-broadcom-adapter)  
14. Teaming and VMQ issues with broadcom based network cards on Microsoft OS \- Reddit, accessed October 15, 2025, [https://www.reddit.com/r/networking/comments/9t1wuz/teaming\_and\_vmq\_issues\_with\_broadcom\_based/](https://www.reddit.com/r/networking/comments/9t1wuz/teaming_and_vmq_issues_with_broadcom_based/)  
15. Poor network performance on virtual machines \- Windows Server | Microsoft Learn, accessed October 15, 2025, [https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/poor-network-performance-hyper-v-host-vm](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/poor-network-performance-hyper-v-host-vm)  
16. Hyper-V External network switch kills my host's network performance \- Super User, accessed October 15, 2025, [https://superuser.com/questions/1266248/hyper-v-external-network-switch-kills-my-hosts-network-performance](https://superuser.com/questions/1266248/hyper-v-external-network-switch-kills-my-hosts-network-performance)  
17. Windows server 2022 \- Hyper-v cluster issue \- Microsoft Q\&A, accessed October 15, 2025, [https://learn.microsoft.com/en-us/answers/questions/1304813/windows-server-2022-hyper-v-cluster-issue](https://learn.microsoft.com/en-us/answers/questions/1304813/windows-server-2022-hyper-v-cluster-issue)  
18. E610-XT4 Hyper-V 2025: Queue Pairs limited to 4 per vPort — Event ..., accessed October 15, 2025, [https://community.intel.com/t5/Ethernet-Products/E610-XT4-Hyper-V-2025-Queue-Pairs-limited-to-4-per-vPort-Event/m-p/1700370](https://community.intel.com/t5/Ethernet-Products/E610-XT4-Hyper-V-2025-Queue-Pairs-limited-to-4-per-vPort-Event/m-p/1700370)  
19. Windows Server 2019 Hyper-V VM I/O Performance Problem \- Veeam R\&D Forums, accessed October 15, 2025, [https://forums.veeam.com/microsoft-hyper-v-f25/windows-server-2019-hyper-v-vm-i-o-performance-problem-t62112.html](https://forums.veeam.com/microsoft-hyper-v-f25/windows-server-2019-hyper-v-vm-i-o-performance-problem-t62112.html)  
20. Event ID Hyper-V-VMSwitch 270, 24, 23, 265, 266 and Netlogon Event ID 5783, accessed October 15, 2025, [https://learn.microsoft.com/en-us/answers/questions/751219/event-id-hyper-v-vmswitch-270-24-23-265-266-and-ne](https://learn.microsoft.com/en-us/answers/questions/751219/event-id-hyper-v-vmswitch-270-24-23-265-266-and-ne)  
21. Hyper V switch Issues : r/HyperV \- Reddit, accessed October 15, 2025, [https://www.reddit.com/r/HyperV/comments/18ioi5a/hyper\_v\_switch\_issues/](https://www.reddit.com/r/HyperV/comments/18ioi5a/hyper_v_switch_issues/)  
22. Hyper-V Virtual Switch Generic failure \- Super User, accessed October 15, 2025, [https://superuser.com/questions/1816102/hyper-v-virtual-switch-generic-failure](https://superuser.com/questions/1816102/hyper-v-virtual-switch-generic-failure)  
23. Guest VM disconnecting/rebooting randomly : r/HyperV \- Reddit, accessed October 15, 2025, [https://www.reddit.com/r/HyperV/comments/x8azhn/guest\_vm\_disconnectingrebooting\_randomly/](https://www.reddit.com/r/HyperV/comments/x8azhn/guest_vm_disconnectingrebooting_randomly/)  
24. Hyper-V vmswitch.sys VmsPtpIpsecTranslateAddv2toAddv2Ex OOBR Guest to Host BugCheck \[42452211\] \- Project Zero, accessed October 15, 2025, [https://project-zero.issues.chromium.org/42452211](https://project-zero.issues.chromium.org/42452211)  
25. Microsoft Hyper-V Virtual Network Switch VmsMpCommonPvtSetRequestCommon Out of Bounds Read \- Zero Day Engineering, accessed October 15, 2025, [https://zerodayengineering.com/research/hyper-v-vmswitch-oobr.html](https://zerodayengineering.com/research/hyper-v-vmswitch-oobr.html)  
26. Event ID 280 on Windows 2022 Hyper-V Host – Vport QueuePairs of virtual switch, accessed October 15, 2025, [https://community.hpe.com/t5/operating-system-microsoft/event-id-280-on-windows-2022-hyper-v-host-vport-queuepairs-of/td-p/7181382](https://community.hpe.com/t5/operating-system-microsoft/event-id-280-on-windows-2022-hyper-v-host-vport-queuepairs-of/td-p/7181382)  
27. Top 20 Tips on How to Improve VM Performance in Hyper-V \- NAKIVO, accessed October 15, 2025, [https://www.nakivo.com/blog/top-20-tips-improve-vm-performance-hyper-v/](https://www.nakivo.com/blog/top-20-tips-improve-vm-performance-hyper-v/)  
28. Hyper-V with VMQ \- NVIDIA Docs, accessed October 15, 2025, [https://docs.nvidia.com/networking/display/winofv55053000/hyper-v+with+vmq](https://docs.nvidia.com/networking/display/winofv55053000/hyper-v+with+vmq)  
29. Disable-NetAdapterVmq (NetAdapter) | Microsoft Learn, accessed October 15, 2025, [https://learn.microsoft.com/en-us/powershell/module/netadapter/disable-netadaptervmq?view=windowsserver2025-ps](https://learn.microsoft.com/en-us/powershell/module/netadapter/disable-netadaptervmq?view=windowsserver2025-ps)  
30. Hyper-V-VmSwitch Event ID 95 created on each replication \- Veeam R\&D Forums, accessed October 15, 2025, [https://forums.veeam.com/microsoft-hyper-v-f25/hyper-v-vmswitch-event-id-95-created-on-each-replication-t24481.html](https://forums.veeam.com/microsoft-hyper-v-f25/hyper-v-vmswitch-event-id-95-created-on-each-replication-t24481.html)  
31. Advanced Settings for Intel® Ethernet Adapters, accessed October 15, 2025, [https://www.intel.com/content/www/us/en/support/articles/000005593/ethernet-products.html](https://www.intel.com/content/www/us/en/support/articles/000005593/ethernet-products.html)  
32. Intel Ethernet Adapters and Devices User Guide \- Dell, accessed October 15, 2025, [https://dl.dell.com/manuals/all-products/esuprt\_data\_center\_infra\_int/esuprt\_data\_center\_infra\_network\_adapters/intel-pro-adapters\_user's-guide\_en-us.pdf](https://dl.dell.com/manuals/all-products/esuprt_data_center_infra_int/esuprt_data_center_infra_network_adapters/intel-pro-adapters_user's-guide_en-us.pdf)  
33. Optimizing Ethernet Adapter Settings for Maximum Performance \- FlexRadio, accessed October 15, 2025, [https://helpdesk.flexradio.com/hc/en-us/articles/202118518-Optimizing-Ethernet-Adapter-Settings-for-Maximum-Performance](https://helpdesk.flexradio.com/hc/en-us/articles/202118518-Optimizing-Ethernet-Adapter-Settings-for-Maximum-Performance)  
34. Tuning Tips to Improve Hyper-V Performance and Find Common Issues \- DNSstuff, accessed October 15, 2025, [https://www.dnsstuff.com/hyper-v-performance-tuning](https://www.dnsstuff.com/hyper-v-performance-tuning)  
35. Hyper-V Network I/O Performance \- Microsoft Learn, accessed October 15, 2025, [https://learn.microsoft.com/en-us/windows-server/administration/performance-tuning/role/hyper-v-server/network-io-performance](https://learn.microsoft.com/en-us/windows-server/administration/performance-tuning/role/hyper-v-server/network-io-performance)  
36. Configuration of Hyper-V Networking Best Practices \- NAKIVO, accessed October 15, 2025, [https://www.nakivo.com/blog/hyper-v-networking-best-practices/](https://www.nakivo.com/blog/hyper-v-networking-best-practices/)