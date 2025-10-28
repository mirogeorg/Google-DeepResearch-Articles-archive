
# **A Multi-Layered Security Framework for Creating a Hardened Write-Only SMB Share on Windows Server**

## **Executive Summary**

This report provides a comprehensive, defense-in-depth methodology for creating a highly secure, write-only file share on a Windows Server platform. The primary objective is to configure a dedicated user account with the absolute minimum privileges required to deposit files into a network share, while simultaneously hardening the host server against remote exploits that could leverage this account. This approach is founded on the principle of least privilege, ensuring that every component, from the user account to the network protocols, is configured with the minimal necessary functionality to reduce the overall attack surface.

The framework is structured across seven distinct layers of security controls. The process begins with the foundational creation of an isolated, low-privilege local user account, stripped of all default group memberships and permissions. It then details the critical architecture of Windows access control, explaining the nuanced interaction between Share and NTFS permissions to construct a true "drop box" folder where users can write files but cannot list, read, or modify existing content.

Subsequent layers address remote attack vectors by hardening account logon rights, explicitly denying interactive and remote desktop access to prevent credential misuse. The report then focuses on securing the data-in-transit by mandating modern, secure Server Message Block (SMB) protocols, including the complete removal of the vulnerable SMBv1 and the enforcement of SMBv3 encryption. Network-level defenses are established using the Windows Defender Firewall to create a perimeter that restricts SMB access to only pre-approved, trusted IP addresses. The hardening process concludes by examining system-level services, providing guidance based on industry standards like the Center for Internet Security (CIS) Benchmarks to disable non-essential services that could otherwise serve as entry points for attackers. Finally, a rigorous verification protocol is provided to ensure all security controls are implemented correctly and are effective.

## **Section 1: Foundational Security: Principle of Least Privilege User Configuration**

The cornerstone of any secure system is the principle of least privilege. This principle dictates that any user, program, or process should have only the bare minimum permissions necessary to perform its specific, authorized function. For the purpose of creating a secure file drop location, this means establishing a user account that is fundamentally isolated and devoid of any rights beyond its single, narrow purpose. This initial step is the most critical in containing the potential impact of a credential compromise, as an attacker who gains control of the account will find themselves in a tightly constrained digital sandbox.

### **1.1 Creating a Dedicated, Low-Privilege Local User Account**

The first action is to create a new, standard local user account. This account will exist only on the file server and will not be part of any domain, limiting its scope of influence. While this can be accomplished through the graphical user interface (GUI), the recommended method for security, consistency, and auditability is to use PowerShell.

#### **Implementation (PowerShell \- Recommended)**

Using PowerShell and its Microsoft.PowerShell.LocalAccounts module ensures that the user creation process is scriptable, repeatable, and less prone to human error.1 A critical aspect of this method is the use of SecureString objects for password handling. This practice prevents the password from being stored as plain text in scripts, command history, or system memory, which is a significant security enhancement over passing passwords as simple strings.

The following PowerShell commands will create a secure, isolated local user:

PowerShell

\# Prompt for a password and store it as a SecureString object.  
\# This prevents the password from being exposed in plain text.  
$Password \= Read-Host \-AsSecureString

\# Define the parameters for the new user account in a hash table for clarity.  
$UserParams \= @{  
    Name \= 'SecureDropUser'  
    Password \= $Password  
    Description \= 'Limited-access account for write-only file drop share. This account has no interactive logon rights.'  
    PasswordNeverExpires \= $false  
    UserMayNotChangePassword \= $true  
}

\# Create the new local user with the specified parameters.  
New-LocalUser @UserParams

Each parameter in this command is deliberately chosen to enforce a secure baseline:

* **Name:** A descriptive name (SecureDropUser) makes the account's purpose clear during audits. Usernames cannot contain certain characters (", /, \\, \[, \], :, ;, |, \=, ,, \+, \*, ?, \<, \>, @) and cannot consist solely of periods or spaces.1  
* **Password:** The use of a SecureString object is paramount for protecting the credential during creation.  
* **Description:** A clear description provides essential context for other administrators and security auditors who may review the system's accounts.  
* **PasswordNeverExpires \= $false:** This ensures the account is subject to the server's password expiration policies, forcing periodic credential rotation. Setting passwords to never expire is a significant security risk.3  
* **UserMayNotChangePassword \= $true:** This setting prevents the user account itself from changing its own password, centralizing control with the administrator.

#### **Implementation (GUI \- for Reference)**

For administrators who prefer a graphical interface, the user can be created through the Local Users and Groups management console (lusrmgr.msc) or the Computer Management tool.3

1. Open **Computer Management** from the Start Menu or by running compmgmt.msc.  
2. Navigate to **System Tools \> Local Users and Groups \> Users**.  
3. Right-click in the right-hand pane and select **New User...**.  
4. Fill in the **User name** (SecureDropUser) and a strong, complex **Password**.  
5. Provide a **Full name** and **Description** for clarity.  
6. Crucially, configure the password options as follows:  
   * Uncheck **User must change password at next logon**. This is important because this is a service-like account without an interactive user to perform the change.  
   * Check **User cannot change password**. This aligns with the PowerShell parameter \-UserMayNotChangePassword $true.  
   * Leave **Password never expires** unchecked to ensure local policies are enforced.  
   * Leave **Account is disabled** unchecked.  
7. Click **Create** to finalize the account creation.

The choice between these two methods is not merely one of preference but carries strategic security implications. The GUI method is a manual, point-in-time action. It is susceptible to misconfiguration if an administrator overlooks a checkbox. In contrast, the PowerShell script codifies the exact, secure configuration. This script can be stored in version control, subjected to peer review, and deployed consistently across any number of servers, guaranteeing a reproducible and auditable security posture. For any environment where security and operational maturity are priorities, the PowerShell method is unequivocally superior.

### **1.2 Enforcing Account Security Policies**

The new user account's password strength is governed by the server's local or domain-level security policies. These policies define the minimum requirements for password complexity, length, and history. It is essential to ensure these policies are configured to a strong standard.

These settings can be viewed in the **Local Security Policy** editor (secpol.msc) under **Account Policies \> Password Policy**. A strong baseline, aligning with recommendations from security frameworks like the CIS Benchmarks, would include 7:

* **Enforce password history:** 24 or more passwords remembered.  
* **Maximum password age:** 60 days or less.  
* **Minimum password age:** 1 or more days.  
* **Minimum password length:** 14 or more characters.  
* **Password must meet complexity requirements:** Enabled.  
* **Store passwords using reversible encryption:** Disabled.

