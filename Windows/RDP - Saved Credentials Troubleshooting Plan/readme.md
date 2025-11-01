
# **An In-Depth Analysis and Resolution Guide for Remote Desktop Protocol Saved Credential Failures**

## **Executive Summary: Navigating RDP Authentication Failures in Managed Environments**

The failure of saved Remote Desktop Protocol (RDP) credentials for specific machines within a managed Active Directory environment is a common yet complex issue. The primary symptom—a "login attempt failed" message that is immediately resolved by manually re-entering the identical password—points away from simple credential corruption or user error. Instead, it indicates a systemic conflict within the authentication and credential delegation process. This report provides an exhaustive analysis of this problem, demonstrating that its roots are almost always found in misconfigured security policies, either on the client initiating the connection, the server receiving it, or through the influence of domain-level Group Policy Objects (GPOs).

The investigation will proceed through a structured methodology, beginning with foundational diagnostics to capture precise error data from server-side event logs. It will then deconstruct the intricate web of GPOs that govern RDP behavior, focusing on two critical domains: client-side credential delegation policies, which control how and when a workstation is permitted to send saved credentials, and server-side security enforcement policies, which can mandate re-authentication regardless of client settings. Finally, the report will address advanced security layers, such as Credential Security Support Provider (CredSSP) and Windows Defender Credential Guard, which are increasingly prevalent in modern operating systems and can override traditional policy settings, leading to these exact symptoms. The ultimate objective is to provide not only a direct resolution but also a durable framework for diagnosing and preventing such authentication anomalies in the future.

## **Section 1: Foundational Diagnostics and Initial Triage**

Before delving into complex policy analysis, a baseline must be established by ruling out fundamental configuration issues. This initial triage focuses on capturing precise error data, verifying essential permissions, and ensuring a clean state for all subsequent troubleshooting. This methodical approach transforms the investigation from guesswork into a data-driven process.

### **1.1. Analyzing Logon Failures with Windows Event Viewer**

The generic "login attempt failed" message displayed by the RDP client is a superficial symptom that obscures the true cause of the failure. The detailed reason for the authentication denial is logged exclusively on the target server, making the Windows Event Viewer the single most critical diagnostic tool.

Failed RDP authentication attempts are recorded in the **Windows Logs \> Security** log as **Event ID 4625**.1 Analysis of this event provides a wealth of information crucial for diagnosis. Upon opening the event details, several fields are of paramount importance:

* **Logon Type:** For RDP connections, this will be **Type 3** (Network) or **Type 10** (RemoteInteractive).  
* **Account For Which Login Failed:** This confirms the security principal (user account) that was denied.  
* **Failure Information:** This section contains the most valuable data.  
  * **Failure Reason:** A textual description of why the logon failed.  
  * **Status and Sub Status:** Hexadecimal codes that provide a specific, machine-readable reason for the failure.

Common failure reasons that point toward policy or permission issues include "Unknown user name or bad password," "User not allowed to log on at this computer," "The user has not been granted the requested logon type at this machine," or "Account restriction".2 An explicit message indicating a policy denial immediately shifts the focus of the investigation from credential validity to Group Policy conflicts. Capturing this specific failure reason is the first and most important step in narrowing the scope of potential causes.

### **1.2. Verifying Core Permissions and Group Memberships**

While the success of a manual logon suggests that basic permissions are in place, it is essential to formally verify them. A restrictive GPO, such as a Deny policy, can override basic permissions and could be the source of the inconsistency across machines. Two layers of permissions must be validated on the target server.

First, the user account must be a member of the appropriate local group. By default, only members of the local **Administrators** group and the **Remote Desktop Users** group are permitted to establish RDP sessions.2 This can be checked using the Local Users and Groups snap-in (lusrmgr.msc) on the target machine.

Second, the relevant groups must be granted specific user rights through the Local Security Policy (secpol.msc) or an overriding GPO. The two critical User Rights Assignments to inspect are:

1. **Allow log on through Remote Desktop Services:** This policy, located under Computer Configuration\\Windows Settings\\Security Settings\\Local Policies\\User Rights Assignment, must explicitly contain the Administrators and Remote Desktop Users groups.5 If this policy is defined by a GPO and does not include these groups, RDP access will be denied.  
2. **Access this computer from the network:** While this policy is broader, it is also required for network-based logons like RDP. The default configuration typically includes permissive groups like Everyone, Users, and Administrators, but in hardened environments, this list may be restricted. If the Remote Desktop Users group is not present or implicitly included, it can cause connection failures.3

The fact that the issue is isolated to specific machines strongly implies that a GPO is applying a different set of permissions or restrictions to the non-working computers compared to the working ones.

### **1.3. Systematic Credential Cache and Profile Cleansing**

To eliminate the possibility of corrupted or outdated cached data interfering with the authentication process, a complete purge of all stored RDP-related information on the client machine is a necessary diagnostic step. This ensures that subsequent connection attempts are made from a clean slate.

