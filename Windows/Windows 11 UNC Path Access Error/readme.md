
# **Analysis and Resolution of Post-Update UNC Path Failures in Windows 11 24H2/25H2**

## **Section 1: Executive Summary & Symptom Analysis**

### **1.1. Overview of the Emergent Issue**

Recent cumulative updates for Windows 11, particularly for versions 24H2 and 25H2, have introduced a series of network authentication and access issues that manifest in specific and often confusing ways. System administrators and users report a distinct failure when attempting to access network resources directly via a Universal Naming Convention (UNC) path, such as \\\\server\\drive$. Concurrently, browsing to the same location via a pre-existing mapped network drive (e.g., S:) often remains functional. This behavior is not an isolated incident but a widely reported pattern emerging in enterprise and peer-to-peer network environments.1

This core symptom is frequently accompanied by a broader profile of authentication failures across multiple network protocols. These related issues include:

* **Inaccessible File and Printer Shares:** Standard attempts to connect to shared folders or network printers fail, often prompting for credentials that are subsequently rejected despite being correct.3  
* **Failed Remote Desktop Protocol (RDP) Connections:** Attempts to establish RDP sessions between affected Windows 11 machines fail with errors indicating an "incorrect username or password" or a generic logon failure, even when valid credentials are provided.1  
* **General "Access Denied" Errors:** When accessing various network resources, users encounter "access denied" or similar permission-related errors that did not exist prior to the update installation.5

The dichotomy between a failing UNC path connection and a working mapped drive connection is a critical diagnostic clue. Accessing a resource via a direct UNC path forces the operating system to initiate a new authentication handshake, using protocols like Kerberos or New Technology LAN Manager (NTLM), to establish a security context and access rights. The recent Windows updates have inserted new, more stringent validation steps directly into this handshake process. If these new security checks fail, the handshake is aborted, and access is denied, leading to the observed error. In contrast, a pre-existing mapped drive often maintains a persistent security token from a previous, successful login session established before the update's stricter rules were in place. When a user browses this drive, the operating system may be able to reuse this existing, valid token rather than initiating a full, new handshake. This explains why an existing connection may survive while a new one fails, narrowing the problem from a general network connectivity failure to a specific breakdown in the new-session authentication protocol.

Furthermore, the simultaneous emergence of issues across disparate services like file sharing (SMB), printer sharing (SMB), and remote desktop (RDP) indicates that the problem does not lie within a single service's configuration. These services, while functionally different, all rely on the fundamental Windows authentication framework (NTLM and Kerberos) to verify user and machine identity. A bug isolated to the SMB service would be unlikely to affect RDP, and vice versa. The fact that multiple services are failing with similar credential-related symptoms strongly implies the problem lies in their common dependency: the core authentication protocols themselves. This elevates the issue from a simple service misconfiguration to a fundamental security infrastructure challenge.

### **1.2. Identification of Causative Windows Updates**

The root of these widespread issues can be traced to a pair of specific updates released by Microsoft. These updates were not flawed but rather introduced intentional, significant security enhancements to the Windows operating system. The primary updates responsible are:

* **KB5064081 (Preview Update):** Released on August 29, 2025, for OS Build 26100.5074, this preview update first introduced the stricter security checks that led to initial reports of NTLM and Kerberos authentication failures in certain environments.6  
* **KB5065426 (Cumulative Update):** Released on September 9, 2025, for OS Build 26100.6584, this mandatory cumulative update finalized and broadly deployed the security enhancements from the August preview. Its widespread installation resulted in the significant increase of file sharing, RDP, and UNC path access failures in environments with non-compliant configurations.1

It is crucial to understand that these updates are functioning as designed. They have exposed latent, insecure configurations and non-standard deployment practices that were previously tolerated by the operating system but are no longer permitted under the new, hardened security posture.

### **1.3. Introduction to the Dual Root Causes**

The investigation into these authentication failures has revealed two distinct, independent, yet philosophically linked root causes. An administrator's environment may be affected by one or both of these issues, necessitating a multi-faceted diagnostic and remediation approach. The two primary causes are:

