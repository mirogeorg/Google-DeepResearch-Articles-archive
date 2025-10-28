
# **Architecting a Secure File Share: A Defense-in-Depth Implementation Guide**

## **Introduction: Architecting a Secure Share with a Defense-in-Depth Strategy**

The creation of a network file share is a common administrative task, yet its configuration is critical to the security posture of an organization. A hastily configured share can become a significant vulnerability, providing a vector for data exfiltration or the lateral movement of malware. This report provides an exhaustive, step-by-step methodology for provisioning a highly secure network share on a Windows operating system. The objective is to grant write access to a single, dedicated remote user while systematically hardening the host system against potential exploits initiated by that user.

This guide is architected around several core cybersecurity principles that form a cohesive, multi-layered defense. By implementing controls at each layer of the system—from user identity to network access—the final configuration will be resilient, auditable, and secure by design.

The first governing principle is the **Principle of Least Privilege (PoLP)**. This fundamental concept dictates that any user, program, or process should have only the bare minimum permissions necessary to perform its intended function.1 This principle will be applied throughout the guide, informing the creation of a non-administrative user account, the assignment of granular file system permissions, and the configuration of network firewall rules.

The second principle is **Defense-in-Depth**. This strategy rejects reliance on a single security control, instead layering multiple, independent security measures to protect a resource.2 Should one layer fail or be bypassed, subsequent layers are in place to thwart an attack. The architecture detailed herein builds successive layers of security: a non-privileged user (the identity layer), strict file system and share permissions (the access control layer), hardening of the network protocol (the protocol layer), and network traffic isolation (the network layer).

Finally, this process is an exercise in **Attack Surface Reduction**. An attack surface represents the sum of all possible points where an unauthorized user can attempt to enter or extract data from an environment. Each hardening step—from creating a standard user account to disabling obsolete protocols and restricting network access—is a deliberate action to minimize these entry points, thereby reducing the overall risk to the system.

By adhering to these principles, the following procedures will construct not merely a functional file share, but a robust and defensible system component.

## **Part 1: Foundational Setup: User and Share Provisioning**

The initial phase involves establishing the two fundamental components of the system: the user identity that will access the resource and the shared folder itself. These foundational elements must be created with security as the primary consideration from the outset.

### **Section 1.1: Creating a Dedicated, Non-Administrative Local User Account**

The cornerstone of this secure configuration is a dedicated user account that adheres strictly to the Principle of Least Privilege. A local, standard (non-administrative) user account is the optimal choice. This approach isolates the share's access credentials to the host machine, preventing a compromise of a linked Microsoft or domain account from immediately granting access to this sensitive resource.3 A local account's credentials exist only on the host computer, inherently limiting the potential impact of a credential leak.5

#### **Procedure 1 (GUI Method)**

This method utilizes the standard Windows graphical user interface for creating a local user.

1. Open the **Settings** application by pressing the Windows key \+ I, or by right-clicking the Start button and selecting "Settings".  
2. Navigate to the **Accounts** section.  
3. In the left-hand pane, select **Other users** (this may be labeled "Family & other users" in some versions of Windows).3  
4. Under the "Other users" heading, click on **Add someone else to this PC** or **Add account**.3  
5. The system will prompt for Microsoft account sign-in information. It is critical to bypass this. Click the link that reads **I don't have this person's sign-in information**.4  
6. On the next screen, again decline the option to create a Microsoft account by clicking **Add a user without a Microsoft account**.4  
7. You will now be prompted to create the local account. Enter a descriptive username (e.g., shareuser), a strong, complex password, and provide answers for the security questions.  
8. Click **Next**. The account will be created as a "Standard User" by default.

#### **Procedure 2 (Command-Line Method)**

For system administrators and for scriptable automation, creating the user via the command line is a more efficient method. This procedure should be performed in a Command Prompt or PowerShell window running with administrative privileges.

1. Open an elevated Command Prompt or PowerShell terminal.  
2. Execute the following command to create the user, replacing \<username\> and \<password\> with the desired credentials. It is imperative to use a strong, unique password.  
   net user "\<username\>" "\<password\>" /add

3. By default, this command creates a standard user account that is a member of the Users group. To explicitly verify or add the user to this group, use the following command:  
   net localgroup users "\<username\>" /add

#### **Verification**

Regardless of the creation method, it is essential to verify that the account has been created correctly and does not possess administrative privileges.

1. Press the Windows key \+ R to open the Run dialog, type compmgmt.msc, and press Enter to open the **Computer Management** console.8  
2. In the left pane, navigate to **System Tools \> Local Users and Groups \> Users**.4  
3. Confirm that the new user account (e.g., shareuser) is listed.  
4. Double-click the user account and select the **Member Of** tab.  
5. Verify that the user is a member of the **Users** group and is **not** a member of the **Administrators** group.4 If "Administrators" is listed, select it and click "Remove".

### **Section 1.2: Establishing the Network Share**