A thorough cleansing involves four distinct actions on the client machine:

1. **Windows Credential Manager:** The Credential Manager is the primary storage location for saved passwords. It can be opened by running rundll32.exe keymgr.dll,KRShowKeyMgr. All stored credentials related to RDP connections, which are identifiable by the TERMSRV/ prefix, should be removed from both the **Windows Credentials** and **Generic Credentials** vaults.6  
2. **Registry Cleanup:** The RDP client stores connection history and user hints in the registry. Using the Registry Editor (regedit.exe), navigate to the key HKEY\_CURRENT\_USER\\Software\\Microsoft\\Terminal Server Client. Within this key, delete all MRU\* (Most Recently Used) values from the Default subkey and delete the server-specific subkeys under the Servers key.7  
3. **Delete Default.rdp File:** The RDP client saves the settings of the last connection to a hidden file named Default.rdp located in the user's Documents folder. Deleting this file removes any lingering connection-specific configurations.6  
4. **Command-Line Purge:** For a more rapid and comprehensive removal, the cmdkey utility can be used from an administrative command prompt. The following command will list all TERMSRV credentials, which can then be deleted individually or via a script.10  
   Code snippet  
   cmdkey /list  
   For /F "tokens=1,2 delims= " %G in ('cmdkey /list ^| findstr "target=TERMSRV"') do cmdkey /delete %H

Performing this full purge definitively rules out local data corruption as a cause and forces the RDP client to establish a new, clean connection profile.

## **Section 2: Deconstructing Group Policy Conflicts: The Client-Side Configuration**

The policies configured on the client machine—the computer initiating the RDP connection—play a decisive role in whether saved credentials can be transmitted to a remote host. By default, Windows imposes security restrictions that prevent the automatic delegation of credentials to servers whose identities cannot be fully verified, a common scenario in environments with multiple domains, workgroups, or complex network segmentation. This section details the client-side policies that must be configured to permit this delegation.

### **2.1. The Critical Role of Credential Delegation Policies**

Credential Delegation policies are the mechanism by which an administrator can create explicit exceptions to the default security posture, defining a set of trusted servers to which the client is allowed to send saved credentials. These policies are located within the Group Policy Editor (gpedit.msc for local policy or gpmc.msc for domain policy) at the following path:  
Computer Configuration\\Administrative Templates\\System\\Credentials Delegation.12  
To enable the use of saved credentials, particularly for servers that may rely on NTLM authentication or are outside the client's direct trust boundary, the following key policies must be set to **Enabled**:

* Allow delegating saved credentials with NTLM-only server authentication  
* Allow delegating saved credentials  
* Allow delegating default credentials 14

Enabling these policies is only the first step; they must also be configured with a list of target servers to which delegation is permitted.

### **2.2. Configuring TERMSRV/\* for Trusted and Untrusted Hosts**

When a delegation policy is enabled, clicking the "Show..." button in the policy's properties reveals a list where target servers must be specified. This list acts as an explicit allow-list, and the RDP client will only delegate credentials to servers that match an entry. The entries must use the Service Principal Name (SPN) format for Remote Desktop Services, which is TERMSRV/hostname.20

The following formats are accepted:

* TERMSRV/server1.corp.com: Allows delegation to a single, specific server identified by its Fully Qualified Domain Name (FQDN).  
* TERMSRV/\*.corp.com: Allows delegation to any server within the corp.com domain.  
* TERMSRV/\*: A broad wildcard that permits credential delegation to any server. While useful for troubleshooting, this configuration is the least secure and should be used with caution.12

Two critical details must be observed for this to function correctly. First, the TERMSRV prefix must be in uppercase.12 Second, the server name used in the RDP client's "Computer" field must *exactly* match the value specified in the policy. For instance, if the policy lists TERMSRV/server1.corp.com, attempting to connect using the server's IP address or its short name (server1) will cause the delegation to fail, as the strings will not match.12 The inconsistency experienced with "certain remote computers" may stem from this very issue, where working connections are to servers within a trusted domain (not requiring explicit delegation) and failing connections are to servers in untrusted domains or workgroups that do require these policies to be correctly configured.

### **2.3. Identifying and Disabling Prohibitive Client-Side Policies**

In addition to policies that permit delegation, other client-side policies can be configured to explicitly forbid the saving or use of passwords, overriding any user-selected options in the RDP client. These policies are located at:  
Computer Configuration\\Administrative Templates\\Windows Components\\Remote Desktop Services\\Remote Desktop Connection Client  
The two policies to inspect here are:

* **Do not allow passwords to be saved:** This policy must be set to **Disabled** or **Not Configured**. If it is **Enabled**, the RDP client will be blocked from storing any passwords in the Credential Manager, and the "Allow me to save credentials" checkbox may be grayed out or ineffective.12  
* **Prompt for credentials on the client computer:** This policy should also be set to **Disabled** or **Not Configured** to allow for seamless, non-interactive logon with saved credentials.12