1. **Security Identifier (SID) Duplication Enforcement:** The updates introduced a new security policy that actively blocks authentication handshakes between two machines that share the same machine SID. This primarily affects environments where operating systems were cloned or deployed without proper generalization procedures.5  
2. **Server Message Block (SMB) Protocol Hardening:** The updates enforce a more secure configuration for the SMB protocol by default. This includes mandating SMB signing (which ensures data integrity) and disabling insecure guest logons (which prevents anonymous access). This change breaks connectivity with older network devices and shares that rely on these less secure, legacy behaviors.9

The following table provides a quick-reference summary of the problematic updates and their primary impact on network operations.

| KB Number | Release Date | Affected OS Builds | Primary Impact Summary |
| :---- | :---- | :---- | :---- |
| KB5064081 | August 29, 2025 | 26100.5074 | Introduced stricter SID uniqueness checks and SMB security defaults, leading to initial reports of NTLM/Kerberos authentication failures. |
| KB5065426 | September 9, 2025 | 26100.6584 | Finalized and broadly deployed the security enhancements from the preview, causing widespread file sharing, RDP, and UNC path access failures in non-compliant environments. |

## **Section 2: Technical Analysis of Causal Factors**

### **2.1. The Security Identifier (SID) Collision: A Latent Flaw Exposed**

#### **2.1.1. What is a SID?**

A Security Identifier (SID) is a unique, immutable alphanumeric string that the Windows operating system uses to identify a security principal, such as a user account, a security group, or a computer account. For example, a typical machine SID might look like S-1-5-21-3623811015-3361044348-30300820-1013. When an administrator grants access to a resource, the system records the SID in the resource's Access Control List (ACL), not the user or computer name. During any access attempt, the operating system compares the SID of the requester against the SIDs in the ACL to make its authorization decision. This reliance on the SID as the definitive identifier is fundamental to the entire Windows security model.5

#### **2.1.2. The Source of Duplication**

The primary source of duplicate machine SIDs is the improper cloning or imaging of a Windows installation. In many rapid deployment scenarios, administrators create a "golden image" of a configured operating system and then copy that image directly onto multiple machines. If this is done without using the official Microsoft System Preparation tool (Sysprep) with the /generalize switch, every machine created from that image will have an identical machine SID. The /generalize command is specifically designed to remove all unique system information, including the SID, so that a new, unique SID is generated when the cloned OS boots for the first time. Skipping this critical step is a long-standing but incorrect practice that leads to a network of machines with conflicting identities.5

#### **2.1.3. The Policy Shift in KB5065426**

Prior to the August and September 2025 updates, the existence of duplicate SIDs on a network was a known violation of best practices and could cause esoteric issues, particularly with services like Windows Server Update Services (WSUS), but it was not a condition that actively blocked core authentication protocols. The updates for Windows 11 24H2 and 25H2 fundamentally changed this. They introduced a new security design that *enforces SID uniqueness checks* during the NTLM and Kerberos authentication handshakes.5 This is not a bug, but a deliberate policy shift to mitigate potential security risks associated with ambiguous machine identities. Microsoft's documentation states, "This design change blocks authentication handshakes between such devices".5

#### **2.1.4. The Authentication Breakdown**

The failure mechanism occurs deep within the Local Security Authority Subsystem Service (LSASS), which is responsible for enforcing security policy on the system. When one Windows 11 machine (the client) attempts to connect to another (the server) for a resource like an SMB share or an RDP session, an authentication handshake begins. As part of this process, the new security checks compare identity information from the two machines. If both machines present the same machine SID, the system detects what it logs as a "partial mismatch in the machine ID." This condition is now treated as a potential security compromise, suggesting that an authentication ticket may have been manipulated or belongs to a different session. Consequently, LSASS actively rejects the authentication attempt, preventing the connection from being established. This rejection manifests to the end-user as the observed "incorrect password" or "access denied" errors. System administrators can find definitive evidence of this failure in the System event log on the target machine, where an Event ID 6167 from the source lsasrv will be recorded with the description, "There is a partial mismatch in the machine ID".4

### **2.2. The SMB Security Mandate: Deprecating Legacy Protocols**

#### **2.2.1. Introduction to SMB Signing**

