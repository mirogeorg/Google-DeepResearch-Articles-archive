
# **An Architectural Security Analysis of Passwordless Local Accounts for Network Automation**

## **Part I: Deconstruction of Windows Authentication and Blank Password Policies**

The use of local accounts with blank passwords for network-based automated tasks represents a significant deviation from the hardened security posture of modern Windows operating systems. This practice, while potentially offering a superficial layer of convenience, directly contravenes foundational security principles and requires the deliberate dismantling of built-in protective measures. An in-depth analysis of the Windows security model reveals that the default prohibition of network access for such accounts is not an operational hurdle but a critical, intentional security control. Understanding the architecture and rationale behind these controls is the first step in assessing the profound risks associated with their circumvention.

### **1.1 The Default Security Posture: A Deliberate Barrier**

The Windows security architecture is fundamentally predicated on the principles of explicit authentication and authorization for all network resource access. Within this model, a blank or null password is not treated as a weak credential but as the functional absence of an authentication secret for any non-console interaction. The operating system's default behavior—blocking network logons for accounts with blank passwords—is a deliberate and essential security barrier designed to prevent the most trivial forms of unauthorized access.1

This design philosophy is a direct result of lessons learned from earlier computing eras, where lax default permissions were a primary vector for the propagation of malware and unauthorized network traversal. Beginning with Windows Vista and Windows Server 2008, Microsoft initiated a significant architectural shift towards a "secure by default" posture.4 This shift embedded more stringent security controls directly into the operating system's baseline configuration. The prohibition of network logons for blank-password accounts is a cornerstone of this modern security model. Microsoft's own documentation unequivocally identifies blank passwords as a "serious threat to computer security" that must be forbidden through a combination of robust organizational policy and effective technical countermeasures.4

Therefore, the observation that Windows blocks this type of network access should not be viewed as an inconvenience to be bypassed. Instead, it should be interpreted as the operating system's security subsystem functioning precisely as intended, enforcing a baseline security requirement that protects the system and the broader network from a high-risk configuration. Any strategy that necessitates disabling this protection is, by definition, a regression to a less secure state and introduces a significant and unnecessary vulnerability into the environment.

### **1.2 Technical Deep Dive: The Accounts: Limit local account use of blank passwords to console logon only Policy**

The primary technical control governing this behavior is a specific Group Policy setting: Accounts: Limit local account use of blank passwords to console logon only. A granular understanding of this policy's function, scope, and default state is critical to appreciating the security implications of altering it.

#### **Function and Scope**

This policy setting precisely defines the permissible logon types for local accounts that do not have a password. When enabled, it strictly limits these accounts to "console logons" only. A console logon is defined as an interactive session initiated from the physical keyboard and mouse directly attached to the machine.4

Crucially, the policy explicitly blocks remote interactive logons for these accounts. This includes network services that are common vectors for both administrative access and malicious attacks, such as:

* Remote Desktop Services (RDP)  
* File Transfer Protocol (FTP)  
* Telnet 4

It is important to note the policy's scope limitations. It does not affect domain accounts, which are governed by the domain's password policies. It also does not necessarily prevent non-Microsoft applications that use custom remote logon mechanisms from potentially bypassing this setting, highlighting the need for defense-in-depth.4 However, for the vast majority of standard Windows network protocols, this policy is the definitive enforcement mechanism.

#### **Default Configuration**

The status of this policy as a baseline security standard is reinforced by its default configuration across all modern Windows operating systems. As detailed in the table below, the policy is effectively enabled by default on all stand-alone servers, member servers, and client computers, establishing a consistent security posture out of the box.

| Server Type or GPO | Default Value |
| :---- | :---- |
| Default Domain Policy | Not defined |
| Default Domain Controller Policy | Not defined |
| Stand-Alone Server Default Settings | Enabled |
| DC Effective Default Settings | Enabled |
| Member Server Effective Default Settings | Enabled |
| Client Computer Effective Default Settings | Enabled |
| Data compiled from Microsoft documentation.4 |  |

The "Not defined" status in the default domain policies means that without explicit configuration at the domain level, the secure local setting of individual machines will take precedence. This ensures that even in an unmanaged or newly-joined domain scenario, the endpoint remains protected by default.

#### **Implementation and Verification**

This policy can be configured and verified through standard Windows management tools.

* **Group Policy Management Console (GPMC):** The policy is located at: Computer Configuration\\Windows Settings\\Security Settings\\Local Policies\\Security Options.4  
* **Registry:** The underlying registry value that corresponds to this policy is:  
  * **Registry Hive:** HKEY\_LOCAL\_MACHINE  
  * **Registry Path:** \\SYSTEM\\CurrentControlSet\\Control\\Lsa\\  
  * **Value Name:** LimitBlankPasswordUse  
  * **Value Type:** REG\_DWORD  
  * **Value:** 1 (for Enabled) 3

Changes to this policy become effective without a system restart, making its enforcement or remediation straightforward.4

### **1.3 The Perils of Circumvention: Disabling Default Protections**

To facilitate the use of passwordless accounts over the network, administrators sometimes resort to several "workarounds." These methods do not resolve an underlying issue but rather create new, often more severe, security vulnerabilities by dismantling layers of the Windows security model.

One common but highly insecure method is to disable "Password protected sharing." This setting, found in the "Manage advanced sharing settings" control panel, essentially reverts the system's file-sharing security model to its pre-Windows Vista state.5 When this feature is turned off, the system no longer requires network users to authenticate with a valid user account and password on the target machine to access shared folders. Instead, it allows guest-only access, effectively opening any shares configured for "Everyone" to any user on the local network without any form of authentication. This is an extremely dangerous configuration in any shared network environment, as it provides an open door for unauthorized data access, modification, and malware propagation.5

Another dangerous workaround involves enabling the "Enable insecure guest logons" policy. This policy, located under Computer Configuration \> Administrative Templates \> Network \> Lanman Workstation, allows a client machine to connect to network shares using unauthenticated guest access.10 While this may appear to solve the immediate connectivity problem, it does so by fundamentally weakening the client's security posture, making it vulnerable to a variety of network-based attacks, such as man-in-the-middle attacks, where an attacker could redirect the client to a malicious server.