These settings are often enabled in high-security environments as a matter of policy to prevent credentials from being stored on user workstations.

### **2.4. Investigating Global Security Policies**

Beyond RDP-specific settings, a broader, system-wide security policy can prevent the Credential Manager from storing any network-related passwords, which would inherently affect RDP's ability to save credentials. This powerful policy is located at:  
Computer Configuration\\Windows Settings\\Security Settings\\Local Policies\\Security Options  
The policy is named **Network access: Do not allow storage of passwords and credentials for network authentication**. For saved RDP credentials to work, this policy must be set to **Disabled**.12 If this policy is **Enabled**, it serves as a master switch that prevents the Credential Manager from caching any credentials for domain authentication, rendering all RDP-specific "allow" policies moot.22 An error code of 0x80070520 when trying to save credentials manually in Credential Manager is a direct symptom of this policy being enabled.12

## **Section 3: Deconstructing Group Policy Conflicts: The Server-Side Configuration**

Even if the client is perfectly configured to send saved credentials, the target server has the final authority to accept or reject them. Server-side security policies can be configured to enforce re-authentication for every connection, a configuration that precisely matches the symptoms of the reported issue. These policies are often implemented as part of security hardening baselines and can be the root cause of the problem.

### **3.1. The "Always prompt for password upon connection" Policy: A Primary Suspect**

This server-side policy is arguably the most likely cause of the issue. When enabled, it instructs the Remote Desktop Session Host to ignore any credentials sent by the client and to force an interactive logon prompt for every connection. This behavior perfectly explains why saved credentials fail on the first attempt, yet a manual entry of the same password succeeds—the user is simply complying with the server's mandatory prompt.

This policy is located on the target server within the Group Policy Editor at:  
Computer Configuration\\Administrative Templates\\Windows Components\\Remote Desktop Services\\Remote Desktop Session Host\\Security  
To allow the use of saved credentials, the **Always prompt for password upon connection** policy must be set to **Disabled** or **Not Configured**.6 If this policy is **Enabled**, the server will always present a login screen.25

This setting can also be verified directly in the server's registry. The policy corresponds to the DWORD value fPromptForPassword under the key HKEY\_LOCAL\_MACHINE\\SOFTWARE\\Policies\\Microsoft\\Windows NT\\Terminal Services. A value of 1 indicates the policy is enabled, while a value of 0 indicates it is disabled.12

The rationale for enabling this policy is rooted in security hardening. Standards such as the Center for Internet Security (CIS) Benchmarks recommend enabling this setting to mitigate the risk of lateral movement by an attacker.26 If an attacker compromises a user's workstation, they cannot use saved .rdp files on that machine to pivot to other servers without knowing the user's password.27 This creates a direct conflict between security posture and operational convenience. The fact that this issue affects only "certain" machines strongly suggests that a hardening GPO containing this setting has been applied to a specific Organizational Unit (OU) containing the non-working servers.

### **3.2. Network Level Authentication (NLA) and Its Implications**

Network Level Authentication (NLA) is a security feature that requires a user to authenticate to the remote server *before* a full RDP session is established and system resources are allocated. This helps protect the server from denial-of-service attacks and malicious users.28 NLA is enabled by default on modern Windows operating systems.

The relevant GPO setting is **Require user authentication for remote connections by using Network Level Authentication**, located in the same Security section of the Remote Desktop Session Host policies.28

While NLA failures typically produce a different, pre-connection error message, misconfigurations can contribute to authentication problems. For example, NLA relies on the Credential Security Support Provider (CredSSP), and issues with CredSSP (as discussed in the next section) can manifest as NLA failures.3 Furthermore, if a user's account is not a member of the Remote Desktop Users group, the NLA pre-authentication check will fail.3 While disabling NLA is a valid troubleshooting step to isolate the problem, it is not recommended as a permanent solution because it lowers the security of the RDP connection by allowing unauthenticated users to reach the server's login screen.3

## **Section 4: Advanced Authentication and Security Layer Conflicts**

In modern Windows environments, traditional Group Policy settings are not the only factors governing authentication. Advanced, often hardware-assisted, security features can override standard policies and introduce subtle but powerful restrictions on how credentials are handled. These features, designed to combat modern threats like credential theft, are a frequent cause of saved credential failures on up-to-date and security-hardened systems.

### **4.1. Understanding and Resolving CredSSP Mismatches**

The Credential Security Support Provider (CredSSP) is a protocol that RDP uses to securely forward user credentials from the client to the server for NLA.32 In 2018, Microsoft released a security update to patch a critical vulnerability in CredSSP (CVE-2018-0886). This update introduced a new policy setting that can cause connection failures if the client and server have mismatched patch levels.33