Server Message Block (SMB) signing, also known as security signatures, is a security feature of the SMB protocol. When enabled, every SMB message exchanged between a client and a server contains a digital signature. This signature is generated using a session key and a cryptographic hash of the entire message. When a message is received, the recipient recalculates the hash and compares it to the signature. If they match, it proves two things: 1\) the message has not been altered in transit (integrity), and 2\) the message originated from the authenticated sender (authenticity). This mechanism is a powerful defense against man-in-the-middle (MITM) attacks, where an attacker intercepts and modifies network traffic, and relay attacks.10

#### **2.2.2. The New Default in Windows 11 24H2**

A major security posture enhancement in Windows 11 24H2 (specifically for Pro, Enterprise, and Education editions) is that SMB signing is now *required by default* for all outbound client connections and inbound server connections. Previous versions of Windows supported SMB signing, but it was not mandatory unless specifically configured via Group Policy or if connecting to high-security shares like SYSVOL on a domain controller. This shift from "supported" to "required" is a significant change aimed at improving baseline network security across the Windows ecosystem.10

#### **2.2.3. The Impact on Legacy Devices**

This new requirement is the direct cause of connection failures to many older or non-Windows network devices. A large number of consumer-grade Network Attached Storage (NAS) devices, older Windows servers, and some default configurations of Samba (the SMB implementation for Linux/Unix) do not support or do not have SMB signing enabled. When a fully updated Windows 11 24H2 client attempts to connect to one of these legacy devices, it initiates the connection by insisting that all subsequent communication be signed. The non-compliant server, unable to fulfill this requirement, rejects the connection parameters, and the session fails to establish. This results in an access error, even if credentials are correct and permissions are properly configured.9

#### **2.2.4. Disabling of Insecure Guest Logons**

In parallel with the SMB signing mandate, Microsoft has also disabled insecure guest logons by default. For years, Windows has allowed a fallback to a "guest" account for network share access if authentication failed. This permitted anonymous, unauthenticated access to any share configured for "Everyone" or "Guest" access. While convenient for simple home networks, this practice is a significant security vulnerability in any managed environment. The recent updates have disabled this behavior, meaning that all access to network shares must now be authenticated with a valid user account and password. This change directly breaks access to any shares on NAS devices or servers that were intentionally configured to rely on guest access for their operation.3

These updates, targeting both SID duplication and SMB security, should not be viewed as a series of unrelated bugs. They are a deliberate "forcing function" by Microsoft, designed to compel administrators to abandon insecure, legacy practices that have persisted for years. The practices of cloning without Sysprep and using unsigned SMB or guest-based shares have long been discouraged, yet they remained common because they were convenient and, until now, functional. By changing the default behavior from permissive to secure and causing active failures, Microsoft has created a compelling event that forces IT departments to address their accumulated technical debt. This represents a strategic shift from passive guidance to active enforcement at the protocol level.

Fundamentally, the two root causes are philosophically linked by the common theme of identity and trust. The SID issue is about the integrity of *machine identity*; a duplicate SID means Windows cannot uniquely and trustworthily identify a computer, and the update now treats this ambiguity as a security threat. The SMB hardening issue is about *communication trust*; unsigned messages can be tampered with, and guest logons offer no verifiable identity. The update now refuses to participate in communication that cannot guarantee integrity and sender authenticity. Both problems stem from a failure to meet a higher, modern standard for cryptographic proof of identity and trust, framing the issue as a unified security posture enhancement across the operating system.

## **Section 3: Strategic Remediation Guide**

The following remediation steps are organized to provide a tiered response, from immediate but temporary workarounds to permanent, supported solutions. The choice of which path to follow depends on the urgency of service restoration, the scale of the impact, and the organization's security posture.

### **3.1. Tactical Rollback: Update Uninstallation (Immediate but Temporary)**

#### **3.1.1. Rationale and Warnings**

Uninstalling the causative update (primarily KB5065426) is the fastest method to restore network functionality. However, this must be considered a strictly temporary measure. This action reverts the security enhancements and also removes any other critical security patches contained within the cumulative update, leaving the system vulnerable to exploitation. This approach should only be used as a stopgap to restore critical services while a permanent remediation plan is implemented.1

#### **3.1.2. Method 1: Using the Settings GUI**

For individual machines, the update can be uninstalled through the Windows Settings interface:

1. Open the **Settings** app (Windows Key \+ I).  
2. Navigate to **Windows Update** in the left-hand pane.  
3. Click on **Update history**.  
4. Under the "Related settings" section, click **Uninstall updates**.  
5. Locate the update identified as **KB5065426** in the list, select it, and click the **Uninstall** button.16  
6. Restart the computer when prompted to complete the process.

#### **3.1.3. Method 2: Using Command Line (wusa.exe)**

For scripting, remote administration, or automation, the Windows Update Standalone Installer (wusa.exe) provides a command-line interface for uninstallation.

1. Open **Command Prompt** or **PowerShell** as an Administrator.  
2. Execute the following command:  
   wusa /uninstall /kb:5065426

   This command will initiate the uninstallation process. Switches like /quiet and /norestart can be added for unattended execution.17

#### **3.1.4. Troubleshooting Uninstallation Errors**

In some cases, the uninstallation process may fail with specific error codes.

* **Error 0x800f0905:** This error typically indicates that the update's source files are missing or the Component-Based Servicing (CBS) store is corrupted. Before re-attempting the uninstall, run system file integrity checks from an administrative Command Prompt 20:  
  DISM /Online /Cleanup-Image /RestoreHealth  
  sfc /scannow

  After these commands complete, restart the machine and try the uninstall again.  
* **Error 0x800F0825:** This error has been linked to a conflict with the Windows Sandbox feature. If this error occurs, a potential fix is to temporarily disable the feature 21:  
  1. Open **Turn Windows features on or off**.  
  2. Uncheck the box next to **Windows Sandbox**.  
  3. Click **OK** and restart the PC.  
  4. Attempt to uninstall the update again. The feature can be re-enabled after the update is removed.

A failed update uninstallation should be treated as a significant red flag. It often points to deeper system file corruption, suggesting that the machine's overall health is compromised. The update failure, in this context, serves as a diagnostic trigger for performing a more thorough system health check, as simply addressing the network issue may leave other underlying problems unresolved.

### **3.2. Configuration-Based Mitigation (Addressing SMB Hardening)**

These steps address the issues caused by the new SMB security requirements. They involve selectively weakening the new security defaults and should be implemented with a clear understanding of the associated risks.

#### **3.2.1. Disabling the SMB Signing Requirement**

* **Security Advisory:** Disabling the requirement for SMB signing exposes network traffic to potential tampering and man-in-the-middle attacks. This change should only be made on physically secure and trusted networks where all devices are known and managed. It is not recommended for enterprise or untrusted environments.13  
* **PowerShell Method (Client-side):** To configure a Windows 11 client to connect to servers that do not support SMB signing, execute the following command in an elevated PowerShell session:  
  PowerShell  
  Set-SmbClientConfiguration \-RequireSecuritySignature $false

  This command tells the SMB client that signing is preferred but not mandatory.10  
* **Group Policy Method:** For managed environments, this setting can be controlled via Group Policy:  
  1. Open the Local Group Policy Editor (gpedit.msc).  
  2. Navigate to: Computer Configuration \> Windows Settings \> Security Settings \> Local Policies \> Security Options.  
  3. Find the policy named **Microsoft network client: Digitally sign communications (always)**.  
  4. Set this policy to **Disabled**.10

#### **3.2.2. Re-enabling Insecure Guest Logons**

* **Security Advisory:** Enabling insecure guest logons allows anonymous access to network shares and is a major security risk. It circumvents proper authentication and access control. This should be considered a last resort for accessing legacy devices that cannot be reconfigured for authenticated access.3  
* **Group Policy Method (for Pro/Enterprise Editions):**  
  1. Open the Local Group Policy Editor (gpedit.msc).  
  2. Navigate to: Computer Configuration \> Administrative Templates \> Network \> Lanman Workstation.  
  3. Find the policy named **Enable insecure guest logons**.  
  4. Set this policy to **Enabled**.24  
* **Registry Method (for all Editions):**  
  1. Open the Registry Editor (regedit.exe).  
  2. Navigate to the key: HKEY\_LOCAL\_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\LanmanWorkstation\\Parameters.  
  3. Right-click in the right pane, select **New** \> **DWORD (32-bit) Value**.  
  4. Name the new value AllowInsecureGuestAuth.  
  5. Double-click the new value and set its data to 1\.16  
  6. A system restart is required for the change to take effect.

### **3.3. The Definitive Solution: Resolving SID Duplication with Sysprep**