These actions are not fixes; they are deliberate security regressions. The administrative act of disabling these protections is the direct cause that activates an otherwise dormant vulnerability. An account with a blank password that is restricted to console-only logon is a minor local security issue; the same account exposed to the network via disabled security policies becomes a critical, remotely exploitable vulnerability.

## **Part II: Threat Model: The Anatomy of a Compromise**

The security of an automated process cannot be evaluated solely on the basis of its intended, legitimate use. A comprehensive risk assessment must consider how the configuration can be abused by a malicious actor. The strategy of using a passwordless local account, even when initiated from a securely connected session, creates a fragile security model that collapses upon the initial compromise of the endpoint. Once an attacker gains even a low-privilege foothold on the machine hosting the passwordless account, that account becomes a powerful tool for both vertical privilege escalation on the local machine and horizontal lateral movement across the network.

### **2.1 The Fallacy of the Initial Connection: Compromise of the Endpoint**

A common misconception is that if the primary connection to a machine (e.g., an RDP session for administration) is secured with strong credentials and multi-factor authentication, then subsequent actions performed on that machine are inherently secure. This perspective is dangerously flawed, as it ignores the vast landscape of initial access techniques employed by adversaries.

The security of the initial administrative connection is rendered irrelevant the moment an attacker gains any level of code execution on the endpoint through an alternative vector. Attackers have a wide array of methods for achieving this initial foothold, including:

* **Phishing:** Tricking a legitimate user of the machine into executing a malicious attachment or clicking a link that downloads malware.12  
* **Exploitation of Unpatched Software:** Leveraging vulnerabilities in web browsers, document readers, or other third-party applications installed on the system.13  
* **Social Engineering:** Persuading a user to install a seemingly benign application that contains a malicious payload.

Once an attacker has established this initial, often low-privilege, presence on the machine, the security context shifts entirely. The focus is no longer on breaching the network perimeter but on leveraging the resources available on the compromised endpoint. In this context, a pre-existing local account with a blank password is not a minor configuration detail; it is a critical vulnerability and a prime target for exploitation.

### **2.2 Attack Vector 1: Local Privilege Escalation**

Privilege escalation is the process by which an attacker expands their access rights from those of a standard user to those of an administrator or the NT AUTHORITY\\SYSTEM account, granting them complete control over the local machine.15 A passwordless local account can serve as a direct pathway or a crucial intermediate step in multiple privilege escalation techniques.

#### **Technique A: Exploiting Unquoted Service Paths**

This classic Windows vulnerability arises when the path to a service's executable contains spaces but is not enclosed in quotation marks in the registry.17 For example, consider a service with the following ImagePath registry value: C:\\Program Files\\Vendor App\\service.exe.

When Windows attempts to start this service, it may interpret the path sequentially:

1. It first looks for C:\\Program.exe.  
2. If not found, it looks for C:\\Program Files\\Vendor.exe.  
3. If not found, it finally looks for C:\\Program Files\\Vendor App\\service.exe.17

An attacker with standard user permissions who finds they have write access to a higher-level directory (e.g., C:\\Program Files\\) can place a malicious executable named Vendor.exe in that location. If the service is configured to run with the privileges of the passwordless local account, and that account happens to be a member of the local Administrators group, the attacker's malicious code will be executed with full administrative privileges the next time the service starts.15 The passwordless account becomes the vehicle for the exploit, turning a configuration flaw into a full system compromise.

#### **Technique B: Insecure Service Permissions and Weak Registry Keys**

A similar attack vector exists when a service itself has weak permissions. An attacker who gains a foothold on the system may discover that they have permissions to modify the files or registry keys associated with a service.13

If a service is configured to run as the passwordless "archive" account, and an attacker finds they can modify the service's ImagePath value in the registry (located under HKLM\\SYSTEM\\CurrentControlSet\\Services\\\<ServiceName\>), they can redirect the service to execute a malicious payload of their choosing.15 Upon the next service restart, which the attacker may be able to trigger, their code will run with all the permissions granted to the "archive" account. If this account has been granted excessive privileges for convenience, this leads directly to privilege escalation.13

#### **Technique C: Credential and Token Theft**

Perhaps the most significant risk is the role of the passwordless account as the first step in a more complex attack chain. An attacker may initially use a simple exploit to gain access as a standard user. From there, they may not be able to directly achieve administrative rights. However, if they can access or impersonate the passwordless local account, they can use its permissions to gain a more stable foothold on the system.

From this new position, they can probe for other vulnerabilities. If they successfully exploit a separate vulnerability to escalate to NT AUTHORITY\\SYSTEM, they can then execute tools like Mimikatz or perform memory scraping on the Local Security Authority Subsystem Service (LSASS) process.21 This action allows them to dump all credential hashes and Kerberos tickets stored in memory for any user who has logged onto that machine, including domain administrators or other privileged service accounts.23

In this scenario, the passwordless account is the weak link that enables the entire chain. It transforms a low-impact initial compromise into a catastrophic breach of domain credentials, allowing the attacker to break out of the local machine and begin moving laterally throughout the entire Active Directory environment.

### **2.3 Attack Vector 2: Lateral Movement Across the Network**

Lateral movement refers to the techniques an attacker uses to pivot from a compromised machine to other systems within the network, with the ultimate goal of reaching high-value targets like domain controllers or sensitive data repositories.25 Disabling the default Windows security policy that blocks network access for blank-password accounts directly enables multiple forms of lateral movement.

#### **Technique A: Pass-the-Hash (PtH) with a Blank Password**

The Pass-the-Hash attack is a powerful technique where an adversary uses a user's NTLM password hash, rather than the plaintext password, to authenticate to remote systems.21 This bypasses the need to crack the password.

For a standard account, an attacker must first compromise a machine and steal the hash from memory or the SAM database. However, the situation with a blank password is far more dire. The NTLM hash for a blank password is a static, publicly known value: 31d6cfe0d16ae931b73c59d7e0c089c0.33

An attacker on the network does not need to steal this hash; they already know it. If they can discover the username of the passwordless account (e.g., "archive\_svc"), which can often be enumerated through network scanning or discovered in configuration files, they can immediately launch a Pass-the-Hash attack against other network resources. Using tools like Mimikatz or Invoke-TheHash, they can attempt to authenticate to file shares (SMB), remote management interfaces (WMI), or other services on other servers using the known username and the static blank password hash.22 This is one of the most frictionless forms of lateral movement, as it completely eliminates the credential theft phase of the attack.