With the user account provisioned, the next step is to create the folder that will be shared across the network.

#### **Folder Creation**

1. Using File Explorer, navigate to a non-system drive if possible (e.g., a D: drive). While the C: drive can be used, placing shares on a separate data volume is a common best practice for organization and backup.  
2. Create a new folder. Give it a descriptive local name, for example, DataShare.9 This folder must be on a volume formatted with the NTFS file system to support the granular permissions that will be configured later.8

#### **Sharing Procedure (Advanced Sharing)**

While Windows offers a simple "Give access to" wizard, the **Advanced Sharing** dialog provides the necessary granular control for a secure configuration.8

1. Right-click the newly created folder (DataShare) and select **Properties**.  
2. Navigate to the **Sharing** tab.  
3. Click the **Advanced Sharing...** button. A User Account Control (UAC) prompt may appear; if so, provide administrative credentials.  
4. In the Advanced Sharing dialog, check the box for **Share this folder**.  
5. In the **Share name** field, provide the name that network users will use to access the resource (e.g., SecureData). This name can be different from the local folder name. The network path to this share will be \\\\\<hostname\>\\\<ShareName\>, for example, \\\\FILESERVER01\\SecureData.11  
6. The subsequent steps of configuring permissions within this dialog will be detailed in Part 2\. For now, the share has been created.

## **Part 2: Implementing a Granular Access Control Model**

This section addresses the most critical aspect of the configuration: defining precisely who can access the share and what actions they are permitted to perform. A secure outcome depends on a thorough understanding of the two distinct layers of permissions that Windows applies to network shares: Share Permissions and NTFS Permissions.

### **Section 2.1: A Deep Dive into Windows Access Control: Share vs. NTFS Permissions**

Windows employs a dual-permission model for resources accessed over a network. Both permission sets are evaluated independently, and their interaction determines a user's final, effective permissions. Misunderstanding this interaction is a common source of security misconfiguration.12

**Share Permissions** can be thought of as the "gatekeeper" at the main entrance to a building. They apply *only* when the folder is accessed over the network; they have no effect on a user logged in locally to the machine.13 Share permissions are a legacy system, originating from older file systems like FAT that lacked robust, file-level security controls.16 As a result, they are relatively coarse, offering only three levels of access:

* **Read:** Allows users to view folder and file names, read file data, and run programs.13  
* **Change:** Includes all Read permissions, plus the ability to create, delete, and modify files and folders.13  
* **Full Control:** Includes all Change permissions, plus the ability to change permissions on the share itself.13

**NTFS Permissions**, in contrast, are the "security guards" inside the building, controlling access to every individual room (folder) and file cabinet (file). These permissions are an integral feature of the New Technology File System (NTFS) and apply to users whether they are accessing the resource locally or over the network.13 NTFS permissions are highly granular and provide fine-grained control over user actions. The basic NTFS permissions include:

* **Full control:** Allows users to read, write, modify, execute, and delete files and folders, as well as change permissions and take ownership of them.14  
* **Modify:** Allows users to read, write, modify, execute, and delete files and folders, but not change permissions or take ownership.14  
* **Read & execute:** Allows users to view folder contents and read file data, as well as run executable files.14  
* **List folder contents:** Allows users to see the names of files and subfolders within a folder.14  
* **Read:** Allows users to view folder contents and read file data.14  
* **Write:** Allows users to create new files and folders and make changes to existing files.14

The interaction between these two permission sets is governed by a single, inviolable rule: **When a resource is accessed over the network, both Share and NTFS permissions are calculated, and the *most restrictive* permission becomes the effective permission**.12 For instance, if a user is granted "Full Control" at the Share level but only "Read" at the NTFS level, their effective permission will be "Read". Conversely, if they have "Read" at the Share level and "Full Control" at the NTFS level, their effective permission is still only "Read".

The following table summarizes the key differences between the two permission types.

| Attribute | Share Permissions | NTFS Permissions |
| :---- | :---- | :---- |
| **Scope of Application** | Network access only. No effect on users logged in locally.13 | Applies to both network access and local access.13 |
| **Granularity** | Coarse. Three levels: Read, Change, Full Control.13 | Highly granular. Six basic permissions and numerous advanced permissions.14 |
| **Point of Application** | Applies to the entire shared folder. Cannot set different permissions for subfolders or files.16 | Can be applied to individual files and folders. Supports inheritance, which can be disabled.1 |
| **Interaction Rule** | The most restrictive permission between Share and NTFS is the effective permission for network users.12 | The most restrictive permission between Share and NTFS is the effective permission for network users.12 |
| **Best Practice** | Set to be permissive (e.g., "Full Control" for the intended user) to act as a simple gateway.16 | Use for all granular access control. This is the definitive control layer.13 |

This dual-system can create administrative complexity and lead to troubleshooting difficulties. When access is denied, an administrator must check two separate locations and mentally compute the restrictive outcome. The modern best practice, therefore, is to simplify this model by effectively neutralizing the Share permission layer and using the more powerful NTFS layer as the single source of truth for access control.