The characteristic symptom of this issue is a very specific error message: "An authentication error has occurred. The function requested is not supported... This could be due to CredSSP encryption oracle remediation".32 This error occurs when a security policy on one end of the connection deems the other end to be insecure due to its patch status.

The controlling policy, **Encryption Oracle Remediation**, is located on both the client and server at Computer Configuration\\Administrative Templates\\System\\Credentials Delegation. It has three protection levels:

* **Force Updated Clients:** The most secure setting. It blocks connections between patched and unpatched systems in either direction.  
* **Mitigated:** The default setting. It allows a patched server to accept connections from unpatched clients but blocks a patched client from connecting to an unpatched server.  
* **Vulnerable:** The least secure setting. It allows connections between systems regardless of their patch status but exposes the connection to the original vulnerability.32

The ideal and recommended solution is to ensure that all client and server machines are fully updated with the latest Windows patches, which aligns their CredSSP versions and resolves the conflict.33 As a temporary workaround, the policy on the patched client can be set to **Enabled** with a **Protection Level** of **Vulnerable**. This will allow the connection to the unpatched server but should only be used as an interim measure due to the significant security risk it introduces.32

### **4.2. The Impact of Windows Defender Credential Guard**

Windows Defender Credential Guard is a virtualization-based security feature available in Enterprise editions of Windows 10 and 11\. It uses hardware virtualization to isolate and protect sensitive secrets like NTLM password hashes and Kerberos Ticket Granting Tickets, making them inaccessible to attackers even if they gain system-level privileges.35

A significant side effect of Credential Guard is that it actively blocks the use of saved RDP credentials that are stored as a "Domain Password" credential type. This is precisely the type of credential that is created when a user checks the "Remember me" box in the standard RDP client GUI.9 This behavior perfectly matches the user's report: the credential is saved, but the security subsystem on the client machine prevents it from being delegated, causing the initial logon to fail. The subsequent manual password entry works because it is an interactive logon, not a delegation of a saved secret.

The solution is **not** to disable Credential Guard, as doing so would remove a critical layer of defense against credential theft attacks.9 Instead, the correct workaround is to store the RDP password in a format that Credential Guard permits for delegation. This is achieved by creating a **"Generic"** credential instead of a "Domain Password" credential. This can be done in two ways:

1. **Via Credential Manager GUI:** Manually open Credential Manager, select "Windows Credentials," and click "Add a generic credential." Enter the target server's address (prefixed with TERMSRV/), the username, and the password.14  
2. Via Command Line (cmdkey): This is the more reliable and scriptable method. Open an administrative command prompt and use the following command syntax:  
   cmdkey /generic:TERMSRV/\<targetname\_or\_IP\> /user:\<username\> /pass:\<password\>.9

Creating the credential as "Generic" makes it compatible with Credential Guard's security boundaries, allowing it to be used for RDP authentication. This is a highly likely solution if the client machines experiencing the issue are modern, security-hardened Windows Enterprise endpoints, as the problem may have appeared after a feature update or deployment of new hardware with virtualization-based security enabled by default.

## **Section 5: A Systematic Troubleshooting Methodology**

To efficiently resolve the RDP credential issue, a systematic approach is required. This involves comparing the configurations of working and non-working systems to pinpoint the exact policy discrepancy, followed by a logical workflow that addresses the most likely causes in order.

### **5.1. Establishing a Baseline: Comparing Effective Policies**

The core of effective troubleshooting is comparison. Since the issue affects only a subset of machines, there is an opportunity to compare a known-good configuration with a failing one. The goal is to identify the specific policy setting that differs between the two.

The **Resultant Set of Policy (RSoP)** is the cumulative effect of all local and domain GPOs applied to a computer and user. The gpresult command-line tool is the most effective way to generate a comprehensive report of these settings.

1. **On a failing client machine:** Open a command prompt as an administrator and run the command gpresult /h C:\\FailingClientReport.html. This will generate a detailed HTML report of all applied policies for the computer and the logged-on user.30  
2. **On a working client machine:** Run the same command: gpresult /h C:\\WorkingClientReport.html.  
3. **On a non-working target server:** Execute gpresult /h C:\\FailingServerReport.html.  
4. **On a working target server:** Execute gpresult /h C:\\WorkingServerReport.html.

By comparing these reports side-by-side, an administrator can quickly identify differences in the key policies discussed in Sections 2, 3, and 4\. For a more automated comparison, Microsoft's **Policy Analyzer** tool, part of the Security Compliance Toolkit, can be used. This tool can import and compare GPO backups, allowing for a detailed analysis of discrepancies in settings between different policies.37

### **5.2. A Step-by-Step Resolution Workflow**

The following workflow provides a logical sequence of diagnostic and resolution steps, starting with data collection and progressing from the most common to the most advanced causes.