#### **Technique B: Pivoting to Network Shares and Resources**

The intended purpose of the passwordless account—to pull archives—implies that it has been granted permissions to at least one, and possibly multiple, network file shares. Once the security policies are disabled and the account is accessible over the network, these permissions become a direct vector for an attacker.

An adversary who has compromised the account can now use its credentials (the username and a blank password or the known hash) to enumerate and access every network resource it has permissions for.34 This access can be abused in several ways:

* **Data Exfiltration:** The attacker can copy sensitive data from the archive shares to a system under their control.  
* **Malware Staging:** The attacker can write malicious tools or ransomware payloads to the shares, using them as a central distribution point to infect other systems that connect to the same shares.36  
* **Data Destruction:** The attacker could encrypt or delete the archives, causing significant operational disruption.

#### **Technique C: Credential Stuffing and Reuse**

A common but insecure practice in many organizations is the reuse of local account credentials across multiple systems for administrative consistency. Attackers are well aware of this tendency.

After discovering the username of the passwordless account on the initial target machine (e.g., "archive\_svc"), an attacker can launch an automated, network-wide credential stuffing attack.12 They will attempt to authenticate to the SMB or RDP services of every other server and workstation on the network using the username "archive\_svc" and a blank password. If the same insecure configuration has been replicated on other machines for similar automated tasks, the attacker will gain access to each of them, rapidly expanding their foothold across the environment. Each newly compromised machine becomes another potential source of valuable data and cached domain credentials.

The presence of the passwordless account fundamentally alters the attack calculus. It is not merely a passive vulnerability but an active catalyst that dramatically lowers the complexity, time, and skill required for an adversary to escalate a minor endpoint compromise into a full-scale network breach. A local configuration choice on a single machine, when combined with the disabling of default security policies, creates a direct and severe threat to the security of the entire domain.

## **Part III: A Framework for Secure Automated System Interaction**

To effectively mitigate the risks associated with automated processes, it is necessary to move beyond insecure shortcuts and adopt an architectural framework grounded in established cybersecurity principles. The strategy of using passwordless local accounts is fundamentally at odds with modern security paradigms. A robust and defensible approach must be built on two pillars: the Principle of Least Privilege (PoLP) and the architectural imperative to eliminate or securely manage static credentials.

### **3.1 The Principle of Least Privilege (PoLP) for Service Accounts**

The Principle of Least Privilege is a foundational concept in computer security which dictates that any account, user, or process should be granted only the minimum levels of access—or permissions—necessary to perform its required functions.28 When applied to service accounts used for automation, this principle has several concrete implications:

* **Single-Purpose Accounts:** Each automated task or service should have its own dedicated service account. Sharing a single account across multiple applications or tasks complicates management and violates PoLP. If that shared account is compromised, all associated services are compromised. For the task of pulling archives, a dedicated account should be created that does nothing else.37  
* **Minimal Permissions:** The account must be granted only the precise permissions it needs. For an archive-pulling task, this would typically mean Read access to the source directory and Write access to the destination directory. The account should have no other permissions.  
* **Restricted Logon Rights:** The account should be explicitly denied unnecessary rights. This includes denying interactive logon, remote desktop logon, and logon as a batch job if not required. It should not be a member of any administrative groups, such as local Administrators or Domain Admins.37

Adhering to PoLP ensures that if the service account is ever compromised, the potential damage is strictly contained to its narrowly defined function. An attacker gaining control of the archive account could only read and write to the specific archive folders, rather than having the ability to manage the system or access other network resources.

### **3.2 The Architectural Imperative: Eliminating Static and Blank Passwords**

Static credentials—passwords that are manually set and rarely, if ever, changed—are a primary target for attackers. Blank passwords represent the most extreme and insecure form of a static credential. Modern security architecture is increasingly focused on solutions that either completely eliminate the use of passwords for non-human accounts or automate their management to remove the risks associated with human handling.37

The goal is to move to a model where no human administrator knows or needs to know the password for a service account. This prevents passwords from being written down, stored insecurely in scripts, or reused across different systems. The implementation blueprints that follow in Part IV are direct applications of this architectural principle. They represent three distinct, modern approaches to solving the problem of secure automation:

1. **Automated Password Management:** Using a system where the operating system itself manages the account's complex password, rotating it automatically without human intervention. This is the approach taken by Windows Managed Service Accounts.41  
2. **Externalized Credential Management:** Storing the credential in a secure, encrypted vault and having the script or application retrieve it at runtime. This removes the password from the script file and centralizes its management. This is the approach of PowerShell Secret Management.42  
3. **Passwordless Cryptographic Authentication:** Replacing passwords entirely with public key cryptography, where authentication is based on possession of a private key rather than knowledge of a shared secret. This is the gold standard for protocols like SFTP and SSH.43

By adopting one of these architectural patterns, an organization can build automated processes that are not only functional but also inherently secure and resilient against the types of credential-based attacks detailed in the threat model.

## **Part IV: Implementation Blueprints for Secure Automation**

Transitioning from the high-risk strategy of using passwordless local accounts to a secure, modern framework requires the implementation of robust alternatives. The following three blueprints provide detailed, step-by-step guidance for architecting secure automated interactions in a Windows environment. These solutions are not theoretical; they are practical, well-documented, and represent industry best practices for service account management and automated file transfers.

### **4.1 Blueprint A: The Native Solution \- Windows Managed Service Accounts (MSAs/gMSAs)**

For automated tasks running within an Active Directory domain, Windows Managed Service Accounts (MSAs) and Group Managed Service Accounts (gMSAs) are the premier native solution. They are designed specifically to address the security challenges of traditional service accounts by completely automating password management.41

#### **Overview**

The core benefit of an MSA or gMSA is the elimination of manual password management. The password for the account is a complex, 240-character string that is automatically generated and managed by the domain controllers. This password is changed periodically (every 30 days by default) without any administrative intervention.45 This makes the password effectively unknown to any human, preventing it from being stolen or misused.

* **Standalone MSA (sMSA):** Can be used on a single, specific computer.41  
* **Group MSA (gMSA):** Can be used across multiple servers in a cluster or farm, making it more flexible for modern applications.41 For most new deployments, a gMSA is the recommended choice.

#### **Prerequisites: Creating the KDS Root Key**