By creating the SecureDropUser account without special flags, it automatically becomes subject to these system-wide security requirements, ensuring its primary credential is not a weak link.

### **1.3 Validating Minimal Group Membership**

By default, Windows adds any new local user to the built-in Users group.8 This group is granted a baseline set of permissions, including potentially problematic rights like "Allow log on locally" and "Access this computer from the network." For our highly restricted account, these default privileges are excessive and represent an unnecessary security risk. To adhere strictly to the principle of least privilege, the SecureDropUser account must be removed from all groups.

This action fundamentally shifts the account's security posture from "allowed by default, must be explicitly denied" to "denied by default, must be explicitly allowed." This is a proactive hardening measure that neutralizes multiple potential attack vectors at their source. Even if an administrator later fails to apply an explicit "Deny" policy for a specific logon right, the user will still be blocked because it lacks the group membership that would grant the corresponding "Allow" right in the first place.

#### **Implementation**

1. Open **Computer Management** and navigate to **Local Users and Groups \> Users**.  
2. Double-click the SecureDropUser account to open its **Properties**.  
3. Go to the **Member Of** tab.  
4. Select the Users group from the list and click **Remove**.  
5. Click **OK**. The list of groups should now be empty.

Alternatively, this can be accomplished with PowerShell:

PowerShell

Remove-LocalGroupMember \-Group "Users" \-Member "SecureDropUser"

After this step, the SecureDropUser account is a truly isolated security principal, possessing no inherent rights or privileges on the system. It is now a blank slate upon which we can grant the single, specific permission it requires.

## **Section 2: Architecting Access Control: The Duality of Share and NTFS Permissions**

With a secure, isolated user account established, the next layer of defense involves architecting the access control mechanisms for the shared folder itself. In a Windows environment, this is a two-part system composed of **Share Permissions** and **NTFS Permissions**. Understanding the distinct role of each, and how they interact, is absolutely essential for implementing granular and effective security. Failure to correctly configure both layers can lead to unintended access or overly restrictive permissions that are difficult to troubleshoot.

### **2.1 A Deep Dive into Share vs. NTFS Permissions: Understanding the Interaction and Precedence**

Share and NTFS permissions serve different purposes and operate at different levels of the access control stack.

* **Share Permissions:** These permissions apply *only* when the folder is accessed over the network via its share name (e.g., \\\\SERVER\\ShareName). They do not apply to users who are logged on locally to the server and accessing the folder directly through the file system. Share permissions are relatively simple, offering only three levels of access: Full Control, Change, and Read.9 They apply to the entire shared folder and all its contents; you cannot set different share permissions for individual subfolders or files within the share.12  
* **NTFS Permissions:** These permissions are part of the New Technology File System (NTFS) and are applied directly to files and folders on the disk. They govern access regardless of whether the resource is being accessed locally or over the network. NTFS permissions are highly granular, offering a wide range of basic permissions (Full Control, Modify, Read & Execute, List Folder Contents, Read, Write) and an even more detailed set of advanced permissions that allow for fine-tuned control over specific actions.10

When a user connects to a network share, Windows evaluates both sets of permissions and enforces the **most restrictive** of the two. This is the golden rule of Windows file sharing security.11 For example, if a user is granted "Full Control" at the share level but only "Read" at the NTFS level, their effective permission when accessing the share will be "Read." Conversely, if they have "Read" at the share level and "Full Control" at the NTFS level, their effective permission will still be "Read," as the more restrictive share permission takes precedence for network access.9

The following table provides a comparative analysis of these two permission systems.

| Feature | Share Permissions | NTFS Permissions |
| :---- | :---- | :---- |
| **Scope of Application** | Applies only to access over the network. | Applies to both local and network access. |
| **Granularity** | Low (3 levels: Read, Change, Full Control). | High (6 basic levels, plus numerous advanced permissions). |
| **Point of Application** | Applied to the shared folder as a whole. | Can be applied to individual files and subfolders. |
| **File System Support** | Works with NTFS, FAT, and FAT32 file systems. | Works only with the NTFS file system. |
| **Best Practice Use Case** | Used as a broad gateway for network access. | Used for detailed, granular control of all resources. |

(Data sourced from 9)

### **2.2 Establishing the Network Share: Configuring Share-Level Permissions as a Gateway**

Given the limited granularity of share permissions and the complexity that arises from managing two separate sets of rules, the established industry best practice, recommended by Microsoft, is to use share permissions as a simple, broad gateway and to enforce all meaningful security controls at the more granular NTFS level.12 This approach centralizes access control logic in one place (NTFS), making the security model easier to manage, audit, and troubleshoot.9

The common implementation of this best practice is to set the share permission to Everyone \- Full Control. However, the Everyone group can be problematic in some environments and may trigger warnings from security scanners. A more refined and equally effective approach is to use the Authenticated Users group with the Change permission. The Authenticated Users group includes all users who have successfully authenticated with a local or domain account, which is a slightly more secure posture than Everyone. The Change permission (which includes the ability to read, execute, write, and delete) is sufficient to pass all access control decisions down to the NTFS layer, which will then apply the final, more restrictive rules.10

It is important to note that granting broad permissions at the share level does not grant users the ability to manage the share itself. Only members of the local Administrators or Server Operators groups can modify the share's properties, regardless of the permissions assigned for data access.12

#### **Implementation**

1. Create a new folder on an NTFS-formatted volume (e.g., D:\\SecureDrop).  
2. Right-click the folder and select **Properties**.  
3. Go to the **Sharing** tab and click **Advanced Sharing...**.  
4. Check the box for **Share this folder**.  
5. Enter a descriptive **Share name** (e.g., SecureDropShare).  
6. Click the **Permissions** button.  
7. By default, the Everyone group may be present with Read permissions. Click **Remove** to delete this entry.  
8. Click **Add...**, type Authenticated Users, and click **OK**.  
9. With Authenticated Users selected, check the **Allow** box for **Change**. This will also automatically check Read.  
10. Click **OK** to close the Permissions window, and **OK** again to close the Advanced Sharing window.

### **2.3 Implementing "Drop Box" Functionality with Advanced NTFS Permissions**

With the share acting as an open gateway, the NTFS permissions become the sole and absolute arbiter of access. To meet the user's specific requirement for a "write-only" drop box, where a user can deposit files but cannot see, read, or modify any other files in the directory, we must configure a precise set of advanced NTFS permissions.