1. **Diagnose on the Server:** Begin on the non-working target server. Open **Event Viewer** and filter the **Security** log for **Event ID 4625**. Analyze the **Failure Reason** and **Status** code to get a specific reason for the logon denial.  
2. **Cleanse the Client:** On the client machine, perform a full credential and profile cleanse as detailed in Section 1.3. This includes clearing the Credential Manager, RDP registry keys, and the Default.rdp file.  
3. **Verify Server Permissions:** On the target server, confirm the user account is a member of the **Remote Desktop Users** or **Administrators** group and that these groups are included in the **Allow log on through Remote Desktop Services** user right.  
4. **Compare Effective Policies:** Generate gpresult reports from working and non-working client/server pairs. Compare the reports, focusing on the policies listed in the table below.  
5. **Test for Credential Guard Conflict (Client-Side):** If the client is running a modern version of Windows Enterprise, assume Credential Guard may be active. Use the cmdkey /generic command to add the RDP credential as a "Generic" type and test the connection.  
6. **Check Client-Side Delegation:** If connecting to a server in an untrusted domain or workgroup, review the client's **Credentials Delegation** policies. Ensure they are enabled and the target server is correctly listed with the TERMSRV/ prefix.  
7. **Check Server-Side Prompt Policy:** If all else fails, the most likely cause is the server-side policy forcing a prompt. On the target server, check the **Always prompt for password upon connection** policy and ensure it is **Disabled**.  
8. **Investigate CredSSP Mismatch:** If the error message explicitly mentions "CredSSP" or "Encryption Oracle Remediation," investigate the patch levels of the client and server and adjust the **Encryption Oracle Remediation** policy as a temporary workaround if necessary.

### **Table 1: Diagnostic and Resolution Checklist**

| Diagnostic Step | Tool/Location (on relevant machine) | What to Check | Expected State for Success | Source(s) |
| :---- | :---- | :---- | :---- | :---- |
| **Server-Side Diagnostics** |  |  |  |  |
| Check Server Event Log | eventvwr.msc on Target Server | Security Log, Event ID 4625 | No policy denial codes in Failure Reason. | 1 |
| Verify User Group Membership | lusrmgr.msc on Target Server | Remote Desktop Users or Administrators group | User is a member of one of these groups. | 2 |
| Verify User Rights | secpol.msc on Target Server | Allow log on through Remote Desktop Services | Remote Desktop Users group is included. | 5 |
| Check Mandatory Prompt Policy | gpedit.msc on Target Server | Always prompt for password upon connection | Disabled or Not Configured. | 12 |
| **Client-Side Diagnostics** |  |  |  |  |
| Purge Credential Cache | control userpasswords2, regedit.exe, cmdkey on Client | TERMSRV/ credentials, RDP registry keys | All related cached items are removed. | 7 |
| Check Credential Delegation | gpedit.msc on Client | Allow delegating saved credentials... policies | Enabled with TERMSRV/\* or specific server listed. | 12 |
| Check Prohibitive Policies | gpedit.msc on Client | Do not allow passwords to be saved | Disabled or Not Configured. | 12 |
| Check Global Credential Storage | gpedit.msc on Client | Network access: Do not allow storage of... | Disabled. | 16 |
| **Advanced Security Layers** |  |  |  |  |
| Test Credential Guard Workaround | cmd.exe on Client | Use cmdkey /generic:... to add credential | Credential is created as "Generic" type. | 9 |
| Check for CredSSP Error | RDP Client Error Message | Error text mentions "CredSSP" or "Encryption Oracle" | No CredSSP error present. | 32 |

## **Section 6: Strategic Recommendations and Best Practices**

Resolving the immediate RDP issue is only part of the solution. A strategic approach to Active Directory design and policy management is necessary to prevent similar, inconsistent behaviors from recurring. This involves creating a logical structure for policy application and making informed decisions that balance security requirements with operational needs.

### **6.1. Structuring Active Directory OUs for Consistent RDP Policy Application**

The root cause of inconsistent policy application across a set of computers is often a disorganized or flat Active Directory Organizational Unit (OU) structure. When servers with different roles and security requirements reside in the same OU, it becomes impossible to apply granular policies without affecting all of them.

The recommended best practice is to design an OU structure that groups objects with similar management and security needs. For example:

* Create parent OUs for major resource types, such as "Servers" and "Workstations."  
* Within the "Servers" OU, create child OUs based on server role or environment, such as "Domain Controllers," "Application Servers," "Database Servers," and "Web Servers".40  
* Place computer accounts into their respective OUs. Avoid mixing user and computer accounts in the same OU, as this simplifies policy targeting by allowing entire sections of a GPO (User or Computer Configuration) to be disabled.42

This hierarchical structure allows administrators to link specific GPOs at the appropriate level. For instance, a highly restrictive GPO that enables Always prompt for password upon connection can be linked exclusively to the "Domain Controllers" OU, while a more permissive policy for general application servers can be linked to the "Application Servers" OU. This ensures that policies are applied predictably and consistently, eliminating the "some machines work, some don't" scenario.