Before any MSAs can be created, the Key Distribution Service (KDS) Root Key must be generated in the Active Directory forest. This is a one-time operation performed on a domain controller. This key serves as the cryptographic foundation for generating MSA passwords.45

Execute the following command in an elevated PowerShell window on a domain controller:

PowerShell

Add-KdsRootKey \-EffectiveTime ((Get-Date).AddHours(\-10))

The \-EffectiveTime parameter allows the key to become available for use immediately by setting its effective date in the past, bypassing a default replication delay. This is safe for initial setup in most environments.45

#### **Implementation Guide (gMSA)**

The following steps outline the creation and deployment of a gMSA for the archive-pulling task. All commands are to be run in PowerShell with the Active Directory module installed.

1. **Create a Security Group:** First, create a security group that will contain the computer accounts authorized to use the gMSA.  
   PowerShell  
   New-ADGroup \-Name "ArchiveSvcHosts" \-GroupScope Global  
   Add-ADGroupMember \-Identity "ArchiveSvcHosts" \-Members "REMOTE-SERVER-01$"

   Replace REMOTE-SERVER-01$ with the computer account of the server that will run the archive task.  
2. **Create the gMSA:** Create the gMSA and link it to the security group created in the previous step.  
   PowerShell  
   New-ADServiceAccount \-Name "gMSA-ArchiveSvc" \-DNSHostName "gMSA-ArchiveSvc.yourdomain.com" \-PrincipalsAllowedToRetrieveManagedPassword "ArchiveSvcHosts"

   This command creates the gMSA and specifies that only members of the "ArchiveSvcHosts" group are permitted to retrieve its password from Active Directory.45  
3. **Install the gMSA on the Host Server:** On the server that will run the task (REMOTE-SERVER-01), install the gMSA. This action tests the server's ability to retrieve the password and prepares it for use.  
   PowerShell  
   \# Run on REMOTE-SERVER-01  
   Install-ADServiceAccount \-Identity "gMSA-ArchiveSvc"

   A successful installation requires a reboot or a restart of the Netlogon service for the changes to take full effect.  
4. **Configure the Scheduled Task:** Configure the scheduled task that pulls the archives to run under the context of the gMSA.  
   * When specifying the user account for the task, enter the gMSA name in the format DOMAIN\\gMSA-ArchiveSvc$.  
   * **Leave the password field completely blank.** Windows will handle the authentication with the domain controller automatically.46

#### **Security Benefits**

This blueprint directly resolves the core security flaw of the original strategy. It replaces a static, blank password with a highly complex, automatically rotated, and unknown password. Access is cryptographically controlled and limited only to authorized machines, perfectly embodying the principles of least privilege and secure credential management.

### **4.2 Blueprint B: The Scripting Solution \- Secure PowerShell Credential Management**

For scenarios requiring more flexibility or operating outside a pure Active Directory environment, securely managing credentials within scripts is paramount. The PowerShell ecosystem provides a modern framework for this purpose with the SecretManagement and SecretStore modules, which prevent credentials from ever being stored in plaintext within a script file.42

#### **Overview**

This approach externalizes the credential from the automation script. The script itself contains no sensitive information. Instead, it queries a secure, encrypted local vault at runtime to retrieve the necessary credential. This vault is protected and can only be accessed by the authorized user context.42

#### **Core Components**

* **PSCredential Object:** This is the standard PowerShell object for securely handling a username and password in memory. The password component is stored as a SecureString, which is encrypted in memory.50  
* **Microsoft.PowerShell.SecretManagement:** An extensible module that provides a common set of cmdlets to interact with various secret vaults.49  
* **Microsoft.PowerShell.SecretStore:** A vault extension for SecretManagement that creates an encrypted file-based vault on the local machine for the current user's context.48

#### **Implementation Guide**

1. **Install and Configure the Modules:** On the machine running the script, install and configure the necessary modules from the PowerShell Gallery.  
   PowerShell  
   \# Run as Administrator  
   Install-Module Microsoft.PowerShell.SecretManagement  
   Install-Module Microsoft.PowerShell.SecretStore  
   Register-SecretVault \-Name LocalStore \-ModuleName Microsoft.PowerShell.SecretStore \-DefaultVault

   The first time you use the SecretStore, you will be prompted to create a password for the vault. This password is required to unlock the vault in new PowerShell sessions.  
2. **Store the Credential Securely:** Create a PSCredential object and store it in the newly configured vault. This is a one-time setup step.  
   PowerShell  
   \# This command will prompt for the username and password of the service account  
   $cred \= Get-Credential  
   Set-Secret \-Name "ArchiveSvcCredential" \-Secret $cred

   The secret, named "ArchiveSvcCredential," is now stored securely in the local encrypted vault.42  
3. **Retrieve and Use the Credential in the Automation Script:** The script that pulls the archives will now retrieve the credential from the vault at runtime.  
   PowerShell  
   \# Script: Pull-Archives.ps1

   \# Check if the vault is locked and prompt for the password if needed  
   if ((Get-SecretVault \-Name LocalStore).IsVaultLocked) {  
       Unlock-SecretVault \-Name LocalStore  
   }

   \# Retrieve the credential object from the vault  
   $retrievedCred \= Get-Secret \-Name "ArchiveSvcCredential" \-AsPSCredential

   \# Use the credential in a command that requires network authentication  
   \# Example: Copying files from a remote share  
   Copy-Item \-Path "\\\\SOURCE-SERVER\\Archives\\\*" \-Destination "C:\\LocalArchives" \-Credential $retrievedCred

   When this script runs, it will first check the vault status. If it's the first time accessing the vault in the session, it will prompt for the vault password. Then, it retrieves the PSCredential object and uses it for the network operation.49

#### **Security Benefits**

This blueprint removes all static credentials from the script code. The password is encrypted at rest in the SecretStore vault. The script only ever handles the secure PSCredential object in memory, dramatically reducing the risk of accidental credential exposure through source code control, log files, or shoulder surfing.47

### **4.3 Blueprint C: The Transfer Solution \- Public Key Cryptography with SFTP/SSH**

Given that the specified task is to "pull archives," a file transfer protocol is at the core of the operation. The industry gold standard for secure, automated file transfers is SFTP (SSH File Transfer Protocol) using public key authentication. This method completely eliminates passwords in favor of strong asymmetric cryptography.43