This is the only Microsoft-supported and permanent solution for fixing authentication failures caused by duplicate SIDs. The process involves using the System Preparation tool to regenerate a unique SID for the affected machine, thereby bringing it into compliance with the new security requirements.5

#### **3.3.1. Rationale**

The Sysprep tool with the /generalize switch is designed specifically for this purpose. It strips the unique identity information from a Windows installation, including the machine SID, and prepares the OS for a "mini-setup" on the next boot, during which a new, unique SID is generated. This resolves the root cause of the SID collision.

#### **3.3.2. Critical Prerequisites**

Executing Sysprep is a significant system-level operation. Failure to perform these prerequisite steps can result in data loss or a non-bootable system.

* **Full System Backup:** Before proceeding, create a complete, verified backup of the system disk.  
* **Disjoin from Domain:** If the machine is a member of an Active Directory domain, it must be disjoined and moved to a workgroup. The Sysprep process will break the existing domain trust.  
* **Local Administrator Account:** Ensure there is a local administrator account on the machine and that its password is known. After Sysprep, domain accounts will not be able to log in until the machine is re-joined to the domain.

#### **3.3.3. Step-by-Step Execution**

1. Open **Command Prompt** as an Administrator.  
2. Navigate to the Sysprep directory by typing: cd C:\\Windows\\System32\\Sysprep.  
3. Execute the main Sysprep command:  
   sysprep.exe /generalize /oobe /reboot

   * /generalize: This is the most critical switch. It removes the system-specific data, including the SID and other hardware-related information.  
   * /oobe: This switch instructs the system to run the "Out-of-Box Experience" (the initial Windows setup screens) on the next reboot.  
   * /reboot: This switch automatically restarts the computer after Sysprep completes.

#### **3.3.4. Post-Sysprep Actions**

After the machine reboots, it will present the initial Windows setup screens (OOBE). The following steps are required:

1. Proceed through the setup screens to configure region, keyboard layout, etc.  
2. When prompted, create a new temporary local user account.  
3. Once logged in with the new account, rename the computer to its desired name.  
4. Re-join the computer to the Active Directory domain, if applicable.  
5. Migrate any user profile data from the old user profiles to the new ones created after the domain join.

#### **3.3.5. Automated/Remote Execution**

For managing multiple affected machines, especially in a remote context, the post-Sysprep OOBE can be automated using an unattend.xml answer file. This advanced technique allows an administrator to pre-configure all the setup options, including computer name, domain join information, and local account creation. The Sysprep command is modified to reference this file, enabling a "zero-touch" regeneration of the SID 11:

sysprep.exe /generalize /oobe /reboot /unattend:C:\\path\\to\\autounattend.xml

The available remediation paths exist on a spectrum of risk versus effort. Uninstalling an update is a low-effort, high-risk action. Modifying registry or policy settings is a medium-effort, medium-risk approach that selectively weakens security. Running Sysprep is a high-effort, zero-risk solution that represents the correct and permanent fix. This framework allows an administrator to make an informed, strategic decision that aligns with their organization's specific security posture and operational constraints.

## **Section 4: Proactive Administration and Long-Term Strategy**

Moving beyond reactive fixes, administrators should adopt a proactive stance to prevent these issues from recurring and to align their infrastructure with modern security best practices. The recent update-related failures serve as a critical learning event, highlighting areas of technical debt and procedural gaps that require strategic attention.

### **4.1. Ancillary Troubleshooting and Considerations**

Even after addressing the primary root causes, some lingering issues may persist due to cached information or secondary environmental factors.

* **Clearing Cached Credentials:** Windows Credential Manager stores saved usernames and passwords for network locations. If an incorrect or stale credential for a server is stored, it can cause authentication to fail even if the underlying SID or SMB issue is resolved. Administrators should instruct users to clear these cached entries 11:  
  1. Open **Control Panel** and navigate to **Credential Manager**.  
  2. Under the **Windows Credentials** section, locate any saved credentials corresponding to the problematic server (e.g., \\\\server-name).  
  3. Select the credential and click **Remove**.  
  4. The next connection attempt will prompt for credentials anew.  