#### **2.3.1 Disabling Inheritance to Enforce Explicit Control**

Folders on an NTFS volume inherit permissions from their parent directory by default.11 These inherited permissions are almost always too permissive for a secure folder and would grant unintended access (e.g., the Users group often has Read & execute rights inherited from the root of the drive). Therefore, the first and most critical step is to disable permission inheritance on our D:\\SecureDrop folder. This severs the link to the parent folder's permissions and allows us to build a unique, explicit Access Control List (ACL) from scratch.15

#### **Implementation**

1. Right-click the D:\\SecureDrop folder, select **Properties**, and go to the **Security** tab.  
2. Click the **Advanced** button to open the Advanced Security Settings window.  
3. Click the **Disable inheritance** button.  
4. A dialog box will appear. Select the option to **"Convert inherited permissions into explicit permissions on this object"**.15 This is the preferred choice as it provides a clean baseline of the existing permissions, which we can then modify. The alternative, "Remove all inherited permissions," would strip even administrative access, which is generally not desirable.  
5. Immediately after converting, review the list of permission entries. Select and **Remove** any entries that are not required, such as Users or Authenticated Users. The only entries that should remain are for SYSTEM, CREATOR OWNER, and Administrators, which should all have Full control.

#### **2.3.2 Configuring Granular Permissions for Write-Only Access**

Standard NTFS permissions like Write or Modify are insufficient for creating a true drop box. The Write permission allows file creation but also allows reading the files you create, while Modify includes full read, write, and delete capabilities.18 To achieve the desired behavior, we must configure a custom set of advanced permissions.

A particularly subtle but critical aspect of this configuration involves the CREATOR OWNER special identity. By default, Windows grants the user who creates a file or folder (CREATOR OWNER) full control over that object. In our drop box scenario, this would allow the SecureDropUser to read, modify, and delete the very file they just uploaded, violating the "drop-and-forget" principle. To enforce a true write-only model, the CREATOR OWNER entry must be removed from the parent folder's ACL.

#### **Implementation**

1. In the **Advanced Security Settings** window for the D:\\SecureDrop folder, ensure all unnecessary principals have been removed as described in the previous section.  
2. Select the CREATOR OWNER principal and click **Remove**.  
3. Click **Add**, then click **Select a principal**. Type SecureDropUser and click **OK**.  
4. In the **Permission Entry** window, set **Type** to Allow and **Applies to** to This folder, subfolders and files.  
5. Click **Show advanced permissions**.  
6. Carefully select only the following permissions. Do **not** select any others.  
   * Create files / write data  
   * Create folders / append data  
7. Click **OK**.  
8. The final ACL should now be configured. The SecureDropUser can create new files and folders but lacks the permissions to list the directory's contents (List folder / read data) or read any files, including their own.

The table below provides a precise blueprint for the final, hardened NTFS ACL for the drop folder.

| Principal | Type | Applies to | Basic Permissions | Advanced Permissions |
| :---- | :---- | :---- | :---- | :---- |
| SYSTEM | Allow | This folder, subfolders and files | Full control | Full control |
| Administrators | Allow | This folder, subfolders and files | Full control | Full control |
| SecureDropUser | Allow | This folder, subfolders and files | Special | Create files / write data, Create folders / append data |

(Data sourced from analysis of 11)

This precise combination of permissive share settings, disabled inheritance, and granular, advanced NTFS permissions creates a robust and secure file drop location that strictly adheres to the principle of least privilege.

## **Section 3: Mitigating Remote Threats: Hardening Account Logon Rights**

Creating a user with minimal file system permissions is only part of the solution. To effectively harden the system against remote exploits, it is imperative to also restrict *how* and *where* the user account's credentials can be used. A compromised password for an account that can only write to a single folder is a low-impact event. A compromised password for an account that can be used to establish an interactive remote session on the server is a critical security breach. This section focuses on using Windows User Rights Assignment policies to prevent the SecureDropUser account from being used for any form of interactive logon, thereby neutralizing a primary vector for lateral movement and privilege escalation.

### **3.1 Denying Interactive Logon to Prevent Console Access**

The "Deny log on locally" policy setting (SeDenyInteractiveLogonRight) is a powerful security control that prevents an account from being used to log on at the computer's console.20 This includes both the physical console and a virtual machine console session (e.g., through Hyper-V Manager). Applying this policy to the SecureDropUser ensures that its credentials cannot be used to gain an interactive shell on the server, even if an attacker is physically or virtually present at the console.

#### **Implementation**

1. Open the **Local Security Policy** editor by running secpol.msc.  
2. Navigate to **Local Policies \> User Rights Assignment**.  
3. In the right-hand pane, find and double-click the policy named **Deny log on locally**.  
4. Click **Add User or Group...**, type SecureDropUser, and click **OK**.  
5. Click **Apply** and then **OK** to save the change.

This policy change takes effect the next time the user attempts to log on. It is important to understand that a "Deny" right always supersedes an "Allow" right.20 Even if the SecureDropUser were a member of a group that is granted the "Allow log on locally" right, this explicit denial would take precedence and block the logon attempt.

This creates a redundant security control. As established in Section 1, the SecureDropUser was removed from the default Users group, which is typically on the "Allow log on locally" list. This already prevents local logon by default. Adding an explicit "Deny" right provides a second layer of defense. If a future administrator mistakenly adds the SecureDropUser back to the Users group, the explicit "Deny" policy will still prevent a local logon, thus protecting the system against configuration drift and human error.

### **3.2 Denying Remote Desktop Logon to Block Terminal Services**

While denying local logon is crucial, the most likely vector for a remote attacker is through Remote Desktop Protocol (RDP). The "Deny log on through Remote Desktop Services" policy (SeDenyRemoteInteractiveLogonRight) specifically prevents an account from establishing an RDP session.22 This is a critical hardening step for any account that does not require administrative access.

#### **Implementation**

1. Within the **User Rights Assignment** console (secpol.msc), find and double-click the policy named **Deny log on through Remote Desktop Services**.  
2. Click **Add User or Group...**, type SecureDropUser, and click **OK**.  
3. Click **Apply** and then **OK**.

Assigning this right is a recommended countermeasure against attackers using compromised credentials to gain remote access and potentially run malicious software to elevate their privileges.24 The CIS Benchmarks also recommend applying this restriction to local accounts to reduce the attack surface.25