#### **Overview**

Public key authentication relies on a mathematically linked key pair: a private key, which is kept secret on the client machine, and a public key, which is placed on the server. The server uses the public key to issue a challenge that can only be answered correctly by the client holding the corresponding private key, thus proving its identity without ever transmitting a password.43

#### **Server-Side Setup (Windows OpenSSH Server)**

1. **Install OpenSSH Server:** On the server hosting the archives, install the OpenSSH Server feature. This can be done via Settings \> Optional features or with PowerShell:  
   PowerShell  
   Add-WindowsCapability \-Online \-Name OpenSSH.Server\~\~\~\~0.0.1.0

2. **Start and Configure the Service:** Ensure the sshd service is running and set to start automatically.  
   PowerShell  
   Start-Service sshd  
   Set-Service \-Name sshd \-StartupType 'Automatic'

3. **Configure Firewall:** Allow inbound traffic on TCP port 22\.  
   PowerShell  
   New-NetFirewallRule \-Name sshd \-DisplayName 'OpenSSH Server (sshd)' \-Enabled True \-Direction Inbound \-Protocol TCP \-Action Allow \-LocalPort 22

#### **Client-Side Setup and Key Generation**

1. **Generate Key Pair:** On the client machine that will run the archive script, generate an SSH key pair using ssh-keygen.exe.  
   PowerShell  
   ssh\-keygen.exe \-t rsa \-b 4096

   When prompted, it is highly recommended to provide a strong passphrase to encrypt the private key. This acts as a final layer of protection; even if the private key file is stolen, it is useless without the passphrase.55 This will create two files, typically id\_rsa (the private key) and id\_rsa.pub (the public key), in the user's \\.ssh directory.  
2. **Deploy the Public Key to the Server:** The contents of the public key file (id\_rsa.pub) must be copied to the server and placed in a specific file. For a standard user account archiveuser, the path is C:\\Users\\archiveuser\\.ssh\\authorized\_keys.55  
   * Create the .ssh directory on the server if it doesn't exist.  
   * Create the authorized\_keys file.  
   * Paste the entire content of the client's id\_rsa.pub file into the authorized\_keys file. Ensure permissions on this file are restricted to the owner account.57

#### **Implementation Guide**

The automation script on the client can now use an SFTP client that supports public key authentication.

PowerShell

\# Example using a conceptual PowerShell SFTP module or command-line tool

\# The SFTP client will automatically use the private key from the user's.ssh directory  
\# If the key is passphrase protected, the user will be prompted to enter it once  
sftp\-get \-Host "SOURCE-SERVER" \-User "archiveuser" \-RemotePath "/Archives/\*" \-LocalPath "C:\\LocalArchives"

Many SFTP clients and libraries can integrate with ssh-agent, which can cache the decrypted private key for a session, avoiding repeated passphrase prompts.55

#### **Security Benefits**

This is arguably the most secure solution for file transfers. It eliminates shared secrets (passwords) entirely. Authentication is based on possession of the private key, which is protected by both file system permissions and an optional strong passphrase. Access can be granularly controlled and instantly revoked by simply removing the corresponding public key from the server's authorized\_keys file.54

### **Comparative Analysis of Strategies**

The following table provides a comparative analysis of the current insecure strategy against the three secure blueprints proposed.

| Strategy | Primary Use Case | Security Level | Implementation Complexity | Key Dependencies | Password Management |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **Current Strategy (Passwordless Local Account)** | General-purpose automation (insecurely) | **Very Low** | Low | Disabled Windows Security Policies | None (the vulnerability) |
| **Blueprint A (gMSA)** | Windows-native services and scheduled tasks in an AD environment | **High** | Medium | Active Directory, KDS Root Key | Fully automated by AD |
| **Blueprint B (Secure PowerShell)** | Flexible PowerShell scripting for various protocols | **High** | Medium | PowerShell 7+, SecretManagement/SecretStore modules | Manual setup, secure retrieval at runtime |
| **Blueprint C (SFTP w/ Keys)** | Cross-platform, secure automated file transfers | **Very High** | Medium | OpenSSH Server/Client | Eliminated (uses cryptographic keys) |

This analysis clearly demonstrates that while the current strategy offers low implementation complexity, it does so at the cost of creating an unacceptable security risk. Each of the proposed blueprints provides a high or very high level of security, representing a significant and necessary improvement to the organization's security posture.

## **Part V: Strategic Conclusions and Actionable Recommendations**

The analysis of using a passwordless local account for network-based automation tasks leads to an unambiguous conclusion: the practice is fundamentally insecure and introduces severe, unnecessary risks to the organization. The default Windows security policies that prevent this behavior are not obstacles to be circumvented but are essential safeguards that form a critical part of a defense-in-depth strategy. Disabling these protections creates an easily exploitable vulnerability that can serve as a catalyst for both local privilege escalation and lateral movement across the network.

### **5.1 Final Assessment: An Untenable Security Risk**

The strategy of pulling archives with a passwordless local account is an untenable security risk. It is a direct contradiction of security best practices recommended by Microsoft and the broader cybersecurity community. The risks are not theoretical but are based on well-understood and commonly exploited attack vectors:

* **Trivial Network Access:** By disabling default protections, the account becomes accessible over the network to any attacker who can discover its username.  
* **Known Credential Exploitation:** The NTLM hash for a blank password is a publicly known constant, allowing attackers to bypass the credential theft phase entirely and immediately launch Pass-the-Hash attacks.  
* **Privilege Escalation and Lateral Movement:** The account provides a frictionless entry point for attackers to gain a foothold on a system, from which they can escalate privileges, dump domain credentials from memory, and pivot to other high-value systems on the network.

Continuing this practice exposes the organization to a high probability of a security breach, data exfiltration, or a ransomware event originating from this single point of weakness.

### **5.2 Prioritized Remediation Plan**

A clear, prioritized plan of action is required to mitigate this risk and transition to a secure and sustainable model for automation. The following steps should be taken:

1. **Immediate Containment:**  
   * Immediately cease all network-based operations that use the passwordless local account.  
   * Disable the passwordless local account on all systems where it exists to prevent any further use. A strong, complex password should be assigned to the account if it cannot be disabled immediately for operational reasons, and its use should be strictly limited to console-only access pending full remediation.  