### **Section 2.2: Configuring Share Permissions: The Network Gateway**

Following the industry best practice, Share permissions will be configured to be permissive for the intended user, delegating all granular control to the NTFS layer. This simplifies administration and eliminates a potential source of conflicting permissions.14

1. Return to the **Advanced Sharing** dialog for the folder (Properties \> Sharing \> Advanced Sharing...).  
2. Click the **Permissions** button.  
3. By default, the "Everyone" group may be listed with "Read" permissions. This is overly permissive. Select the **Everyone** group and click **Remove**.  
4. Click the **Add...** button.  
5. In the "Select Users or Groups" dialog, type the name of the local user created in Part 1 (e.g., FILESERVER01\\shareuser) and click **Check Names** to validate it. Click **OK**.  
6. With the newly added user selected in the list, check the **Allow** box for **Full Control** in the permissions pane below.  
7. Click **Apply**, then **OK**.

By setting the Share permission to "Full Control" for the dedicated user, the share permission layer will never become a restrictive bottleneck. All effective access control will now be determined solely by the NTFS permissions.

### **Section 2.3: Configuring NTFS Permissions: The Definitive Control Layer**

This is the most critical step in defining access. Here, precise file-system-level permissions will be set, ensuring the user has exactly the access they need and no more.

1. Navigate to the shared folder's **Properties** dialog and select the **Security** tab.  
2. Click the **Advanced** button to open the "Advanced Security Settings" dialog.  
3. **Disable Inheritance:** The first and most crucial step is to create a clean permission slate. Click the **Disable inheritance** button at the bottom of the dialog. A prompt will appear; select **Remove all inherited permissions from this object**. This prevents any unintended permissions from parent folders from affecting the share.1 The permissions list should now be empty except for a default entry that cannot be removed.  
4. **Add the User Principal:** Click the **Add** button. In the new window, click the **Select a principal** link.  
5. Type the name of the dedicated local user (FILESERVER01\\shareuser), click **Check Names**, and then click **OK**.  
6. **Assign Permissions:** The "Auditing Entry" window will now be displayed. In the "Basic permissions" section, check the box for **Modify**. Do not select "Full Control".  
   * The **Modify** permission provides all necessary rights for a user to have "write access": they can create, read, write, and delete files and subfolders.14  
   * The **Full Control** permission additionally grants the ability to change permissions and take ownership of files. These are administrative-level rights that violate the Principle of Least Privilege and should not be granted to a standard user.1  
7. Click **OK**.  
8. **Review Final Permissions:** The "Advanced Security Settings" window should now show that the dedicated user has "Modify" permissions. The only other entries should be for essential system accounts like SYSTEM and Administrators, which typically have "Full Control". Ensure no other generic groups like "Users" or "Everyone" are listed.  
9. Click **Apply**, then **OK** to close all dialogs.

The access control model is now correctly configured. The share is accessible over the network, but only the dedicated user has permissions, and those permissions are strictly limited to what is necessary for their task.

## **Part 3: Hardening the Host and Network Attack Surface**

With the user and permissions correctly configured, the focus now shifts to hardening the system itself. This involves securing the communication protocol used for sharing (SMB), isolating the service on the network with a firewall, and implementing a critical control to mitigate the risk of remote code execution.

### **Section 3.1: Securing the SMB Protocol: Disabling and Auditing SMBv1**

The Server Message Block (SMB) protocol is the foundation of Windows file sharing. However, its first version, SMBv1, is an obsolete protocol from the 1980s with well-documented cryptographic weaknesses.18 It was famously exploited by devastating ransomware worms like WannaCry and NotPetya. Modern Windows versions use the far more secure SMBv2 and SMBv3 dialects. Leaving SMBv1 enabled on a system is a significant and unnecessary security risk.18

#### **Step 1: Auditing for SMBv1 Usage**

Before disabling the protocol, it is prudent to verify that no legacy devices on the network (such as old multifunction printers, scanners, or network-attached storage devices) still rely on it for communication. Disabling it without checking could cause operational disruptions.

1. Open an elevated PowerShell terminal.  
2. Execute the following command to enable auditing of SMBv1 access attempts:  
   PowerShell  
   Set-SmbServerConfiguration \-AuditSmb1Access $true

   20  
3. Allow the system to run for a period (e.g., several days) to collect data.  
4. To review the logs, open the **Event Viewer** (eventvwr.msc).  
5. Navigate to **Application and Services Logs \> Microsoft \> Windows \> SMBServer \> Audit**.  
6. Look for events with **Event ID 3000**. The details of these events will include the IP address of any client device that attempted to connect to the server using the SMBv1 protocol.20 If such events are found, the legacy device should be updated or replaced before proceeding.

#### **Step 2: Disabling SMBv1**