### **3.3 Security Impact Analysis: Preventing Lateral Movement and Remote Shells**

The combination of these User Rights Assignment policies directly addresses the user's primary concern: defending against remote exploits. If an attacker on a client machine manages to compromise the SecureDropUser credentials (e.g., through a phishing attack or malware), their ability to leverage those credentials is now severely curtailed.

* They cannot RDP into the server.  
* They cannot log on at the server's console.

This effectively prevents them from escalating a simple credential theft into a full-blown interactive session on the server, which is a common tactic for lateral movement within a network. Without an interactive shell, an attacker cannot easily run reconnaissance tools, deploy additional malware, or attempt privilege escalation attacks on the server itself.

#### **Additional Hardening Recommendation**

For an even more secure configuration, the SecureDropUser should also be added to the **"Deny access to this computer from the network"** policy. At first glance, this seems counterintuitive, as the user needs network access for the file share. However, the Server service (LanmanServer) that handles SMB connections grants a specific type of network logon right that is separate from this general policy. By denying general network access, other potential network-based logon types, such as those used by PowerShell Remoting (WinRM) or other remote administration tools, are blocked by default. The SMB service functions as an explicit exception to this rule, ensuring the account can do its job and absolutely nothing else over the network.26 This configuration embodies the "deny by default" security posture.

## **Section 4: Securing Data in Transit: SMB Protocol Hardening**

Securing the user account and file system permissions protects data at rest. The next critical layer of defense is securing the data while it is in transit across the network. This is accomplished by hardening the Server Message Block (SMB) protocol itself. Modern servers must be configured to use only the latest, most secure versions of the SMB protocol and to enforce encryption to ensure the confidentiality and integrity of the data being transferred.

### **4.1 Eradicating Legacy Risks: Auditing and Disabling the SMBv1 Protocol**

The SMB version 1.0 (SMBv1) protocol is a legacy technology dating back to the 1980s. It is notoriously insecure, lacking modern security features and containing critical vulnerabilities that have been actively exploited by widespread ransomware attacks such as WannaCry and Petya.27 For this reason, SMBv1 is deprecated and should be disabled on all modern Windows systems. Its continued presence represents a significant and unnecessary security risk.

#### **Implementation (PowerShell)**

PowerShell provides a straightforward method for auditing the status of SMBv1 and disabling it if it is found to be active. These commands must be run in an elevated PowerShell session.

1. **Audit the current SMBv1 state:** This command queries the optional features of the operating system to determine if the SMBv1 protocol is installed and enabled.  
   PowerShell  
   Get-WindowsOptionalFeature \-Online \-FeatureName SMB1Protocol

   If State is Enabled, the protocol must be disabled.  
2. **Disable the SMBv1 protocol:** This command removes the SMBv1 feature. A system restart is required for the change to take full effect.28  
   PowerShell  
   Disable-WindowsOptionalFeature \-Online \-FeatureName SMB1Protocol

   After running this command and restarting the server, the system will no longer be able to communicate using the insecure SMBv1 dialect, protecting it from known exploits targeting that protocol.

### **4.2 Enforcing Confidentiality: Requiring SMBv3 Encryption**

Modern versions of the SMB protocol (specifically SMB 3.0 and newer) support strong, end-to-end encryption using the AES-128-GCM algorithm.29 Enforcing SMB encryption ensures that all data transferred between the client and the server is confidential and protected from eavesdropping attacks. This is particularly important on networks that may not be fully trusted.

A key benefit of enforcing encryption is that it also implicitly forces the use of the SMB 3.x dialect. Since encryption is a feature exclusive to SMB 3.0 and later, any client that does not support it (e.g., an older system attempting to connect with SMBv2) will be unable to establish a connection.30 This provides a more elegant and forward-compatible method for ensuring modern protocol usage than attempting to manually disable older protocols like SMBv2, which is not possible as SMBv2 and SMBv3 share the same protocol stack.27

#### **Implementation (PowerShell \- Server-Side)**

To configure the server to require encryption for all incoming SMB connections, use the Set-SmbServerConfiguration cmdlet:

PowerShell

Set-SmbServerConfiguration \-EncryptData $true

This command modifies the server's global SMB configuration to mandate encryption. No restart is required for this setting to take effect.

#### **Implementation (Group Policy \- Enterprise-Wide Enforcement)**

For consistent enforcement across multiple servers and clients, Group Policy is the preferred method.31

1. Open the **Group Policy Management Editor**.  
2. Navigate to **Computer Configuration \> Administrative Templates \> Network \> Lanman Workstation**.  
3. Find and enable the policy **"Require Encryption"**. This forces client machines to use encryption for all outbound SMB connections.  
4. For server-side enforcement, navigate to **Computer Configuration \> Administrative Templates \> Network \> Lanman Server**.  
5. Find and enable the policy **"Encrypt data"**.

Deploying these policies ensures that both clients and servers in your environment are configured to protect SMB traffic.

### **4.3 Verification of Secure SMB Protocol Configuration**

After applying these hardening measures, it is essential to verify that they are active and functioning correctly.

1. **Verify Server Configuration:** Run the following PowerShell command on the server to confirm that SMBv1 is disabled and encryption is required.  
   PowerShell  
   Get-SmbServerConfiguration | Select EnableSMB1Protocol, EncryptData

   The expected output is EnableSMB1Protocol : False and EncryptData : True.  
2. **Verify Live Connection:** After a client has connected to the hardened share, run the following command on the server to inspect the active SMB session.  
   PowerShell  
   Get-SmbConnection | ft ServerName,ShareName,Dialect,Encrypted

   The output should show that the connection's Dialect is 3.0 or higher (e.g., 3.1.1) and that the Encrypted status is True.27 This provides real-time confirmation that the secure protocol and encryption are being used as intended.

## **Section 5: Network Perimeter Defense: Implementing Firewall Restrictions**

While user permissions and protocol security are vital, a robust defense-in-depth strategy requires a network-level perimeter. The Windows Defender Firewall with Advanced Security provides a powerful, host-based firewall that can be configured to restrict network access to the SMB service. By creating specific rules, an administrator can ensure that only authorized computers on the network are even capable of initiating a connection to the file share, effectively making the server invisible to unauthorized systems and significantly reducing its attack surface.

### **5.1 Configuring Windows Defender Firewall for Inbound SMB Traffic**