2. **Environmental Audit and Hardening:**  
   * Conduct a comprehensive audit of all servers and workstations within the domain to identify any other local accounts configured with blank passwords.  
   * Use Group Policy to centrally enforce the Accounts: Limit local account use of blank passwords to console logon only policy, setting it to "Enabled" across all Organizational Units (OUs) containing computers. This will establish a consistent and secure baseline and override any insecure local configurations.4  
   * Simultaneously, ensure that insecure settings like "Password protected sharing" are disabled (i.e., password protection is *on*) and "Enable insecure guest logons" is disabled.  
3. **Strategic Implementation of a Secure Alternative:**  
   * Review the three implementation blueprints presented in Part IV of this report.  
   * **Primary Recommendation:** For the specific task of pulling archives, **Blueprint C: Public Key Cryptography with SFTP/SSH** is the most secure and appropriate solution. It is the industry standard for automated file transfers and completely eliminates password-based authentication.  
   * **Secondary Recommendation:** If the task is performed by a Windows Scheduled Task in a purely Windows-based environment and there is a broader need to secure other services, **Blueprint A: Windows Managed Service Accounts (gMSAs)** is an excellent, highly secure, and natively integrated alternative.  
   * **Tertiary Recommendation:** For more complex scripting logic that involves protocols other than file transfer, **Blueprint B: Secure PowerShell Credential Management** provides a flexible and secure framework.  
4. **Policy and Procedure Enhancement:**  
   * Update the organization's information security policy to explicitly forbid the use of local accounts with blank passwords for any purpose.  
   * Establish a formal standard operating procedure (SOP) for the creation and management of all service and automation accounts. This SOP must mandate the use of one of the approved secure methods (gMSAs, secure credential vaults, or key-based authentication) and incorporate the Principle of Least Privilege for all permission assignments.  
   * Provide training to all system administrators and IT operations staff on these new policies and the technical implementation of the approved solutions.

#### **Works cited**