Once it is confirmed that no devices require SMBv1, it should be disabled.

* **PowerShell Method (Recommended):** This method is fast and does not require a system reboot.20 In an elevated PowerShell terminal, run:  
  PowerShell  
  Set-SmbServerConfiguration \-EnableSMB1Protocol $false

  18  
* **Windows Features Method (GUI):** This method provides a graphical interface but may require a reboot.  
  1. Open **Control Panel**, navigate to **Programs and Features**.  
  2. Click **Turn Windows features on or off**.19  
  3. In the list, find and uncheck the box for **SMB 1.0/CIFS File Sharing Support**.21  
  4. Click **OK** and restart the computer if prompted.

#### **Verification**

To confirm that SMBv1 has been successfully disabled, use one of the following PowerShell commands:

PowerShell

Get-SmbServerConfiguration | Select EnableSMB1Protocol

The output should show False.18

PowerShell

Get-WindowsOptionalFeature \-Online \-FeatureName SMB1Protocol

The output should show the state as Disabled.23

### **Section 3.2: Network Isolation: Restricting Access with Windows Defender Firewall**

By default, a new SMB share is accessible to any device on the same network. Implementing a host-based firewall rule provides a powerful layer of network isolation, ensuring that only a specific, trusted client machine can even attempt to connect to the share. This control prevents scanning and exploitation attempts from any other machine on the network.2

The following procedure details how to create a custom inbound rule using **Windows Defender Firewall with Advanced Security** (wf.msc).

1. Press the Windows key \+ R, type wf.msc, and press Enter.24  
2. In the left pane, select **Inbound Rules**.  
3. In the right "Actions" pane, click **New Rule...**.  
4. **Rule Type:** Select **Custom** and click **Next**. The "Custom" option provides the most granular control over all rule parameters.25  
5. **Program:** Leave **All programs** selected and click **Next**.  
6. **Protocol and Ports:**  
   * For **Protocol type**, select **TCP**.  
   * For **Local port**, select **Specific Ports** and enter 445\. Port 445 is the standard port for SMB traffic.2  
   * Click **Next**.  
7. **Scope:** This is the most important step for restricting access.  
   * Under the heading "Which remote IP addresses does this rule apply to?", select the radio button for **These IP addresses**.  
   * Click the **Add...** button. Enter the specific IP address (e.g., 192.168.1.100) or a subnet range (e.g., 192.168.1.0/24) of the trusted client machine(s) that should have access.24  
   * Click **OK**, then click **Next**.  
8. **Action:** Select **Allow the connection** and click **Next**.  
9. **Profile:** Check the boxes for **Domain** and **Private**. It is strongly recommended to leave **Public** unchecked, as allowing inbound SMB traffic on an untrusted public network (like a coffee shop Wi-Fi) is a significant security risk.2 Click **Next**.  
10. **Name:** Provide a descriptive name for the rule, such as Allow SMB-In from TrustedClient-192.168.1.100. Add a description for clarity. Click **Finish**.

To ensure this new, specific rule is the only one allowing SMB traffic, it is best practice to disable the more generic, built-in firewall rule.

1. In the list of **Inbound Rules**, find the rule named **File and Printer Sharing (SMB-In)**.  
2. Right-click the rule and select **Disable Rule**.

The following table provides a summary for verification of the new firewall rule.

| Parameter | Value |
| :---- | :---- |
| **Rule Name** | Allow SMB-In from TrustedClient-192.168.1.100 |
| **Direction** | Inbound |
| **Action** | Allow |
| **Program** | All Programs |
| **Protocol** | TCP |
| **Local Port** | 445 |
| **Remote Port** | Any |
| **Remote IP Address (Scope)** | 192.168.1.100 (or user-specified trusted IP) |
| **Profile** | Domain, Private |

### **Section 3.3: Mitigating Remote Execution Risk via Advanced NTFS Permissions**

This section directly addresses the core user concern: hardening the system against exploits that the remote user might run. While system-wide application whitelisting is outside the scope, a highly effective and targeted control can be implemented directly on the share itself. This advanced NTFS permission configuration prevents any executable code from being run from within the shared folder, effectively turning it into a "data-only" repository.

The challenge lies in the fact that standard NTFS permissions bundle folder navigation and file execution together. The basic "Read & Execute" permission and the advanced "Traverse Folder / Execute File" permission both grant these two distinct rights as a single unit.1 The solution is to decouple them by creating two separate and specific Access Control Entries (ACEs) for the same user, one that allows traversal on folders and another that explicitly denies execution on files.

This procedure is performed in the **Advanced Security Settings** for the shared folder (Properties \> Security \> Advanced).