### **6.2. Security Hardening vs. Operational Efficiency**

The conflict at the heart of this issue is often between security hardening mandates and the operational efficiency required by IT administrators. Policies like Always prompt for password upon connection are not arbitrary; they are recommended by security frameworks like the CIS Benchmarks to defend against lateral movement by attackers who have compromised a workstation.26 Similarly, Credential Guard is a powerful defense against pass-the-hash and other credential theft attacks.35

Simply disabling these security features to restore convenience is often not an acceptable solution in a security-conscious organization. A more nuanced approach is required:

* **Choose the Right Policy for the Job:** If the goal is to prevent users from saving RDP passwords on their workstations, it is better to enforce this using the client-side GPO Do not allow passwords to be saved.12 This achieves the security goal without disrupting administrators who may need to use saved credentials from secured management workstations.  
* **Implement Compensating Controls:** For highly privileged access, relying on saved RDP credentials on a general-purpose workstation is a poor security practice. Instead of weakening server security policies, organizations should implement stronger controls for administrative access. This could include:  
  * **Privileged Access Management (PAM) Solutions:** Tools like CyberArk manage and inject credentials for privileged sessions, removing the need for administrators to know or save passwords. Interestingly, these tools often require that the Always prompt for password upon connection policy be disabled to function correctly, providing a strong business case for creating a GPO exception for servers managed by the PAM solution.25  
  * **Privileged Access Workstations (PAWs):** Using dedicated, hardened workstations for all administrative tasks creates a secure environment from which to manage servers, reducing the risk associated with saved credentials.  
  * **Multi-Factor Authentication (MFA):** Integrating MFA into the RDP process, often via an RD Gateway, adds a critical layer of security that far outweighs the protection offered by forcing password prompts.28

By engaging in a risk-based discussion, IT administrators can advocate for solutions that meet security objectives without unduly hindering operational requirements, leading to a more secure and manageable environment.

#### **Works cited**