* **Investigating IPv6:** A subset of users has reported that disabling IPv6 on the client machine's network adapter resolved persistent network share access problems after other fixes were applied.9 This suggests a potential bug or misconfiguration in how Windows 11 24H2 handles name resolution or protocol binding in mixed IPv4/IPv6 environments. While disabling IPv6 is not a recommended long-term solution, it can be used as a temporary, last-resort troubleshooting step to isolate the problem.

### **4.2. Best Practices for System Imaging and Deployment**

The SID duplication crisis underscores the critical importance of adhering to proper system imaging and deployment protocols.

* **The Sysprep Mandate:** The use of Sysprep /generalize is not an optional or "nice-to-have" step; it is a mandatory requirement for any workflow that involves creating a master OS image for deployment to multiple machines. This applies to all deployment technologies, including Microsoft Deployment Toolkit (MDT), System Center Configuration Manager (SCCM), and third-party cloning tools.5 All deployment processes must be audited and updated to ensure this step is correctly implemented.  
* **Golden Image Management:** A "golden image" should be maintained in a generalized state. The recommended workflow is to build the base image, install applications, and configure settings, then run Sysprep /generalize /shutdown. This generalized image is then captured for deployment. Each time a machine is deployed from this image, it will automatically undergo the mini-setup process and generate a unique SID.

This entire event serves as a powerful case study for the importance of treating infrastructure as code and embracing automation. The root cause of the SID issue is a manual, inconsistent, and incorrect system setup process. The fix, especially when applied at scale, is tedious and error-prone if performed manually. The most successful remote remediation documented involved an automated approach using an unattend.xml file to ensure consistency and reduce risk.11 This points to a larger conclusion: the long-term solution is to transition from manual "click-ops" administration to robust, automated deployment systems like MDT or Windows Autopilot. These systems enforce consistency and incorporate best practices like Sysprep by design, making this entire class of problem impossible in the first place. The update is not just pushing a fix; it is pushing a fundamental shift in administrative philosophy.

### **4.3. Modernizing Network Share Security**

The SMB hardening changes signal a clear direction from Microsoft: insecure legacy protocols will no longer be tolerated by default. Administrators must proactively modernize their network environments.

* **Auditing and Upgrading:** A comprehensive audit of the network should be conducted to identify all devices (especially NAS units, routers with USB sharing, and Linux/Samba servers) that do not support modern SMB security features like SMB signing. A plan should be developed to update the firmware on these devices, reconfigure them to enable signing, or replace them with compliant hardware.13  
* **Eliminating Guest Access:** A firm security policy should be established to eliminate all reliance on guest or anonymous network shares. All access to network resources should be explicitly authenticated through unique user accounts that are granted the minimum necessary permissions, adhering to the principle of least privilege.  
* **Transitioning to Authenticated Shares:** Instead of using guest access, administrators should create standard user accounts on their file servers or NAS devices. These accounts can then be granted specific "Share" and "NTFS" (or equivalent) permissions for the folders they need to access. This provides a clear audit trail and ensures that only authorized individuals can access sensitive data.

Ultimately, these recent Windows updates have redefined the landscape of IT risk management. The operational risk is no longer just *failing to patch* (and being exposed to external threats) but also *applying a patch* (and triggering a service outage due to latent internal misconfigurations). This means that Microsoft's security updates now function as part of the "attack surface" for an unprepared environment. IT departments can no longer afford to let technical debt accumulate. They must engage in continuous, proactive hygiene—auditing for legacy protocols, correcting improper deployment practices, and aligning with modern security standards—because the next "Patch Tuesday" could be the catalyst that transforms a dormant configuration issue into a critical, enterprise-wide outage. The update cycle itself has become an unforgiving, but necessary, test of infrastructure health and administrative discipline.

#### **Works cited**