By default, when file and printer sharing is enabled, Windows creates a set of broad inbound firewall rules that allow SMB traffic (over TCP port 445\) from any computer on the same network subnet.33 For a hardened server, this default rule is too permissive. The proper security posture is one of "deny by default, allow by exception." This involves disabling the generic, default rule and creating a new, highly specific rule that only allows traffic under the exact conditions required.

This approach is more resilient and auditable than simply modifying the default rule. A custom rule with a descriptive name (e.g., "Allow Inbound SMB for SecureDrop from Trusted Clients") makes the security policy explicit and is less likely to be inadvertently changed by system updates or other administrative actions.

#### **Implementation**

1. Open **Windows Defender Firewall with Advanced Security** by running wf.msc.  
2. In the left pane, click on **Inbound Rules**.  
3. Scroll down and locate the active rule named **"File and Printer Sharing (SMB-In)"** for the current network profile (typically "Domain" or "Private").  
4. Right-click the rule and select **Disable Rule**. Repeat for any other active, broad file sharing rules.  
5. In the right-hand Actions pane, click **New Rule...**.  
6. Select **Custom** and click **Next**.  
7. Select **All programs** and click **Next**.  
8. In the **Protocol and Ports** screen:  
   * Set **Protocol type** to TCP.  
   * Set **Local port** to Specific Ports, and enter 445\.  
   * Set **Remote port** to All Ports.  
   * Click **Next**.

### **5.2 Implementing IP-Based Scoping to Restrict Network Access**

The most critical part of the new firewall rule is the **Scope** configuration. This is where access is restricted to a specific list of trusted IP addresses. Any connection attempt originating from an IP address not on this list will be silently dropped by the firewall before it can even reach the SMB service, providing a powerful layer of network isolation.34

#### **Implementation (continued from above)**

9. In the **Scope** screen, under the section **"Which remote IP addresses does this rule apply to?"**, select **These IP addresses**.  
10. Click **Add...** and enter the specific IP address or subnet of the client machine(s) that should be allowed to access the share. Add each trusted source individually.  
11. Click **Next**.  
12. In the **Action** screen, ensure **Allow the connection** is selected and click **Next**.  
13. In the **Profile** screen, select the network profiles to which this rule should apply (typically **Domain** and **Private**). Uncheck **Public** unless there is a specific, well-understood reason to allow access from untrusted networks.  
14. Click **Next**.  
15. In the **Name** screen, provide a descriptive name and description for the rule, such as:  
    * **Name:** Allow Inbound SMB for SecureDrop from Trusted Clients  
    * **Description:** Allows inbound TCP port 445 traffic only from specific client IPs for the SecureDrop file share.  
16. Click **Finish**.

With this configuration in place, the server's SMB service is now shielded by the firewall. Even if an attacker on an unauthorized machine has valid credentials for the SecureDropUser account, their connection attempts will be blocked at the network layer, neutralizing the threat before authentication is even attempted.

## **Section 6: Minimizing the Attack Surface: System Service Hardening**

A fully hardened server is defined not only by its secure configurations but also by what is *not* running on it. Every active service on an operating system presents a potential attack surface—a set of code that could contain vulnerabilities and be exploited by an attacker. The practice of system service hardening involves systematically disabling or removing any services that are not absolutely essential for the server's specific role. For a dedicated file server, this means eliminating services related to user experience, remote management protocols that are not in use, and legacy features.

### **6.1 Applying Security Baselines: An Introduction to CIS Benchmarks**

Rather than relying on guesswork to determine which services are safe to disable, administrators should leverage industry-standard security baselines. The Center for Internet Security (CIS) Benchmarks are a set of globally recognized and consensus-developed best practices for securely configuring systems.36 These benchmarks provide prescriptive guidance, including detailed recommendations for system service configurations, tailored to different security levels (e.g., Level 1 for core security hygiene, Level 2 for high-security environments).36 Adhering to these benchmarks helps ensure a robust and defensible security posture.

### **6.2 Recommended Services to Disable on a Dedicated File Server**

The following table provides a curated list of Windows services that are generally not required for a dedicated file server's operation. The recommendations are derived from Microsoft's own security hardening guides and principles outlined in security community best practices.38 Before disabling any service, administrators should confirm that it is not required by any third-party monitoring or backup software in their specific environment.

A critical point of clarification is the Server service (LanmanServer). While some hardening guides for *workstations* recommend disabling this service to prevent the machine from hosting shares 40, this advice is dangerously incorrect when applied to a file server. The Server service is the core component that handles all incoming SMB/CIFS requests; disabling it will render the file server completely non-functional. This highlights the absolute necessity of interpreting hardening guidance within the context of the server's intended role.

| Service Name | Default Startup | Recommended State | Rationale for Disabling on a File Server |
| :---- | :---- | :---- | :---- |
| Spooler (Print Spooler) | Automatic | Disabled | Manages print jobs. Has a history of severe vulnerabilities (e.g., PrintNightmare) and is a high-value target for privilege escalation. Unnecessary on a non-print server.38 |
| RemoteRegistry (Remote Registry) | Manual | Disabled | Allows remote modification of the Windows Registry. An unnecessary risk when modern tools like PowerShell Remoting or Windows Admin Center are used for management. |
| TermService (Remote Desktop Services) | Manual | Disabled | Enables RDP access. If administrative access is handled through other means (e.g., VM console, PowerShell), this service should be disabled to close a major remote access vector.38 |
| WinRM (Windows Remote Management) | Automatic | Disabled | The service for PowerShell Remoting. If not actively used for administration, it should be disabled to reduce the remote management attack surface.38 |
| Fax (Fax Service) | Manual | Disabled | Enables faxing capabilities. This is a legacy service that is obsolete in nearly all modern environments and should be disabled.38 |
| Bluetooth Support Service | Manual | Disabled | Manages Bluetooth connections. Unnecessary on a server, which typically has no Bluetooth hardware, and removes any risk from potential Bluetooth protocol vulnerabilities.38 |
| Xbox Live Services (All) | Manual | Disabled | Provides authentication and game save functionality for Xbox Live. These have no function on a server and should be disabled.39 |

### **6.3 Analysis of High-Risk Services and Their Relevance**

Certain services carry a disproportionately high level of risk and warrant special attention.