1. How Do You Check Logs for Failed RDP Login Attempts? \- AccuWebHosting, accessed September 28, 2025, [https://manage.accuwebhosting.com/knowledgebase/4215/How-Do-You-Check-Logs-for-Failed-RDP-Login-Attempts.html](https://manage.accuwebhosting.com/knowledgebase/4215/How-Do-You-Check-Logs-for-Failed-RDP-Login-Attempts.html)  
2. Troubleshooting access denied and user not authorized issues in RDS \- Windows Server, accessed September 28, 2025, [https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/troubleshooting-access-denied-and-user-not-authorized-rds-issues](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/troubleshooting-access-denied-and-user-not-authorized-rds-issues)  
3. User can't authenticate or must authenticate twice \- Windows Server ..., accessed September 28, 2025, [https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/cannot-authenticate-must-authenticate-twice](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/cannot-authenticate-must-authenticate-twice)  
4. RDP Credentials are NOT working even though they are correct : r/techsupport \- Reddit, accessed September 28, 2025, [https://www.reddit.com/r/techsupport/comments/r8xba6/rdp\_credentials\_are\_not\_working\_even\_though\_they/](https://www.reddit.com/r/techsupport/comments/r8xba6/rdp_credentials_are_not_working_even_though_they/)  
5. \[Four Ways\] Fix “Your Credentials Did Not Work” on Remote Desktop, accessed September 28, 2025, [https://www.anyviewer.com/how-to/your-credentials-did-not-work-2578.html](https://www.anyviewer.com/how-to/your-credentials-did-not-work-2578.html)  
6. How do I stop Remote Desktop from prompting for username and password twice, accessed September 28, 2025, [https://superuser.com/questions/712848/how-do-i-stop-remote-desktop-from-prompting-for-username-and-password-twice](https://superuser.com/questions/712848/how-do-i-stop-remote-desktop-from-prompting-for-username-and-password-twice)  
7. How to View and Clear RDP Connections History in Windows ..., accessed September 28, 2025, [https://woshub.com/how-to-clear-rdp-connections-history/](https://woshub.com/how-to-clear-rdp-connections-history/)  
8. 2 Tested Ways to Clear RDP Connection History on Windows 10, 11 \- AnyViewer, accessed September 28, 2025, [https://www.anyviewer.com/how-to/clear-rdp-connection-history-2578.html](https://www.anyviewer.com/how-to/clear-rdp-connection-history-2578.html)  
9. "Windows Defender Credential Guard does not allow using saved credentials" for RDP connections? \- Super User, accessed September 28, 2025, [https://superuser.com/questions/1756354/windows-defender-credential-guard-does-not-allow-using-saved-credentials-for-r](https://superuser.com/questions/1756354/windows-defender-credential-guard-does-not-allow-using-saved-credentials-for-r)  
10. How to Clear RDP Connections History in Windows? | by bonguides.com \- Medium, accessed September 28, 2025, [https://medium.com/@bonguides25/how-to-clear-rdp-connections-history-in-windows-cf0ffb67f344](https://medium.com/@bonguides25/how-to-clear-rdp-connections-history-in-windows-cf0ffb67f344)  
11. Remove entries from Remote Desktop Connection Computer \- Windows Server | Microsoft Learn, accessed September 28, 2025, [https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remove-entries-from-remote-desktop-connection-computer](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remove-entries-from-remote-desktop-connection-computer)  
12. Fix: Saved RDP Credentials Didn't Work on Windows | Windows OS ..., accessed September 28, 2025, [https://woshub.com/fix-saved-rdp-credentials-windows/](https://woshub.com/fix-saved-rdp-credentials-windows/)  
13. Remote Desktop doesn't allow saved credentials \- InflexionDesk, accessed September 28, 2025, [https://support.inflexionpoint.ai/portal/en/kb/articles/remote-desktop-doesn-t-allowed-saved-credentials](https://support.inflexionpoint.ai/portal/en/kb/articles/remote-desktop-doesn-t-allowed-saved-credentials)  
14. Easily Fixed: Remote Desktop Not Saving Credentials \- AnyViewer, accessed September 28, 2025, [https://www.anyviewer.com/how-to/remote-desktop-not-saving-credentials-0007.html](https://www.anyviewer.com/how-to/remote-desktop-not-saving-credentials-0007.html)  
15. remote desktop \- Why won't RDP accept my stored credentials, and ..., accessed September 28, 2025, [https://serverfault.com/questions/767159/why-wont-rdp-accept-my-stored-credentials-and-makes-me-manually-enter-it-every](https://serverfault.com/questions/767159/why-wont-rdp-accept-my-stored-credentials-and-makes-me-manually-enter-it-every)  
16. How to Allow Saved Credentials for RDP Connection? \- TheITBros.com, accessed September 28, 2025, [https://theitbros.com/enable-saved-credentials-usage-rdp/](https://theitbros.com/enable-saved-credentials-usage-rdp/)  
17. How to Allow Saved Credentials for RDP Connection? \- BMI SmartCloud, accessed September 28, 2025, [https://bmismartcloud.com/how-to-allow-saved-credentials-for-rdp-connection/](https://bmismartcloud.com/how-to-allow-saved-credentials-for-rdp-connection/)  
18. Using Group Policy Settings to Save Credentials \- QK Support, accessed September 28, 2025, [https://support.qikkids.com.au/s/article/Using-Group-Policy-Settings-to-Save-Credentials](https://support.qikkids.com.au/s/article/Using-Group-Policy-Settings-to-Save-Credentials)  
19. Windows 7 \- Group Policy \- Allow RDP Credentials to be Saved \- Server Fault, accessed September 28, 2025, [https://serverfault.com/questions/140363/windows-7-group-policy-allow-rdp-credentials-to-be-saved](https://serverfault.com/questions/140363/windows-7-group-policy-allow-rdp-credentials-to-be-saved)  
20. ADMX\_CredSsp Policy CSP \- Microsoft Learn, accessed September 28, 2025, [https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-admx-credssp](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-admx-credssp)  
21. How to Launch Native RDP Connections in PAM \- Securden, accessed September 28, 2025, [https://www.securden.com/privileged-access-management/help/account-management/how-to-launch-native-rdp-connections-in-pam.html](https://www.securden.com/privileged-access-management/help/account-management/how-to-launch-native-rdp-connections-in-pam.html)  
22. Network access Do not allow storage of passwords and credentials for network authentication \- Windows 10 | Microsoft Learn, accessed September 28, 2025, [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/network-access-do-not-allow-storage-of-passwords-and-credentials-for-network-authentication](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/network-access-do-not-allow-storage-of-passwords-and-credentials-for-network-authentication)  
23. Connecting to the remote desktop requires clicking login every time？ \- Microsoft Learn, accessed September 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/1525388/connecting-to-the-remote-desktop-requires-clicking](https://learn.microsoft.com/en-us/answers/questions/1525388/connecting-to-the-remote-desktop-requires-clicking)  
24. The server's authentication policy does not allow saved credentials issue while connecting to target \- CyberArk, accessed September 28, 2025, [https://community.cyberark.com/s/question/0D52J000085WXBqSAO/the-servers-authentication-policy-does-not-allow-saved-credentials-issue-while-connecting-to-target](https://community.cyberark.com/s/question/0D52J000085WXBqSAO/the-servers-authentication-policy-does-not-allow-saved-credentials-issue-while-connecting-to-target)  
25. PSM \- Prompts for password on target Windows Servers \- CyberArk, accessed September 28, 2025, [https://community.cyberark.com/s/article/00003290](https://community.cyberark.com/s/article/00003290)  
26. 18.9.58.3.9.1 (L1) Ensure 'Always prompt for password upon connection' is set to 'Enabled', accessed September 28, 2025, [https://www.tenable.com/audits/items/CIS\_MS\_Windows\_10\_Enterprise\_Level\_1\_v1.5.0.audit:25d32c49e1735167236957de1a43377f](https://www.tenable.com/audits/items/CIS_MS_Windows_10_Enterprise_Level_1_v1.5.0.audit:25d32c49e1735167236957de1a43377f)  
27. PSM rationale to set "Always prompt for password upon connection" to disabled \- CyberArk, accessed September 28, 2025, [https://community.cyberark.com/s/question/0D5Ht00009tAIuoKAG/psm-rationale-to-set-always-prompt-for-password-upon-connection-to-disabled](https://community.cyberark.com/s/question/0D5Ht00009tAIuoKAG/psm-rationale-to-set-always-prompt-for-password-upon-connection-to-disabled)  
28. Securing Remote Desktop (RDP) for System Administrators, accessed September 28, 2025, [https://security.berkeley.edu/education-awareness/securing-remote-desktop-rdp-system-administrators](https://security.berkeley.edu/education-awareness/securing-remote-desktop-rdp-system-administrators)  
29. RDP Security: The Impact of Secure Defaults & Legacy Protocols \- runZero, accessed September 28, 2025, [https://www.runzero.com/blog/rdp-security/](https://www.runzero.com/blog/rdp-security/)  
30. Using Group Policy to Enable Remote Desktops \- Trio MDM, accessed September 28, 2025, [https://www.trio.so/blog/group-policy-enable-remote-desktop/](https://www.trio.so/blog/group-policy-enable-remote-desktop/)  
31. CredSSP encryption oracle remediation error using RDP \- Microsoft Q\&A, accessed September 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/3908360/credssp-encryption-oracle-remediation-error-using](https://learn.microsoft.com/en-us/answers/questions/3908360/credssp-encryption-oracle-remediation-error-using)  
32. How to Fix Encryption Oracle Remediation Errors (CredSSP) \- V2 Cloud, accessed September 28, 2025, [https://v2cloud.com/blog/how-to-fix-encryption-oracle-remediation-credssp](https://v2cloud.com/blog/how-to-fix-encryption-oracle-remediation-credssp)  
33. Error CredSSP encryption oracle remediation when you try to RDP ..., accessed September 28, 2025, [https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-machines/windows/credssp-encryption-oracle-remediation](https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-machines/windows/credssp-encryption-oracle-remediation)  
34. Remote Desktop Connection Encryption Error, accessed September 28, 2025, [https://usdkb.sandiego.edu/s/article/Remote-Desktop-Connection-Encryption-Error](https://usdkb.sandiego.edu/s/article/Remote-Desktop-Connection-Encryption-Error)  
35. Remote Credential Guard | Microsoft Learn, accessed September 28, 2025, [https://learn.microsoft.com/en-us/windows/security/identity-protection/remote-credential-guard](https://learn.microsoft.com/en-us/windows/security/identity-protection/remote-credential-guard)  
36. Your system administrator does not allow the use of saved credentials to log on to the remote computer \- Server Fault, accessed September 28, 2025, [https://serverfault.com/questions/396722/your-system-administrator-does-not-allow-the-use-of-saved-credentials-to-log-on](https://serverfault.com/questions/396722/your-system-administrator-does-not-allow-the-use-of-saved-credentials-to-log-on)  
37. Two new Group Policy tools from Microsoft \- View Blog, accessed September 28, 2025, [https://www.mdmandgpanswers.com/blogs/view-blog/two-new-group-policy-tools-from-microsoft](https://www.mdmandgpanswers.com/blogs/view-blog/two-new-group-policy-tools-from-microsoft)  
38. Is there an easy way to compare group policy objects between different domains? \- Reddit, accessed September 28, 2025, [https://www.reddit.com/r/sysadmin/comments/95956d/is\_there\_an\_easy\_way\_to\_compare\_group\_policy/](https://www.reddit.com/r/sysadmin/comments/95956d/is_there_an_easy_way_to_compare_group_policy/)  
39. HOWTO: Export and Compare Security Policies between 2 different ..., accessed September 28, 2025, [https://pleasework.robbievance.net/howto-export-and-compare-security-policies-between-2-different-windows-machines/](https://pleasework.robbievance.net/howto-export-and-compare-security-policies-between-2-different-windows-machines/)  
40. Group Policy objects to Remote Desktop Services \- Windows Server \- Microsoft Learn, accessed September 28, 2025, [https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/apply-group-policy-objects-rds](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/apply-group-policy-objects-rds)  
41. How to configure RDP settings via GPO \- TruGrid Help, accessed September 28, 2025, [https://help.trugrid.com/en/article/how-to-configure-rdp-settings-via-gpo-1b8hn6g/](https://help.trugrid.com/en/article/how-to-configure-rdp-settings-via-gpo-1b8hn6g/)  
42. Group Policy overview for Windows Server | Microsoft Learn, accessed September 28, 2025, [https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-overview)