1. just updated windows 11 KB5065426 network shares no longer ..., accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/5551009/just-updated-windows-11-kb5065426-network-shares-n](https://learn.microsoft.com/en-us/answers/questions/5551009/just-updated-windows-11-kb5065426-network-shares-n)  
2. Windows 11 File Explorer & UNC paths : r/sysadmin \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/sysadmin/comments/1ec6jhe/windows\_11\_file\_explorer\_unc\_paths/](https://www.reddit.com/r/sysadmin/comments/1ec6jhe/windows_11_file_explorer_unc_paths/)  
3. KB5065426 update stops file and print sharing from working ..., accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/5551014/kb5065426-update-stops-file-and-print-sharing-from](https://learn.microsoft.com/en-us/answers/questions/5551014/kb5065426-update-stops-file-and-print-sharing-from)  
4. Windows 11 24H2 RDP, SMB, and printer sharing fail after KB5065426 (0xc000006d) between cloned workstations \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-gb/answers/questions/5570237/windows-11-24h2-rdp-smb-and-printer-sharing-fail-a](https://learn.microsoft.com/en-gb/answers/questions/5570237/windows-11-24h2-rdp-smb-and-printer-sharing-fail-a)  
5. Microsoft: Recent Windows updates cause login issues on some PCs \- Bleeping Computer, accessed October 28, 2025, [https://www.bleepingcomputer.com/news/microsoft/microsoft-recent-windows-updates-cause-login-issues-on-pcs-sharing-security-ids/](https://www.bleepingcomputer.com/news/microsoft/microsoft-recent-windows-updates-cause-login-issues-on-pcs-sharing-security-ids/)  
6. Microsoft Addresses Kerberos and NTLM Login Issues in Windows 11 | SSOJet News Central \- Breaking Boundaries. Building Tomorrow, accessed October 28, 2025, [https://ssojet.com/news/microsoft-addresses-kerberos-and-ntlm-login-issues-in-windows-11](https://ssojet.com/news/microsoft-addresses-kerberos-and-ntlm-login-issues-in-windows-11)  
7. Microsoft confirms Windows 11 login issues. Here's what's causing it | PCWorld, accessed October 28, 2025, [https://www.pcworld.com/article/2950783/microsoft-confirms-windows-11-login-issues-heres-what-you-need-to-do.html](https://www.pcworld.com/article/2950783/microsoft-confirms-windows-11-login-issues-heres-what-you-need-to-do.html)  
8. Windows 11 24H2-25/H2, Server 2025: SID duplicates cause NTLM/Kerberos authentication errors \- BornCity, accessed October 28, 2025, [https://borncity.com/win/2025/10/23/windows-11-24h2-25-h2-server-2025-sid-duplicates-cause-ntlm-kerberos-authentication-errors/](https://borncity.com/win/2025/10/23/windows-11-24h2-25-h2-server-2025-sid-duplicates-cause-ntlm-kerberos-authentication-errors/)  
9. Fix shared folders not working in Windows 11 24H2 | Scottie's Tech ..., accessed October 28, 2025, [https://scottiestech.info/2024/12/30/fix-shared-folders-not-working-in-windows-11-24h2/](https://scottiestech.info/2024/12/30/fix-shared-folders-not-working-in-windows-11-24h2/)  
10. Control SMB signing behavior | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-signing](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-signing)  
11. Having issues accessing shares between 3 computers with the same Device ID, accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/5566593/having-issues-accessing-shares-between-3-computers](https://learn.microsoft.com/en-us/answers/questions/5566593/having-issues-accessing-shares-between-3-computers)  
12. Overview of Server Message Block signing in Windows | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-signing-overview](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-signing-overview)  
13. Why Windows 11 24H2 is Blocking Your Old Network Shares (and How to Fix It), accessed October 28, 2025, [https://nzlw.co.nz/tech-posts/windows-11-24h2-smb-guest-logon-fix/](https://nzlw.co.nz/tech-posts/windows-11-24h2-smb-guest-logon-fix/)  
14. Changes to SMB Signing Enforcement Defaults in Windows 24H2 | DSInternals, accessed October 28, 2025, [https://www.dsinternals.com/en/smb-signing-windows-server-2025-client-11-24h2-defaults/](https://www.dsinternals.com/en/smb-signing-windows-server-2025-client-11-24h2-defaults/)  
15. Windows 11 24H2 SMB Signing Requirement May Break Access To Some Shared Network Folders \- Minkatec, accessed October 28, 2025, [https://minkatec.com/windows-11-24h2-smb-signing-requirement-may-break-access-to-some-shared-network-folders/](https://minkatec.com/windows-11-24h2-smb-signing-requirement-may-break-access-to-some-shared-network-folders/)  
16. Fix Print Sharing Not Working After Installing Windows 11 Update KB5065426 \- YouTube, accessed October 28, 2025, [https://www.youtube.com/watch?v=apX33otkA0A\&vl=en](https://www.youtube.com/watch?v=apX33otkA0A&vl=en)  
17. 5 Ways to uninstall an update manually on Windows 11 (2025) \- Pureinfotech, accessed October 28, 2025, [https://pureinfotech.com/uninstall-update-windows-11/](https://pureinfotech.com/uninstall-update-windows-11/)  
18. Problems with Windows Update KB5065426 \- Microsoft Q\&A, accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/5556546/problems-with-windows-update-kb5065426](https://learn.microsoft.com/en-us/answers/questions/5556546/problems-with-windows-update-kb5065426)  
19. Roll back KB5065426 to restore web-based RDP in Chrome (Windows 11 24H2), accessed October 28, 2025, [https://help.trugrid.com/en/article/roll-back-kb5065426-to-restore-web-based-rdp-in-chrome-windows-11-24h2-p95qwo/](https://help.trugrid.com/en/article/roll-back-kb5065426-to-restore-web-based-rdp-in-chrome-windows-11-24h2-p95qwo/)  
20. about trouble with windowsupdate KB5065426 \- Microsoft Q\&A, accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/5594531/about-trouble-with-windowsupdate-kb5065426](https://learn.microsoft.com/en-us/answers/questions/5594531/about-trouble-with-windowsupdate-kb5065426)  
21. For the people that can't uninstall the new Windows 11 security update \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/WindowsHelp/comments/1mz9o5z/for\_the\_people\_that\_cant\_uninstall\_the\_new/](https://www.reddit.com/r/WindowsHelp/comments/1mz9o5z/for_the_people_that_cant_uninstall_the_new/)  
22. \[Network Place (SMB)\] How to Fix Windows 11 Not Working SMB Feature? | Official Support, accessed October 28, 2025, [https://www.asus.com/support/faq/1054736/](https://www.asus.com/support/faq/1054736/)  
23. How do I fix Windows 11 0x80070035 network error? READ VIDEO NOTES\! \- YouTube, accessed October 28, 2025, [https://www.youtube.com/watch?v=ElYZjC7QNpc](https://www.youtube.com/watch?v=ElYZjC7QNpc)  
24. Windows 11 24H2 and cannot login SMB without password : r/unRAID \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/unRAID/comments/1hhl28h/windows\_11\_24h2\_and\_cannot\_login\_smb\_without/](https://www.reddit.com/r/unRAID/comments/1hhl28h/windows_11_24h2_and_cannot_login_smb_without/)  
25. How do I allow insecure guest logon? \- MindFire Technologies, Inc, accessed October 28, 2025, [https://gomindfire.com/2025/02/10/how-do-i-allow-insecure-guest-logon/](https://gomindfire.com/2025/02/10/how-do-i-allow-insecure-guest-logon/)  
26. HOWTO Enable Insecure Guest Logon on W11 \- Stefano Bordoni, accessed October 28, 2025, [https://www.stefanobordoni.cloud/howto-enable-insecure-guest-logon-on-w11/](https://www.stefanobordoni.cloud/howto-enable-insecure-guest-logon-on-w11/)  
27. I'm receiving an error when trying to connect to a network share setup with anonymous logon \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/2186242/im-receiving-an-error-when-trying-to-connect-to-a](https://learn.microsoft.com/en-us/answers/questions/2186242/im-receiving-an-error-when-trying-to-connect-to-a)  
28. Network Issue in Windows 11, Like Everyone Else. \- Microsoft Q\&A, accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/2223909/network-issue-in-windows-11-like-everyone-else](https://learn.microsoft.com/en-us/answers/questions/2223909/network-issue-in-windows-11-like-everyone-else)  
29. Allow insecure guest authentication \- GitHub, accessed October 28, 2025, [https://gist.github.com/2acf07a232cd6dff4a5aa0f9c93319d8](https://gist.github.com/2acf07a232cd6dff4a5aa0f9c93319d8)  
30. Enabling SMB guest access via registry breaks the Workstation service beyond repair, accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/1462771/enabling-smb-guest-access-via-registry-breaks-the](https://learn.microsoft.com/en-us/answers/questions/1462771/enabling-smb-guest-access-via-registry-breaks-the)