* **Print Spooler (Spooler):** This service has historically been a prime target for attackers. Vulnerabilities like "PrintNightmare" have demonstrated that the service can be exploited to achieve remote code execution and full system-level privilege escalation.38 Because the service runs with high privileges (SYSTEM) and is designed to load third-party drivers, a flaw in its implementation can have catastrophic consequences. On a server that does not manage printers, there is no functional reason for this service to be running, and disabling it immediately eliminates a significant and well-known attack vector.  
* **Remote Registry (RemoteRegistry):** While legitimate administrative tools can use this service, it is also highly valuable to an attacker during the reconnaissance phase of an attack. A compromised account with sufficient privileges could use the remote registry service to query for sensitive configuration information, identify security weaknesses, or modify registry keys to disable security software or establish persistence mechanisms. Modern administrative practices have largely superseded the need for this service, making its disablement a prudent security measure.

By systematically reviewing and disabling these and other non-essential services, an administrator can significantly shrink the server's attack surface, leaving fewer avenues for a potential attacker to exploit.

## **Section 7: Comprehensive Verification and Auditing**

The final stage in this multi-layered security framework is a rigorous verification process. Each security control must be tested to confirm that it is not only configured correctly but also produces the intended outcome. This end-to-end testing protocol validates the entire security posture, from user permissions to network firewall rules, ensuring there are no gaps or misconfigurations.

### **7.1 End-to-End Testing Protocol for the Write-Only Share**

This test plan should be executed from a client machine that is on the firewall's allowed IP address list. A separate machine not on the list will be needed for the final network test.

1. **Write Test (Expected Outcome: Success)**  
   * **Action:** From the authorized client machine, map the network share using the SecureDropUser credentials.  
   * **Verification:** Attempt to copy a new file into the share. The file copy operation must complete successfully.  
2. **List Directory Test (Expected Outcome: Failure)**  
   * **Action:** After successfully copying the file, attempt to view the contents of the network share using File Explorer or the dir command in a command prompt.  
   * **Verification:** The attempt must fail with an "Access is denied" error. The user should not be able to see the file they just created or any other files in the directory.  
3. **Read Test (Expected Outcome: Failure)**  
   * **Action:** Attempt to open the file that was just copied into the share *from the share itself* (e.g., by trying to open \\\\SERVER\\SecureDropShare\\newfile.txt).  
   * **Verification:** The open operation must fail with an "Access is denied" error.  
4. **Overwrite/Delete Test (Expected Outcome: Failure)**  
   * **Action:** An administrator should place a test file (e.g., test\_file.txt) in the D:\\SecureDrop directory on the server. Using the SecureDropUser credentials from the client, attempt to delete or overwrite this file.  
   * **Verification:** Both the delete and overwrite attempts must fail with an "Access is denied" error.  
5. **Remote Logon Test (Expected Outcome: Failure)**  
   * **Action:** Attempt to initiate a Remote Desktop Connection to the server using the SecureDropUser credentials.  
   * **Verification:** The connection must be explicitly rejected with a message indicating the user does not have the right to sign in, confirming that the "Deny log on through Remote Desktop Services" policy is effective.26  
6. **Untrusted IP Test (Expected Outcome: Failure)**  
   * **Action:** From a client machine whose IP address is *not* on the firewall rule's allowed list, attempt to map the network share (e.g., using net use \* \\\\SERVER\\SecureDropShare).  
   * **Verification:** The connection attempt should fail to establish, likely resulting in a timeout error. This confirms that the Windows Defender Firewall is correctly blocking unauthorized network traffic at the perimeter.

### **7.2 A Final Security Configuration Checklist**

This checklist provides a concise summary of all the critical security controls implemented in this framework. It can be used for a final audit to ensure that all hardening steps have been completed.

| Layer | Control | Verification Method | Status |
| :---- | :---- | :---- | :---- |
| **User Account** | Local user SecureDropUser exists. | Get-LocalUser \-Name "SecureDropUser" | ☐ |
|  | Account is not a member of any groups. | Check "Member Of" tab in user properties. | ☐ |
|  | Password policies are enforced. | Review secpol.msc \-\> Account Policies. | ☐ |
| **Permissions** | Share permission for Authenticated Users is Change. | Check Advanced Sharing \-\> Permissions. | ☐ |
|  | NTFS inheritance is disabled on the target folder. | Check Security \-\> Advanced. | ☐ |
|  | CREATOR OWNER principal is removed from the ACL. | Check Security \-\> Advanced. | ☐ |
|  | SecureDropUser has only "Create" advanced NTFS permissions. | Review SecureDropUser permissions in Advanced Security. | ☐ |
| **Logon Rights** | SecureDropUser is in "Deny log on locally" policy. | Check secpol.msc \-\> User Rights Assignment. | ☐ |
|  | SecureDropUser is in "Deny log on through Remote Desktop Services" policy. | Check secpol.msc \-\> User Rights Assignment. | ☐ |
| **SMB Protocol** | SMBv1 is disabled. | Get-WindowsOptionalFeature \-Online \-FeatureName SMB1Protocol | ☐ |
|  | SMB encryption is required on the server. | Get-SmbServerConfiguration | Select EncryptData | ☐ |
| **Firewall** | Default "File and Printer Sharing" rule is disabled. | Check wf.msc \-\> Inbound Rules. | ☐ |
|  | Custom "Allow SMB" rule exists. | Check wf.msc \-\> Inbound Rules. | ☐ |
|  | Custom rule is scoped to trusted remote IP addresses only. | Check Scope tab of the custom rule's properties. | ☐ |
| **Services** | Non-essential services (Print Spooler, Remote Registry, etc.) are disabled. | Check services.msc. | ☐ |

By successfully completing this verification protocol and checklist, an administrator can be confident that they have implemented a robust, multi-layered security solution that effectively creates a hardened, write-only file share while minimizing the server's exposure to remote threats.

#### **Works cited**