1. First, remove the single "Modify" permission entry that was created for the shareuser in Part 2.3. This will be replaced with a more granular set of permissions.  
2. **Create Entry 1 (Allow Folder Traversal and Data Modification):**  
   * Click **Add**, select the shareuser principal.  
   * Click **Show advanced permissions**.  
   * Grant the following permissions: Traverse folder / execute file, List folder / read data, Read attributes, Read extended attributes, Create files / write data, Create folders / append data, Write attributes, Write extended attributes, Delete subfolders and files, and Read permissions. This combination is effectively the "Modify" permission set, but with Traverse folder / execute file explicitly included.  
   * In the **Applies to** dropdown menu, select **This folder, subfolders and files**. This is the standard setting.  
   * Click **OK**.  
3. **Create Entry 2 (Deny File Execution):**  
   * Click **Add** again, and again select the same shareuser principal.  
   * At the top of the window, change the **Type** dropdown from "Allow" to **Deny**.  
   * Click **Show advanced permissions**.  
   * Check the box for **Traverse folder / execute file**.  
   * This is the most critical step: In the **Applies to** dropdown menu, select **Files only**.28  
   * Click **OK**.

The logic of this configuration is precise. The first ACE grants the user broad permissions to manage data and navigate the directory structure. The second ACE, however, applies an explicit **Deny** for the execution permission that applies *only to files*. Because an explicit Deny permission always overrides an Allow permission, the user will be prevented from running any program, script (.exe, .bat, .ps1, etc.), or malware they upload to the share.1 However, because the Deny rule does not apply to folders, their ability to traverse the directory tree remains intact. This surgically removes the execution capability without impacting their legitimate data-handling tasks.

This layered approach demonstrates a true defense-in-depth strategy. Disabling SMBv1 hardens the service against protocol-level attacks. The firewall rule cloaks the service from unauthorized network hosts. Finally, the execution-denial permission provides a last line of defense, assuming both prior layers could fail. Even if a compromised client connects and uploads malware, this control prevents that malware from being executed from the file server, containing the breach at its source.

## **Part 4: Advanced Security Posture: Monitoring and Auditing**

A secure configuration is not complete without mechanisms for monitoring and auditing. Security is an ongoing process, and the ability to detect and investigate access attempts is a critical component of a mature security posture.

### **Section 4.1: Enabling Security Auditing for Share Access**

Enabling auditing creates a forensic trail of all access to the shared folder. These logs are invaluable for investigating unauthorized access attempts, troubleshooting permission issues, and meeting compliance requirements.30 The configuration involves two steps: enabling the system-wide audit policy and then applying a specific audit configuration to the folder itself.

#### **Step 1: Configure Audit Policy via Local Group Policy (gpedit.msc)**

1. Press the Windows key \+ R, type gpedit.msc, and press Enter.  
2. Navigate to **Computer Configuration \> Windows Settings \> Security Settings \> Advanced Audit Policy Configuration \> System Audit Policies \- Local Group Policy Object \> Object Access**.  
3. In the right pane, double-click **Audit Detailed File Share**.  
4. Check the box for **Configure the following audit events**, then check both **Success** and **Failure**. Click **OK**.31  
5. Repeat this process for the **Audit File Share** and **Audit File System** subcategories.31  
6. As a fallback for systems that may not process advanced audit policies correctly, it is also wise to configure the basic policy. Navigate to **Computer Configuration \> Windows Settings \> Security Settings \> Local Policies \> Audit Policy**.  
7. Double-click **Audit object access** and check the boxes for both **Success** and **Failure**.32

#### **Step 2: Apply Auditing SACL to the Shared Folder**

A System Access Control List (SACL) defines which actions on an object should be logged.

1. Navigate to the shared folder's **Properties \> Security \> Advanced**.  
2. Select the **Auditing** tab. Click **Continue** if prompted by UAC.  
3. Click the **Add** button.  
4. Click the **Select a principal** link. Type **Everyone** and click **Check Names**, then **OK**. Auditing for "Everyone" ensures that all access attempts are logged, regardless of the user.32  
5. In the "Auditing Entry" window, you can select the specific actions to monitor. For comprehensive logging, select **Full control**. This will automatically select all basic permissions like "Write," "Delete," and "Read".33  
6. Ensure that the checkboxes for both **Success** and **Failure** are selected at the top of the permissions list.  
7. Click **OK**, then **Apply** and **OK** to close the dialogs.

#### **Step 3: Reviewing Logs in Event Viewer (eventvwr.msc)**

With auditing configured, all access attempts will be logged in the Windows Security Event Log.

1. Open **Event Viewer** by pressing Windows key \+ R and typing eventvwr.msc.  
2. Navigate to **Windows Logs \> Security**.  
3. The log may contain a large number of events. Use the "Filter Current Log..." action to find relevant events. Key Event IDs related to file share access include:  
   * **Event ID 4663:** An attempt was made to access an object. This is a highly detailed event that shows the user, the object name (the file or folder), the process that made the request, and the type of access requested (e.g., ReadData, WriteData).32  
   * **Event ID 5145:** A network share object was checked. This event is useful for seeing who is attempting to connect to the share and from which source IP address.33  
   * **Event ID 4656:** A handle to an object was requested. This often precedes a 4663 event and indicates the initial request for access.