1. Local accounts with blank passwords must be ... \- STIG VIEWER, accessed October 18, 2025, [https://stigviewer.com/stigs/microsoft\_windows\_11/2025-02-25/finding/V-253434](https://stigviewer.com/stigs/microsoft_windows_11/2025-02-25/finding/V-253434)  
2. 2.3.1.3 Ensure 'Accounts: Limit local account use of blank passwords to console logon only' is set to 'Enabled', accessed October 18, 2025, [https://www.tenable.com/audits/items/CIS\_MS\_Windows\_7\_v3.2.0\_Level\_1.audit:ff1d2c5dee27562b333b4353050689c7](https://www.tenable.com/audits/items/CIS_MS_Windows_7_v3.2.0_Level_1.audit:ff1d2c5dee27562b333b4353050689c7)  
3. Local accounts with blank passwords must be restricted to prevent access from the network., accessed October 18, 2025, [https://stigviewer.com/stigs/microsoft\_windows\_10/2025-02-25/finding/V-220910](https://stigviewer.com/stigs/microsoft_windows_10/2025-02-25/finding/V-220910)  
4. Accounts Limit local account use of blank passwords \- Windows 10 \- Microsoft Learn, accessed October 18, 2025, [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/accounts-limit-local-account-use-of-blank-passwords-to-console-logon-only](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/accounts-limit-local-account-use-of-blank-passwords-to-console-logon-only)  
5. Unable to access shared folder without user name and password in windows 10, accessed October 18, 2025, [https://superuser.com/questions/959218/unable-to-access-shared-folder-without-user-name-and-password-in-windows-10](https://superuser.com/questions/959218/unable-to-access-shared-folder-without-user-name-and-password-in-windows-10)  
6. Accounts: Limit local account use of blank passwords to console logon only | Microsoft Learn, accessed October 18, 2025, [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/jj852174(v=ws.11)](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/jj852174\(v=ws.11\))  
7. Limit local account use of blank passwords to console logon only \- ManageEngine, accessed October 18, 2025, [https://www.manageengine.com/products/active-directory-audit/kb/security-policy-settings/accounts-limit-local-account-use-of-blank-passwords-to-console-logon-only.html](https://www.manageengine.com/products/active-directory-audit/kb/security-policy-settings/accounts-limit-local-account-use-of-blank-passwords-to-console-logon-only.html)  
8. learn.microsoft.com, accessed October 18, 2025, [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/accounts-limit-local-account-use-of-blank-passwords-to-console-logon-only\#:\~:text=Reference-,The%20Accounts%3A%20Limit%20local%20account%20use%20of%20blank%20passwords%20to,accounts%20that%20have%20blank%20passwords.](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/accounts-limit-local-account-use-of-blank-passwords-to-console-logon-only#:~:text=Reference-,The%20Accounts%3A%20Limit%20local%20account%20use%20of%20blank%20passwords%20to,accounts%20that%20have%20blank%20passwords.)  
9. Prevent remote logon for local accounts with blank password \- GPO, accessed October 18, 2025, [https://www.windows-active-directory.com/prevent-remote-logon-for-local-accounts-with-blank-password-gpo.html](https://www.windows-active-directory.com/prevent-remote-logon-for-local-accounts-with-blank-password-gpo.html)  
10. Your Organization's Security Policies Block Unauthenticated Guest Access \[FIX\] \- YouTube, accessed October 18, 2025, [https://www.youtube.com/watch?v=L4-rHQzMbdk](https://www.youtube.com/watch?v=L4-rHQzMbdk)  
11. Windows 11 24H2 and cannot login SMB without password : r/unRAID \- Reddit, accessed October 18, 2025, [https://www.reddit.com/r/unRAID/comments/1hhl28h/windows\_11\_24h2\_and\_cannot\_login\_smb\_without/](https://www.reddit.com/r/unRAID/comments/1hhl28h/windows_11_24h2_and_cannot_login_smb_without/)  
12. 6 Types of Password Attacks & How to Stop Them | OneLogin, accessed October 18, 2025, [https://www.onelogin.com/learn/6-types-password-attacks](https://www.onelogin.com/learn/6-types-password-attacks)  
13. Windows Privilege Escalation Part 1: Local Administrator Privileges \- NetSPI, accessed October 18, 2025, [https://www.netspi.com/blog/technical-blog/network-pentesting/windows-privilege-escalation-part-1-local-administrator-privileges/](https://www.netspi.com/blog/technical-blog/network-pentesting/windows-privilege-escalation-part-1-local-administrator-privileges/)  
14. Two New Windows Zero-Days Exploited in the Wild — One Affects Every Version Ever Shipped \- The Hacker News, accessed October 18, 2025, [https://thehackernews.com/2025/10/two-new-windows-zero-days-exploited-in.html](https://thehackernews.com/2025/10/two-new-windows-zero-days-exploited-in.html)  
15. Privilege Escalation on Windows (With Examples) \- Delinea, accessed October 18, 2025, [https://delinea.com/blog/windows-privilege-escalation](https://delinea.com/blog/windows-privilege-escalation)  
16. Pentesting Lab \- Privilege Escalation \- Windows \- ISEC, accessed October 18, 2025, [https://www.isec.tugraz.at/wp-content/uploads/2024/09/03-privilege-escalation-windows-handout.pdf](https://www.isec.tugraz.at/wp-content/uploads/2024/09/03-privilege-escalation-windows-handout.pdf)  
17. Unquoted Service Path Privilege Escalation \- Michael Waterman, accessed October 18, 2025, [https://michaelwaterman.nl/2023/02/06/unquoted-service-path-privilege-escalation/](https://michaelwaterman.nl/2023/02/06/unquoted-service-path-privilege-escalation/)  
18. How to fix the Windows unquoted service path vulnerability \- InfoSec Governance, accessed October 18, 2025, [https://isgovern.com/blog/how-to-fix-the-windows-unquoted-service-path-vulnerability/](https://isgovern.com/blog/how-to-fix-the-windows-unquoted-service-path-vulnerability/)  
19. Unlocking the Mystery of Unquoted Service Paths: Another Opportunity for Privilege Escalation \- Blackpoint Cyber, accessed October 18, 2025, [https://blackpointcyber.com/blog/unlocking-the-mystery-of-unquoted-service-paths-another-opportunity-for-privilege-escalation/](https://blackpointcyber.com/blog/unlocking-the-mystery-of-unquoted-service-paths-another-opportunity-for-privilege-escalation/)  
20. Windows Privilege Escalation – An Approach For Penetration Testers \- SEC Consult, accessed October 18, 2025, [https://sec-consult.com/blog/detail/windows-privilege-escalation-an-approach-for-penetration-testers/](https://sec-consult.com/blog/detail/windows-privilege-escalation-an-approach-for-penetration-testers/)  
21. Pass the Hash Attack Defense | AD Security 101 \- Semperis, accessed October 18, 2025, [https://www.semperis.com/blog/how-to-defend-against-pass-the-hash-attack/](https://www.semperis.com/blog/how-to-defend-against-pass-the-hash-attack/)  
22. Pass the Hash Attack Explained | Semperis Identity Attack Catalog, accessed October 18, 2025, [https://www.semperis.com/blog/pass-the-hash-attack-explained/](https://www.semperis.com/blog/pass-the-hash-attack-explained/)  
23. Lateral Movement \- Timur Engin, accessed October 18, 2025, [https://timurengin.com/lateral-movement-813ea27980db](https://timurengin.com/lateral-movement-813ea27980db)  
24. Detecting Lateral Movements in Windows Infrastructure \- CERT-EU, accessed October 18, 2025, [https://cert.europa.eu/static/WhitePapers/CERT-EU\_SWP\_17-002\_Lateral\_Movements.pdf](https://cert.europa.eu/static/WhitePapers/CERT-EU_SWP_17-002_Lateral_Movements.pdf)  
25. Understand and investigate Lateral Movement Paths \- Microsoft Defender for Identity, accessed October 18, 2025, [https://learn.microsoft.com/en-us/defender-for-identity/understand-lateral-movement-paths](https://learn.microsoft.com/en-us/defender-for-identity/understand-lateral-movement-paths)  
26. Lateral Movement \- Portnox, accessed October 18, 2025, [https://www.portnox.com/cybersecurity-101/lateral-movement/](https://www.portnox.com/cybersecurity-101/lateral-movement/)  
27. Protecting against Lateral Movement with Defender for Identity and monitor with Azure Sentinel \- Jeffrey Appel, accessed October 18, 2025, [https://jeffreyappel.nl/protecting-against-lateral-movement-with-defender-for-identity-and-monitor-with-azure-sentinel/](https://jeffreyappel.nl/protecting-against-lateral-movement-with-defender-for-identity-and-monitor-with-azure-sentinel/)  
28. What is a Pass-the-Hash Attack? | CrowdStrike, accessed October 18, 2025, [https://www.crowdstrike.com/en-us/cybersecurity-101/cyberattacks/pass-the-hash-attack/](https://www.crowdstrike.com/en-us/cybersecurity-101/cyberattacks/pass-the-hash-attack/)  
29. What is a Pass-the-Hash-Attack (PtH)? \- Delinea, accessed October 18, 2025, [https://delinea.com/what-is/pass-the-hash-attack-pth](https://delinea.com/what-is/pass-the-hash-attack-pth)  
30. An Expert Guide to Mitigating Pass-the-Hash Attacks in Active Directory | NCC Group, accessed October 18, 2025, [https://www.nccgroup.com/research-blog/defending-your-directory-an-expert-guide-to-mitigating-pass-the-hash-attacks-in-active-directory/](https://www.nccgroup.com/research-blog/defending-your-directory-an-expert-guide-to-mitigating-pass-the-hash-attacks-in-active-directory/)  
31. Use Alternate Authentication Material: Pass the Hash, Sub-technique T1550.002, accessed October 18, 2025, [https://attack.mitre.org/techniques/T1550/002/](https://attack.mitre.org/techniques/T1550/002/)  
32. What Is a Pass the Hash Attack? | Proofpoint US, accessed October 18, 2025, [https://www.proofpoint.com/us/threat-reference/pass-the-hash](https://www.proofpoint.com/us/threat-reference/pass-the-hash)  
33. Abusing empty passwords during your next red teaming engagement, accessed October 18, 2025, [https://northwave-cybersecurity.com/threat-intel-research/abusing-empty-passwords-during-your-next-red-teaming-engagement](https://northwave-cybersecurity.com/threat-intel-research/abusing-empty-passwords-during-your-next-red-teaming-engagement)  
34. 10 Common Lateral Movement Techniques and How to Stop Them | Zero Networks, accessed October 18, 2025, [https://zeronetworks.com/blog/10-common-lateral-movement-techniques-how-to-stop-them](https://zeronetworks.com/blog/10-common-lateral-movement-techniques-how-to-stop-them)  
35. The Basics of Windows Lateral Movement | by Keith Monroe | Medium, accessed October 18, 2025, [https://boxalarm.tech/the-basics-of-windows-lateral-movement-a247b01a480](https://boxalarm.tech/the-basics-of-windows-lateral-movement-a247b01a480)  
36. Lateral Movement \- Riccardo Ancarani \- Red Team Adventures, accessed October 18, 2025, [https://riccardoancarani.github.io/2019-10-04-lateral-movement-megaprimer/](https://riccardoancarani.github.io/2019-10-04-lateral-movement-megaprimer/)  
37. 10 Microsoft service account best practices \- The Quest Blog, accessed October 18, 2025, [https://blog.quest.com/10-microsoft-service-account-best-practices/](https://blog.quest.com/10-microsoft-service-account-best-practices/)  
38. Ten security best practices for Active Directory service accounts \- Specops Software, accessed October 18, 2025, [https://specopssoft.com/blog/service-account-security-best-practices/](https://specopssoft.com/blog/service-account-security-best-practices/)  
39. Best practices for using service accounts | Identity and Access Management (IAM) | Google Cloud, accessed October 18, 2025, [https://cloud.google.com/iam/docs/best-practices-service-accounts](https://cloud.google.com/iam/docs/best-practices-service-accounts)  
40. How to Manage and Secure Service Accounts: Best Practices \- BeyondTrust, accessed October 18, 2025, [https://www.beyondtrust.com/blog/entry/how-to-manage-and-secure-service-accounts-best-practices](https://www.beyondtrust.com/blog/entry/how-to-manage-and-secure-service-accounts-best-practices)  
41. Group Managed Service Accounts overview | Microsoft Learn, accessed October 18, 2025, [https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/group-managed-service-accounts-overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/group-managed-service-accounts-overview)  
42. PowerShell/SecretManagement: PowerShell module to consistent usage of secrets through different extension vaults \- GitHub, accessed October 18, 2025, [https://github.com/PowerShell/SecretManagement](https://github.com/PowerShell/SecretManagement)  
43. How to Set Up SFTP with Public Key Authentication | Aayu Blog, accessed October 18, 2025, [https://aayutechnologies.com/blog/how-to-set-up-sftp-with-public-key-authentication/](https://aayutechnologies.com/blog/how-to-set-up-sftp-with-public-key-authentication/)  
44. Introduction to Active Directory service accounts | Azure Docs, accessed October 18, 2025, [https://docs.azure.cn/en-us/entra/architecture/service-accounts-on-premises](https://docs.azure.cn/en-us/entra/architecture/service-accounts-on-premises)  
45. Managed Service Accounts (MSA): Installing a service \- Advanced Installer, accessed October 18, 2025, [https://www.advancedinstaller.com/managed-service-accounts-installing-service.html](https://www.advancedinstaller.com/managed-service-accounts-installing-service.html)  
46. Using Managed Service Accounts (MSA and gMSA) in Active Directory | Windows OS Hub, accessed October 18, 2025, [https://woshub.com/group-managed-service-accounts-in-windows-server-2012/](https://woshub.com/group-managed-service-accounts-in-windows-server-2012/)  
47. Secure Password Management in PowerShell: Best Practices, accessed October 18, 2025, [https://www.secureideas.com/blog/secure-password-management-in-powershell-best-practices](https://www.secureideas.com/blog/secure-password-management-in-powershell-best-practices)  
48. Overview of the SecretManagement and SecretStore modules \- PowerShell, accessed October 18, 2025, [https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/overview?view=ps-modules](https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/overview?view=ps-modules)  
49. Microsoft.PowerShell.SecretManagement Module, accessed October 18, 2025, [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.secretmanagement/?view=ps-modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.secretmanagement/?view=ps-modules)  
50. Security \- PowerShell Practice and Style \- GitBook, accessed October 18, 2025, [https://poshcode.gitbook.io/powershell-practice-and-style/best-practices/security](https://poshcode.gitbook.io/powershell-practice-and-style/best-practices/security)  
51. How to encrypt credentials & secure passwords with PowerShell | PDQ, accessed October 18, 2025, [https://www.pdq.com/blog/secure-password-with-powershell-encrypting-credentials/](https://www.pdq.com/blog/secure-password-with-powershell-encrypting-credentials/)  
52. Best practice for storing passwords with powershell for use within scripts? \- Reddit, accessed October 18, 2025, [https://www.reddit.com/r/PowerShell/comments/1139tx9/best\_practice\_for\_storing\_passwords\_with/](https://www.reddit.com/r/PowerShell/comments/1139tx9/best_practice_for_storing_passwords_with/)  
53. What's the best practice for passwords in ps scripts? : r/PowerShell \- Reddit, accessed October 18, 2025, [https://www.reddit.com/r/PowerShell/comments/bv7ywa/whats\_the\_best\_practice\_for\_passwords\_in\_ps/](https://www.reddit.com/r/PowerShell/comments/bv7ywa/whats_the_best_practice_for_passwords_in_ps/)  
54. Setting Up SFTP Public Key Authentication On The Command Line | JSCAPE, accessed October 18, 2025, [https://www.jscape.com/blog/setting-up-sftp-public-key-authentication-command-line](https://www.jscape.com/blog/setting-up-sftp-public-key-authentication-command-line)  
55. Key-Based Authentication in OpenSSH for Windows \- Microsoft Learn, accessed October 18, 2025, [https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh\_keymanagement](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)  
56. How to use public/private keys for SSH and SFTP (Windows) \- Krystal Hosting, accessed October 18, 2025, [https://help.krystal.io/cpanel-advanced-topics/how-to-use-public-private-keys-for-ssh-and-sftp-windows](https://help.krystal.io/cpanel-advanced-topics/how-to-use-public-private-keys-for-ssh-and-sftp-windows)  
57. Installing SFTP/SSH Server on Windows using OpenSSH \- WinSCP, accessed October 18, 2025, [https://winscp.net/eng/docs/guide\_windows\_openssh\_server](https://winscp.net/eng/docs/guide_windows_openssh_server)