1. New-LocalUser \- PowerShell \- Microsoft Learn, accessed October 23, 2025, [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/new-localuser?view=powershell-5.1](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/new-localuser?view=powershell-5.1)  
2. Create Local User with PowerShell \- MonoVM, accessed October 23, 2025, [https://monovm.com/blog/create-local-user-with-powershell/](https://monovm.com/blog/create-local-user-with-powershell/)  
3. Create and delete local users on Windows Server \- Rackspace Technology, accessed October 23, 2025, [https://docs.rackspace.com/docs/creating-and-deleting-local-users-on-windows-server](https://docs.rackspace.com/docs/creating-and-deleting-local-users-on-windows-server)  
4. www.server-world.info, accessed October 23, 2025, [https://www.server-world.info/en/note?os=Windows\_Server\_2022\&p=initial\_conf\&f=1\#:\~:text=Run%20%5BServer%20Manager%5D%20and%20Open,%5D%20%2D%20%5BComputer%20Management%5D.\&text=Right%2DClick%20%5BUsers%5D%20under,and%20select%20%5BNew%20User%5D.\&text=Input%20UserName%20and%20Password%20for,intems%20are%20optional%20to%20set.](https://www.server-world.info/en/note?os=Windows_Server_2022&p=initial_conf&f=1#:~:text=Run%20%5BServer%20Manager%5D%20and%20Open,%5D%20%2D%20%5BComputer%20Management%5D.&text=Right%2DClick%20%5BUsers%5D%20under,and%20select%20%5BNew%20User%5D.&text=Input%20UserName%20and%20Password%20for,intems%20are%20optional%20to%20set.)  
5. Adding a Local User Account to Windows Server 2019 \- ComputingForGeeks, accessed October 23, 2025, [https://computingforgeeks.com/adding-local-user-account-to-windows-server-2019/](https://computingforgeeks.com/adding-local-user-account-to-windows-server-2019/)  
6. How to Create a User Account on Window 2016 Server? \- AccuWebHosting, accessed October 23, 2025, [https://manage.accuwebhosting.com/knowledgebase/3108/How-to-Create-a-User-Account-on-Window-2016-Server.html](https://manage.accuwebhosting.com/knowledgebase/3108/How-to-Create-a-User-Account-on-Window-2016-Server.html)  
7. CIS Microsoft Windows Server 2022 Benchmark, accessed October 23, 2025, [https://rayasec.com/wp-content/uploads/CIS-Benchmark/Microsoft-Windows-Server/CIS\_Microsoft\_Windows\_Server\_2022\_Benchmark\_v4.0.0.pdf](https://rayasec.com/wp-content/uploads/CIS-Benchmark/Microsoft-Windows-Server/CIS_Microsoft_Windows_Server_2022_Benchmark_v4.0.0.pdf)  
8. Local Accounts | Microsoft Learn, accessed October 23, 2025, [https://learn.microsoft.com/en-us/windows/security/identity-protection/access-control/local-accounts](https://learn.microsoft.com/en-us/windows/security/identity-protection/access-control/local-accounts)  
9. NTFS Permissions vs. Share Permissions – What's the Difference? \- tenfold, accessed October 23, 2025, [https://www.tenfold-security.com/en/ntfs-permissions-and-share-permissions-whats-the-difference/](https://www.tenfold-security.com/en/ntfs-permissions-and-share-permissions-whats-the-difference/)  
10. NTFS vs Share Permissions: Key Differences and Best Practices | Netwrix, accessed October 23, 2025, [https://netwrix.com/en/resources/blog/ntfs-vs-share-permissions/](https://netwrix.com/en/resources/blog/ntfs-vs-share-permissions/)  
11. PowerEdge: NTFS File Folder and Share Permissions in Windows | Dell US, accessed October 23, 2025, [https://www.dell.com/support/kbdoc/en-us/000137238/understanding-file-and-folder-permissions-in-windows](https://www.dell.com/support/kbdoc/en-us/000137238/understanding-file-and-folder-permissions-in-windows)  
12. Your Guide to NTFS Vs. Share Permissions Best Practices \- Global Knowledge, accessed October 23, 2025, [https://www.globalknowledge.com/us-en/resources/resource-library/articles/your-guide-to-ntfs-vs-share-permissions-best-practices/](https://www.globalknowledge.com/us-en/resources/resource-library/articles/your-guide-to-ntfs-vs-share-permissions-best-practices/)  
13. NTFS Permissions vs Share: Everything You Need to Know \- Varonis, accessed October 23, 2025, [https://www.varonis.com/blog/ntfs-permissions-vs-share](https://www.varonis.com/blog/ntfs-permissions-vs-share)  
14. Enable or Disable Inherited Permissions in Windows 10 \- Winaero, accessed October 23, 2025, [https://winaero.com/enable-disable-inherited-permissions-windows-10/](https://winaero.com/enable-disable-inherited-permissions-windows-10/)  
15. how do you deactivate or change shared permissions inheritance? : r/sysadmin \- Reddit, accessed October 23, 2025, [https://www.reddit.com/r/sysadmin/comments/11aucfj/how\_do\_you\_deactivate\_or\_change\_shared/](https://www.reddit.com/r/sysadmin/comments/11aucfj/how_do_you_deactivate_or_change_shared/)  
16. How to Configure Inherited Permissions for Files and Folders in Windows \- NinjaOne, accessed October 23, 2025, [https://www.ninjaone.com/blog/how-to-configure-inherited-permissions/](https://www.ninjaone.com/blog/how-to-configure-inherited-permissions/)  
17. remove all users except groups from NTFS permission. \- Microsoft Learn, accessed October 23, 2025, [https://learn.microsoft.com/en-us/answers/questions/1063734/remove-all-users-except-groups-from-ntfs-permissio](https://learn.microsoft.com/en-us/answers/questions/1063734/remove-all-users-except-groups-from-ntfs-permissio)  
18. Modify vs Write Permissions: What's the Difference? (2025 Guide) \- Albus Bit, accessed October 23, 2025, [https://albusbit.com/blog/windows-permission-differences/](https://albusbit.com/blog/windows-permission-differences/)  
19. A Complete Guide to Windows NTFS Permissions (2025) \- 2BrightSparks, accessed October 23, 2025, [https://www.2brightsparks.com/resources/articles/a-basic-introduction-to-ntfs-permissions.html](https://www.2brightsparks.com/resources/articles/a-basic-introduction-to-ntfs-permissions.html)  
20. Deny log on locally | Microsoft Learn, accessed October 23, 2025, [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/dn221948(v=ws.11)](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/dn221948\(v=ws.11\))  
21. Deny log on locally \- Windows 10 | Microsoft Learn, accessed October 23, 2025, [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/deny-log-on-locally](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/deny-log-on-locally)  
22. Deny user or group logon via RDP \- Windows Server \- Microsoft Learn, accessed October 23, 2025, [https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/deny-user-permissions-to-logon-to-rd-session-host](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/deny-user-permissions-to-logon-to-rd-session-host)  
23. learn.microsoft.com, accessed October 23, 2025, [https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/deny-user-permissions-to-logon-to-rd-session-host\#:\~:text=Computer%20Configuration%20%7C%20Windows%20Settings%20%7C%20Security,Select%20ok.](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/deny-user-permissions-to-logon-to-rd-session-host#:~:text=Computer%20Configuration%20%7C%20Windows%20Settings%20%7C%20Security,Select%20ok.)  
24. Deny log on through Remote Desktop Services \- Windows 10 | Microsoft Learn, accessed October 23, 2025, [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/deny-log-on-through-remote-desktop-services](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/deny-log-on-through-remote-desktop-services)  
25. 2.2.27 (L1) Ensure 'Deny log on through Remote Desktop Service... | Tenable®, accessed October 23, 2025, [https://www.tenable.com/audits/items/CIS\_Microsoft\_Windows\_Server\_2016\_v3.0.0\_L1\_MS.audit:2281a36f87e7f7e27d46e8a55de4b3ec](https://www.tenable.com/audits/items/CIS_Microsoft_Windows_Server_2016_v3.0.0_L1_MS.audit:2281a36f87e7f7e27d46e8a55de4b3ec)  
26. Blocking Remote Access for Local Accounts by Group Policy, accessed October 23, 2025, [https://www.vkernel.ro/blog/blocking-remote-access-for-local-accounts-by-group-policy](https://www.vkernel.ro/blog/blocking-remote-access-for-local-accounts-by-group-policy)  
27. How to Check, Enable or Disable SMB Protocol Versions on Windows, accessed October 23, 2025, [https://woshub.com/smb-1-0-support-in-windows-server-2012-r2/](https://woshub.com/smb-1-0-support-in-windows-server-2012-r2/)  
28. Detect, enable, and disable SMBv1, SMBv2, and SMBv3 in Windows ..., accessed October 23, 2025, [https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/detect-enable-and-disable-smbv1-v2-v3](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/detect-enable-and-disable-smbv1-v2-v3)  
29. PowerScale: How to disable/enable SMB encryption along with other information in SMB2/SMB3 versions \- Dell Technologies, accessed October 23, 2025, [https://www.dell.com/support/kbdoc/en-us/000206558/how-to-disable-enable-smb-encryption-in-isilon-along-with-additional-information-on-smb2-smb3-verisons](https://www.dell.com/support/kbdoc/en-us/000206558/how-to-disable-enable-smb-encryption-in-isilon-along-with-additional-information-on-smb2-smb3-verisons)  
30. Is it possible to Enable ONLY SMB3, while disabling SMB1 and SMB2 on Windows 10 21H2? : r/sysadmin \- Reddit, accessed October 23, 2025, [https://www.reddit.com/r/sysadmin/comments/1jecfxr/is\_it\_possible\_to\_enable\_only\_smb3\_while/](https://www.reddit.com/r/sysadmin/comments/1jecfxr/is_it_possible_to_enable_only_smb3_while/)  
31. Disable SMB Client Encryption Requirement in Windows 11 | NinjaOne, accessed October 23, 2025, [https://www.ninjaone.com/blog/disable-smb-client-encryption-requirement-in-windows-11/](https://www.ninjaone.com/blog/disable-smb-client-encryption-requirement-in-windows-11/)  
32. Disable SMB Client Encryption Requirement in Windows 11 ..., accessed October 23, 2025, [https://ninjaone.com/blog/disable-smb-client-encryption-requirement-in-windows-11/](https://ninjaone.com/blog/disable-smb-client-encryption-requirement-in-windows-11/)  
33. Windows File Sharing with SMB: Port 445, 139, 138, and 137 Explained, accessed October 23, 2025, [https://petri.com/smb-port-445-139-138-137/](https://petri.com/smb-port-445-139-138-137/)  
34. Preventing SMB traffic from lateral connections and entering or ..., accessed October 23, 2025, [https://support.microsoft.com/en-us/topic/preventing-smb-traffic-from-lateral-connections-and-entering-or-leaving-the-network-c0541db7-2244-0dce-18fd-14a3ddeb282a](https://support.microsoft.com/en-us/topic/preventing-smb-traffic-from-lateral-connections-and-entering-or-leaving-the-network-c0541db7-2244-0dce-18fd-14a3ddeb282a)  
35. How to configure the Windows Firewall to allow only specific IP ..., accessed October 23, 2025, [https://manage.accuwebhosting.com/knowledgebase/2984/How-to-configure-the-Windows-Firewall-to-allow-only-specific-IP-Address-to-connect-your-ports.html](https://manage.accuwebhosting.com/knowledgebase/2984/How-to-configure-the-Windows-Firewall-to-allow-only-specific-IP-Address-to-connect-your-ports.html)  
36. Center for Internet Security (CIS) Benchmarks \- Microsoft Compliance, accessed October 23, 2025, [https://learn.microsoft.com/en-us/compliance/regulatory/offering-cis-benchmark](https://learn.microsoft.com/en-us/compliance/regulatory/offering-cis-benchmark)  
37. CIS Microsoft Windows Server Benchmarks, accessed October 23, 2025, [https://www.cisecurity.org/benchmark/microsoft\_windows\_server](https://www.cisecurity.org/benchmark/microsoft_windows_server)  
38. How to Disable Windows Services That Could Be a Security Risk \- Fortect, accessed October 23, 2025, [https://www.fortect.com/how-to-guides/how-to-disable-windows-services-that-could-be-a-security-risk/](https://www.fortect.com/how-to-guides/how-to-disable-windows-services-that-could-be-a-security-risk/)  
39. Security guidelines for system services in Windows Server 2016 ..., accessed October 23, 2025, [https://learn.microsoft.com/en-us/windows-server/security/windows-services/security-guidelines-for-disabling-system-services-in-windows-server](https://learn.microsoft.com/en-us/windows-server/security/windows-services/security-guidelines-for-disabling-system-services-in-windows-server)  
40. 5.27 Ensure 'Server (LanmanServer)' is set to 'Disabled ... \- Tenable, accessed October 23, 2025, [https://www.tenable.com/audits/items/CIS\_MS\_Windows\_10\_Enterprise\_Level\_2\_v1.12.0.audit:ca3d2fbef0e5106e5358c740a9b9e0ee](https://www.tenable.com/audits/items/CIS_MS_Windows_10_Enterprise_Level_2_v1.12.0.audit:ca3d2fbef0e5106e5358c740a9b9e0ee)