By regularly reviewing these logs, an administrator can monitor for anomalous activity, such as access attempts from unauthorized IP addresses, failed access attempts by the authorized user (indicating a possible credentials issue), or unexpected file deletions.

### **Section 4.2: A Note on Controlled Folder Access (CFA)**

Windows includes a security feature called Controlled Folder Access (CFA), which is designed primarily as an anti-ransomware measure.34 CFA works by preventing untrusted applications from making changes to files within a list of protected folders, which typically includes the user's Documents, Pictures, and Desktop folders.37

While CFA is a valuable tool for endpoint protection, it is not the appropriate security control for this file share scenario. CFA is designed to protect a *locally logged-on user* from malicious processes running *on their own machine*. It is not intended to function as an access control mechanism for a *remote user accessing a network share*. Official documentation explicitly warns against adding network share paths (e.g., \\\\server\\share) to the list of protected folders and instead advises using the local path (e.g., C:\\DataShare).37 Applying CFA in this context would be a misapplication of the technology and would not provide the intended protection against a remote user's actions. The advanced NTFS permissions configured in Section 3.3 provide a more direct and appropriate control for mitigating the risk of remote code execution from the share.

## **Conclusion: A Review of the Hardened Share Architecture**

This report has detailed a comprehensive, multi-layered strategy for creating and securing a network file share on a Windows system. By moving beyond a simple "right-click and share" approach and instead deliberately architecting the solution around established cybersecurity principles, the resulting configuration is robust, resilient, and auditable.

The final architecture incorporates a series of mutually reinforcing controls that exemplify a defense-in-depth strategy:

* **Identity and Access Management:** A dedicated, least-privilege **local user account** was created, isolating the share's credentials from broader identity systems. Access control was implemented using a simplified Share permission model that delegates all granular control to the **definitive NTFS permission layer**, where the user was granted only the "Modify" rights necessary for their function.  
* **Protocol and Network Hardening:** The attack surface was significantly reduced by disabling the obsolete and vulnerable **SMBv1 protocol**. Network isolation was achieved by implementing a specific **Windows Defender Firewall rule** that restricts all inbound SMB traffic to a single, trusted source IP address, cloaking the service from the rest of the network.  
* **Exploit Mitigation:** A critical control was implemented to address the risk of remote code execution. A nuanced, two-part **advanced NTFS permission entry** was configured to explicitly deny the "Execute" permission on all files within the share, effectively transforming it into a data-only repository and preventing it from being used to launch malware.  
* **Detection and Response:** A robust **security auditing configuration** was enabled at both the system policy and folder level, providing a detailed forensic log of all access attempts, which can be reviewed in the Windows Event Viewer.

For ongoing security, system administrators should adhere to the following recommendations:

* **Regularly Review Audit Logs:** Periodically inspect the Security Event Log for anomalous or unauthorized access patterns.  
* **System Patching:** Ensure the host operating system is kept up-to-date with the latest security patches from Microsoft to protect against newly discovered vulnerabilities.  
* **Password Management:** The password for the dedicated share user account should be managed securely, stored in a password vault, and rotated according to organizational policy.

By following the procedures outlined in this guide, an administrator can be confident that they have deployed a network file share that is not only functional but also aligned with modern security best practices.

#### **Works cited**

1. A Complete Guide to Windows NTFS Permissions (2025) \- 2BrightSparks, accessed October 23, 2025, [https://www.2brightsparks.com/resources/articles/a-basic-introduction-to-ntfs-permissions.html](https://www.2brightsparks.com/resources/articles/a-basic-introduction-to-ntfs-permissions.html)  
2. Preventing SMB traffic from lateral connections and entering or leaving the network \- Microsoft Support, accessed October 23, 2025, [https://support.microsoft.com/en-us/topic/preventing-smb-traffic-from-lateral-connections-and-entering-or-leaving-the-network-c0541db7-2244-0dce-18fd-14a3ddeb282a](https://support.microsoft.com/en-us/topic/preventing-smb-traffic-from-lateral-connections-and-entering-or-leaving-the-network-c0541db7-2244-0dce-18fd-14a3ddeb282a)  
3. Manage User Accounts in Windows \- Microsoft Support, accessed October 23, 2025, [https://support.microsoft.com/en-us/windows/manage-user-accounts-in-windows-104dc19f-6430-4b49-6a2b-e4dbd1dcdf32](https://support.microsoft.com/en-us/windows/manage-user-accounts-in-windows-104dc19f-6430-4b49-6a2b-e4dbd1dcdf32)  
4. Migrating to a non-privileged User Account \- ASU Get Protected \- Arizona State University, accessed October 23, 2025, [https://getprotected.asu.edu/migrating-non-privileged-user-account](https://getprotected.asu.edu/migrating-non-privileged-user-account)  
5. How to create a Local account in Windows 11 \- ARTICLE \- Microsoft ..., accessed October 23, 2025, [https://learn.microsoft.com/en-us/answers/questions/2337096/how-to-create-a-local-account-in-windows-11-articl](https://learn.microsoft.com/en-us/answers/questions/2337096/how-to-create-a-local-account-in-windows-11-articl)  
6. How to make a local account in Windows 11 : r/WindowsHelp \- Reddit, accessed October 23, 2025, [https://www.reddit.com/r/WindowsHelp/comments/1cgrzwt/how\_to\_make\_a\_local\_account\_in\_windows\_11/](https://www.reddit.com/r/WindowsHelp/comments/1cgrzwt/how_to_make_a_local_account_in_windows_11/)  
7. How to Create a Local Account on Windows 11 (Without Microsoft Account) \- YouTube, accessed October 23, 2025, [https://www.youtube.com/watch?v=SL2cfavHFe4](https://www.youtube.com/watch?v=SL2cfavHFe4)  
8. How to Set Up Network File Sharing in Windows \- NAKIVO, accessed October 23, 2025, [https://www.nakivo.com/blog/setup-network-file-sharing-in-windows/](https://www.nakivo.com/blog/setup-network-file-sharing-in-windows/)  
9. How to Create a Network Shared Folder \- BDS \- Copiers, Printers, Document Management, accessed October 23, 2025, [https://bdsdoc.com/kb-articles/how-to-create-a-network-shared-folder/](https://bdsdoc.com/kb-articles/how-to-create-a-network-shared-folder/)  
10. Business Storage Windows Server NAS \- How to create a shared folder | Seagate US, accessed October 23, 2025, [https://www.seagate.com/support/kb/business-storage-windows-server-nas-how-to-create-a-shared-folder-005868en/](https://www.seagate.com/support/kb/business-storage-windows-server-nas-how-to-create-a-shared-folder-005868en/)  
11. How to set up private home network between laptop and PC on Windows 10 \- Reddit, accessed October 23, 2025, [https://www.reddit.com/r/techsupport/comments/z0nta8/how\_to\_set\_up\_private\_home\_network\_between\_laptop/](https://www.reddit.com/r/techsupport/comments/z0nta8/how_to_set_up_private_home_network_between_laptop/)  
12. What are the differences between NTFS and Share permissions? \- NetApp Knowledge Base, accessed October 23, 2025, [https://kb.netapp.com/on-prem/ontap/da/NAS/NAS-KBs/What\_are\_the\_differences\_between\_NTFS\_and\_Share\_permissions](https://kb.netapp.com/on-prem/ontap/da/NAS/NAS-KBs/What_are_the_differences_between_NTFS_and_Share_permissions)  
13. NTFS Permissions and Share Permissions – What's the difference?, accessed October 23, 2025, [https://www.tenfold-security.com/en/ntfs-permissions-and-share-permissions-whats-the-difference/](https://www.tenfold-security.com/en/ntfs-permissions-and-share-permissions-whats-the-difference/)  
14. NTFS Permissions vs Share: Everything You Need to Know \- Varonis, accessed October 23, 2025, [https://www.varonis.com/blog/ntfs-permissions-vs-share](https://www.varonis.com/blog/ntfs-permissions-vs-share)  
15. NTFS vs. Share Permissions: What's the Difference? \- DNSstuff, accessed October 23, 2025, [https://www.dnsstuff.com/ntfs-vs-share-permissions](https://www.dnsstuff.com/ntfs-vs-share-permissions)  
16. Your Guide to NTFS Vs. Share Permissions Best Practices \- Global Knowledge, accessed October 23, 2025, [https://www.globalknowledge.com/us-en/resources/resource-library/articles/your-guide-to-ntfs-vs-share-permissions-best-practices/](https://www.globalknowledge.com/us-en/resources/resource-library/articles/your-guide-to-ntfs-vs-share-permissions-best-practices/)  
17. NTFS Permissions : An Overview, accessed October 23, 2025, [https://www.permissionsreporter.com/ntfs-permissions](https://www.permissionsreporter.com/ntfs-permissions)  
18. Disable SMBv1: Understanding Risks and Remediation Steps \- CalCom Software, accessed October 23, 2025, [https://calcomsoftware.com/disable-hardening-smbv1/](https://calcomsoftware.com/disable-hardening-smbv1/)  
19. How to Enable or Disable SMB1 File Sharing Protocol in Windows \- NinjaOne, accessed October 23, 2025, [https://www.ninjaone.com/blog/enable-or-disable-smb1-file-sharing-protocol/](https://www.ninjaone.com/blog/enable-or-disable-smb1-file-sharing-protocol/)  
20. How To Disable SMBv1 on Windows Server Without Rebooting \- Our Cloud Network, accessed October 23, 2025, [https://ourcloudnetwork.com/how-to-disable-smbv1-on-windows-server-without-rebooting/](https://ourcloudnetwork.com/how-to-disable-smbv1-on-windows-server-without-rebooting/)  
21. How to Enable or Disable SMB 1.0 in Windows 10/11 and Windows Server, accessed October 23, 2025, [https://woshub.com/how-to-disable-smb-1-0-in-windows-10-server-2016/](https://woshub.com/how-to-disable-smb-1-0-in-windows-10-server-2016/)  
22. How to disable server-side SMB1? \- Audit Square, accessed October 23, 2025, [https://auditsquare.com/advisory/windows/how-to-disable-smb1](https://auditsquare.com/advisory/windows/how-to-disable-smb1)  
23. Detect, enable, and disable SMBv1, SMBv2, and SMBv3 in Windows ..., accessed October 23, 2025, [https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/detect-enable-and-disable-smbv1-v2-v3](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/detect-enable-and-disable-smbv1-v2-v3)  
24. Configuring Windows Firewall to Allow or Block IP Addresses ..., accessed October 23, 2025, [https://knowledge.civilgeo.com/configuring-windows-firewall-to-allow-or-block-ip-addresses/](https://knowledge.civilgeo.com/configuring-windows-firewall-to-allow-or-block-ip-addresses/)  
25. Configure Firewall Rules With Group Policy | Microsoft Learn, accessed October 23, 2025, [https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure)  
26. Shares not accessible by other computers if Windows 10 firewall is ON \- Super User, accessed October 23, 2025, [https://superuser.com/questions/1062208/shares-not-accessible-by-other-computers-if-windows-10-firewall-is-on](https://superuser.com/questions/1062208/shares-not-accessible-by-other-computers-if-windows-10-firewall-is-on)  
27. File and Folder Advanced Permissions \- NTFS.com, accessed October 23, 2025, [http://ntfs.com/ntfs-permissions-file-advanced.htm](http://ntfs.com/ntfs-permissions-file-advanced.htm)  
28. How to deny execute permissions on a share/folder in Windows ..., accessed October 23, 2025, [https://serverfault.com/questions/221227/how-to-deny-execute-permissions-on-a-share-folder-in-windows](https://serverfault.com/questions/221227/how-to-deny-execute-permissions-on-a-share-folder-in-windows)  
29. windows 7 \- Deny execution permission, allow read permission \- Super User, accessed October 23, 2025, [https://superuser.com/questions/544103/deny-execution-permission-allow-read-permission](https://superuser.com/questions/544103/deny-execution-permission-allow-read-permission)  
30. How To Enable File and Folder Auditing on Windows Server \- Lepide, accessed October 23, 2025, [https://www.lepide.com/how-to/enable-file-folder-access-auditing-windows-server.html](https://www.lepide.com/how-to/enable-file-folder-access-auditing-windows-server.html)  
31. How to configure file and folder access auditing in Windows \- Kaspersky Knowledge Base, accessed October 23, 2025, [https://support.kaspersky.com/common/diagnostics/15924](https://support.kaspersky.com/common/diagnostics/15924)  
32. How to use Security Auditing to track which Computer has accessed ..., accessed October 23, 2025, [https://learn.microsoft.com/en-us/answers/questions/2198494/how-to-use-security-auditing-to-track-which-comput](https://learn.microsoft.com/en-us/answers/questions/2198494/how-to-use-security-auditing-to-track-which-comput)  
33. How to Enable and Configure Audit Settings on Microsoft File server? \- eG Innovations, accessed October 23, 2025, [https://www.eginnovations.com/documentation/Microsoft-File-Server/How-to-Enable-and-Configure-Audit-Settings-on-Microsoft-File-Server.htm](https://www.eginnovations.com/documentation/Microsoft-File-Server/How-to-Enable-and-Configure-Audit-Settings-on-Microsoft-File-Server.htm)  
34. Enable controlled folder access \- Microsoft Defender for Endpoint, accessed October 23, 2025, [https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders](https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders)  
35. How to Configure Protected Folders for Controlled Folder Access in Windows 10 \- NinjaOne, accessed October 23, 2025, [https://www.ninjaone.com/blog/controlled-folder-access/](https://www.ninjaone.com/blog/controlled-folder-access/)  
36. Using Controlled Folder Access to Block Ransomware \- K2 Enterprises, accessed October 23, 2025, [https://www.k2e.com/tech-tips/using-controlled-folder-access-block-ransomware/](https://www.k2e.com/tech-tips/using-controlled-folder-access-block-ransomware/)  
37. Protect important folders from ransomware from encrypting your files ..., accessed October 23, 2025, [https://learn.microsoft.com/en-us/defender-endpoint/controlled-folders](https://learn.microsoft.com/en-us/defender-endpoint/controlled-folders)  
38. What you need to know about Microsoft Controlled Folder Access \- PDQ, accessed October 23, 2025, [https://www.pdq.com/blog/controlledfolders/](https://www.pdq.com/blog/controlledfolders/)