
# **Ransomware Resilience Assessment and Hardening Strategy for Backup Infrastructure**

## **Executive Summary**

This report provides an expert-level assessment of the specified backup server configuration, evaluating its resilience against modern ransomware attack chains. It identifies critical architectural flaws and provides a multi-layered, defense-in-depth framework for remediation.

The current configuration, while demonstrating security awareness, contains a critical vulnerability that undermines its entire defensive posture. The use of persistent credentials stored in the Windows Credential Manager on client machines, combined with an outbound-only SMB firewall rule, creates a direct and authorized pathway for threat actors to compromise the backup server. This architecture is highly susceptible to credential theft and subsequent lateral movement.

The primary vulnerabilities identified are:

1. **Credential Exposure:** Storing backup credentials on endpoints makes them a primary target for common credential harvesting techniques, specifically those identified under MITRE ATT\&CK T1555.004.1  
2. **Architectural Flaw ("Push" Model):** The client-initiated "push" model places the keys to the backup kingdom—the authentication credentials—on the most vulnerable part of the network: the endpoints.2  
3. **Firewall Rule Inversion:** The outbound SMB rule, intended as a defense, becomes an authorized channel for attack once endpoint credentials are stolen, as the malicious traffic will appear legitimate and policy-compliant.4

A strategic shift is required, moving from endpoint-centric authentication to a server-controlled, isolated, and identity-hardened architecture. The core recommendations are to adopt a pull-based architecture, implement robust network segmentation, overhaul credential management by eliminating the use of Windows Credential Manager for service authentication in favor of modern solutions like Group Managed Service Accounts (gMSAs), and achieve data invincibility by implementing the 3-2-1-1-0 backup rule, which incorporates immutable and offline/air-gapped copies.

Implementation of the recommendations outlined in this report will transform the backup infrastructure from a high-risk asset into a hardened, resilient "last line of defense," capable of ensuring business continuity even in the event of a widespread network compromise.

## **Section 1: Analysis of the Current Backup Configuration**

### **1.1 Architectural Overview: A "Push" Model with Perimetric Defenses**

The current configuration is a classic "push" backup model. In this design, the client computers are responsible for initiating the connection and "pushing" their data to the backup server. This is a common and often default configuration for many backup solutions, but it carries inherent security risks when credentials are not managed properly.2 The security controls implemented are layered on top of this push architecture:

* **Disabled SMB Server Service:** This control is intended to prevent the backup server from being used as a distribution point for malware via incoming SMB connections, effectively stopping it from acting as a traditional file server.  
* **Outbound-Only SMB Firewall Rule:** This rule is designed to ensure the backup server can only receive connections initiated by clients and cannot initiate its own SMB connections to other devices on the network. This is a measure to contain the backup server if it were to be compromised.4  
* **Windows Credential Manager:** This component is used to store the unique credentials for the backup server on each remote computer, automating the authentication process for the backup job.

### **1.2 Control Effectiveness Evaluation: Intention vs. Reality**

An evaluation of the implemented controls reveals a significant gap between their intended purpose and their actual effectiveness against a modern, sophisticated ransomware attack.

* **Disabled SMB Server Service:** This is a sound hardening practice. It reduces the attack surface of the backup server by disabling a service commonly exploited for lateral movement and malware propagation.5 However, its effectiveness is limited to *inbound* SMB-based attacks. An attacker who gains administrative control of the server could simply re-enable this service or use other protocols and tools to attack the network, such as PowerShell Remoting or other post-exploitation frameworks.7  
* **Outbound-Only SMB Firewall Rule:** The intention is to create a one-way data flow. While this rule effectively prevents the backup server from *initiating* SMB connections, it does nothing to inspect or block legitimate-looking, but maliciously controlled, connections originating from clients. The firewall will see a valid, authenticated SMB session from a client and permit it, regardless of whether a user or malware is driving that session.4 The security controls, while individually logical, create a false sense of security due to a fundamental misunderstanding of the threat model. The focus is on protecting the backup server from external, unauthenticated attacks, while the most probable attack vector is an internal, authenticated attack originating from a compromised endpoint.  
* **Windows Credential Manager:** This is the configuration's **Achilles' heel**. While convenient for automation, storing persistent credentials for a critical service on dozens or hundreds of endpoints creates a massive, distributed attack surface. Endpoints are the most common point of initial compromise in ransomware attacks.8 The Windows Credential Manager (also known as the Windows Vault) is a well-documented target for credential theft by numerous malware families and post-exploitation tools.1

The entire defensive structure is built on the premise of preventing an unauthenticated attack from reaching the backup server. However, the most likely ransomware attack scenario does not begin this way. It begins with the compromise of a single client, followed by the theft of its stored backup credentials. Using these credentials, the attacker can launch a fully authenticated, legitimate-looking SMB session to the backup server. The firewall will permit this traffic because it originates from an allowed client. At this point, the entire defensive structure is bypassed.

## **Section 2: Threat Model: The Anatomy of an Attack**

### **2.1 Stage 1: Initial Compromise of the Endpoint**

The attack chain will almost certainly begin with the compromise of a standard user workstation, not a direct assault on the hardened backup server. Ransomware operators consistently use well-established initial access vectors to gain a foothold within a target network. Common methods include:

* **Phishing Emails:** Malicious attachments or links designed to trick a user into executing malware. This remains one of the most successful vectors for initial access.8  
* **Exploitation of Vulnerabilities:** Targeting unpatched software on the endpoint, such as browsers, applications, or the operating system itself. Attackers actively scan for and exploit known vulnerabilities.8  
* **Remote Desktop Protocol (RDP):** Exploiting weak RDP credentials or vulnerabilities to gain remote access, a technique favored for its directness.10

Once an attacker gains a foothold on a client machine, even with standard user privileges, the next phase of the attack begins: privilege escalation and credential harvesting.

### **2.2 Stage 2: The Primary Attack Vector \- Credential Theft from the Endpoint**

With code execution on the client machine, the attacker's immediate goal is to harvest the credentials stored in the Windows Credential Manager. This is a high-value, low-effort target that provides the keys needed for lateral movement.11

Threat actors employ a variety of sophisticated tools and techniques to achieve this, many of which are mapped under the MITRE ATT\&CK framework as sub-technique T1555.004: Credentials from Password Stores: Windows Credential Manager.1 The threat is not theoretical; it is a standard operating procedure for numerous cybercriminal groups.

| Technique/Tool | Description | MITRE ATT\&CK ID | Real-World Malware/Group Examples |
| :---- | :---- | :---- | :---- |
| **Mimikatz** | A post-exploitation tool that can dump credentials from LSASS memory and directly query the Windows Vault to extract plaintext passwords and hashes. | T1555.004 | Used by numerous ransomware groups including 8BASE, Black Basta, and PLAY.1 |
| **LaZagne** | An open-source password recovery tool that extracts credentials from various sources, including directly from Vault files by abusing the DPAPI. | T1555.004 | Used by groups like Akira and 8BASE for comprehensive credential harvesting.1 |
| **PowerSploit** | A PowerShell framework containing modules like Invoke-WCMDump specifically for enumerating and extracting credentials from the Credential Manager. | T1555.004 | Used by advanced persistent threat (APT) and ransomware groups like Wizard Spider.1 |
| **Native vaultcmd.exe** | A legitimate Windows executable used by attackers for reconnaissance to list stored credentials and identify high-value targets. | T1555.004 | Used by malware such as Lizar for initial credential discovery.1 |
| **Windows API Abuse** | Direct programmatic calls to Windows APIs like CredEnumerateW and CredReadW to list and retrieve stored credentials without using external tools. | T1555.004 | Used by malware like Lizar and ROKRAT to stealthily steal credentials.1 |

### **2.3 Stage 3: Pivoting Through the Firewall \- Lateral Movement via Authorized SMB**

Once the attacker possesses the unique credentials for the backup server, the outbound-only firewall rule becomes irrelevant. The attacker, now operating from the compromised client machine, uses the stolen credentials to initiate a standard SMB connection to the backup server's IP address on port 445\.

From the firewall's perspective, this is a legitimate, policy-compliant connection: it is *outbound* from the client and *inbound* to the backup server, which is an allowed traffic flow. The attacker now has authenticated access to the backup repository. They can begin to encrypt, delete, or exfiltrate the backup data, rendering it useless for recovery.4 This is a primary objective for modern ransomware groups, as crippling the victim's ability to recover increases the likelihood of a ransom payment.4

While the practice of assigning unique credentials to each client appears to enhance security by segmenting access, this approach fails to protect the central repository. An attacker only needs to compromise a single endpoint to acquire a valid key to the entire backup dataset, rendering the client-level segmentation ineffective against the primary threat. Once the attacker is on the backup server, they have access to the data from *all* clients, making the uniqueness of the initial credential moot.

### **2.4 Scenario Analysis: The Weaponized Backup Server**

Should the attacker gain administrative control over the backup server itself, the disabled SMB server service offers only momentary protection. An attacker could:

* **Re-enable Services:** Simply re-enable the "Server" service (LanmanServer) to turn the backup server into a malware distribution hub.  
* **Install Other Tools:** Use post-exploitation tools like PsExec or Evil-WinRM to push malware or ransomware to other clients on the network, using protocols other than SMB.7  
* **Poison Backups:** Subtly corrupt future backups over a period of weeks or months. This "timebomb" attack ensures that even if the initial attack is detected, the recovery data is also compromised and useless.2

A compromised backup server can become a devastating lateral movement pivot point. Security teams often deploy Endpoint Detection and Response (EDR) solutions on workstations, but backup servers may have less stringent monitoring. An attacker who compromises the backup server can use it as a "clean" source to launch attacks, potentially bypassing security tools that are less likely to scrutinize traffic from a trusted infrastructure server.

## **Section 3: A Defense-in-Depth Framework for Ransomware Resilience**

A resilient backup architecture requires a multi-layered, defense-in-depth strategy. This framework moves beyond simple perimeter controls to create a hardened, isolated, and intelligent system capable of withstanding a sophisticated attack.

### **3.1 Layer 1: Foundational Infrastructure Hardening**

#### **3.1.1 Architectural Shift: Transitioning to a Pull-Based Backup Model**

The most significant architectural improvement is to shift from a "push" to a "pull" model. In a pull model, the hardened backup server initiates connections to the clients to "pull" data from them.2 This fundamentally alters the security posture by centralizing control and credentials on the most secure component of the system.

The security advantages are profound. Credentials for client access are stored only on the secure backup server, not on the vulnerable endpoints, eliminating the primary attack vector.3 A compromised client has no knowledge of the backup server's credentials and no ability to initiate a connection to it.2 Firewall rules also become more intuitive and secure, allowing the backup server's security zone to make *outbound* connections while blocking all *inbound* initiated connections.

| Security Criterion | Push Model (Current State) | Pull Model (Recommended State) |
| :---- | :---- | :---- |
| **Credential Storage Location** | Endpoint (High Risk) | Backup Server (Low Risk) |
| **Network Connection Initiator** | Client | Backup Server |
| **Primary Attack Surface** | All Endpoints | Single Backup Server |
| **Firewall Logic** | Allow Inbound to Backup Server | Allow Outbound from Backup Server |
| **Impact of Endpoint Compromise** | High (Credentials Stolen, Server Access) | Low (No Credentials to Steal, No Server Access) |

#### **3.1.2 Network Isolation: Implementing a Dedicated Backup Security Zone**

The backup infrastructure must not reside on the same network segment as user workstations or general-purpose servers. It must be isolated in its own security zone, typically using a dedicated VLAN, to prevent lateral movement.4

Implementation best practices include:

* **VLANs and Subnets:** Place all backup components (server, repositories) in a dedicated VLAN with a unique subnet.23  
* **Firewall Enforcement:** All traffic into and out of this backup VLAN must pass through a firewall that inspects all traffic, treating it as untrusted.22  
* **Strict ACLs:** Implement Access Control Lists (ACLs) that enforce the Principle of Least Privilege. For a pull model, this means denying all inbound traffic by default and only allowing outbound traffic from the backup server to specific client IPs on the required backup agent port.23  
* **No Internet Access:** The backup VLAN should have no direct inbound or outbound internet access.4

#### **3.1.3 Protocol Security: Advanced SMB Hardening**

If SMB must be used for any part of the backup or recovery process, it must be hardened according to CISA guidelines to prevent exploitation.6 Key steps include enforcing the use of SMBv3.1.1, which offers pre-authentication integrity and stronger encryption, and requiring SMB encryption for all data in transit. If encryption is not possible, SMB signing should be enforced to ensure packet integrity.

### **3.2 Layer 2: Fortifying Identity and Access Management**

#### **3.2.1 Credential Management Overhaul: Decommissioning Windows Credential Manager**

The immediate priority is to stop using the Windows Credential Manager for storing service account credentials for network authentication. This can be enforced via Group Policy by enabling the setting: "Network access: Do not allow storage of passwords and credentials for network authentication." This is the primary mitigation recommended by MITRE for T1555.004.1

#### **3.2.2 Implementing the Principle of Least Privilege (PoLP)**

The Principle of Least Privilege (PoLP) dictates that any account or process should have only the minimum permissions necessary to perform its function.26 For backup service accounts, this means the account should have *read-only* access to the source data on clients and *write/append-only* permissions to the backup repository. It should explicitly be denied permissions to delete or modify existing backup files and must not be a member of any administrative groups.4

#### **3.2.3 Modern Authentication: Leveraging Managed Service Accounts**

Instead of using standard user accounts with static passwords, the system should be transitioned to modern, secure service account types provided by Windows Server.30

**Group Managed Service Accounts (gMSAs)** are the ideal solution. These are Active Directory accounts whose passwords are automatically managed and periodically changed by domain controllers. The password is a complex, 240-character string that is never known to administrators and is never stored on disk in a retrievable format.30 This completely eliminates the problem of static, stored credentials. Since the password is unknown and managed by AD, it cannot be stolen from a configuration file or credential manager and used for lateral movement.30

### **3.3 Layer 3: Achieving Data Invincibility**

#### **3.3.1 The Gold Standard: Adopting the 3-2-1-1-0 Backup Strategy**

The traditional 3-2-1 rule (3 copies, 2 media types, 1 offsite) is no longer sufficient against ransomware that actively targets backups.33 The modern **3-2-1-1-0** strategy adds two critical layers for resilience 4:

* **3** Copies of Data  
* **2** Different Media  
* **1** Copy Offsite  
* **\+1 Immutable Copy:** At least one copy of the backups must be immutable—meaning it cannot be altered, encrypted, or deleted for a defined period, even by an administrator.  
* **\+0 Errors:** The recovery process must be regularly tested to ensure backups are valid and can be restored successfully (0 errors).

#### **3.3.2 Implementing Immutable Backup Repositories (Logical Air Gaps)**

Immutability is the ultimate defense against ransomware attempting to encrypt backup files. It ensures a clean, untouched version of the data is always available for recovery.19 This can be achieved through a Linux Hardened Repository using filesystem attributes, object storage with S3 Object Lock (a WORM policy), or features built into modern backup solutions.3

#### **3.3.3 The Ultimate Failsafe: Incorporating Offline, Air-Gapped Storage**

An air-gapped backup is one that is physically or logically disconnected from the network, providing the highest level of protection.36

* **Physical Air Gap:** Copying backups to removable media like tapes or external hard drives and storing them offline and offsite. This is the most secure but often the slowest to recover from.44  
* **Logical Air Gap:** Using a cloud-based service that isolates backup data in a separate security domain, often managed by the vendor. This provides the security of an air gap with the speed of cloud recovery.4

This defense-in-depth strategy creates a cascading series of failures for an attacker. Compromising an endpoint yields no credentials. The backup server is unreachable due to network segmentation. Even if the server itself is compromised via a zero-day exploit, the primary backups cannot be altered due to immutability, and a final, offline copy exists as a failsafe.

## **Section 4: Strategic Recommendations and Implementation Roadmap**

### **4.1 Operational Imperatives: Beyond Technology**

Technology alone is insufficient. A resilient backup strategy must be supported by robust operational processes.

* **Backup Verification and Recovery Drills:** Backups are useless if they cannot be restored. A schedule for regular, automated backup integrity checks and quarterly recovery drills must be implemented to test the entire restoration process.36 This validates the "0" in the 3-2-1-1-0 rule.35  
* **Continuous Monitoring and Alerting:** Implement robust logging on the backup server and the firewall protecting its VLAN. Monitor for anomalous activity, such as failed login attempts, attempts to change backup jobs or retention policies, or unusual data access patterns. Alerts should be sent via a mechanism that is separate from standard corporate email, which may be compromised during an attack.47  
* **Patch Management:** Keep all components of the backup infrastructure—the server OS, the backup software, and any related hardware firmware—fully patched and up to date to protect against known vulnerabilities.8

### **4.2 Prioritized Action Plan**

The following roadmap provides a prioritized sequence of actions to transition from the current high-risk configuration to a hardened, ransomware-resilient architecture.

| Priority | Recommendation | Risk Mitigated | Estimated Effort | Relevant Sections |
| :---- | :---- | :---- | :---- | :---- |
| **CRITICAL** | Disable storage of network credentials in Windows Credential Manager via GPO. | Primary vector for credential theft and lateral movement. | Low | 2.2, 3.2.1 |
| **CRITICAL** | Isolate the backup server in a new, dedicated VLAN with a restrictive firewall policy. | Uncontrolled lateral movement to the backup server. | Medium | 3.1.2 |
| **HIGH** | Re-architect the backup process to a "pull" model. | Eliminates credential storage on endpoints. | High | 3.1.1 |
| **HIGH** | Replace static service account with a Group Managed Service Account (gMSA). | Eliminates static passwords, automates credential management. | Medium | 3.2.3 |
| **HIGH** | Implement an immutable backup repository (e.g., Linux Hardened Repo, S3 Object Lock). | Ransomware encrypting/deleting online backups. | Medium-High | 3.3.2 |
| **MEDIUM** | Implement an offline/air-gapped backup copy (e.g., tape or dedicated cloud vault). | Total compromise of the primary site and online backups. | Medium | 3.3.3 |
| **MEDIUM** | Formalize and schedule quarterly recovery drills and backup verification tests. | Backup corruption, failed restores during an incident. | Low (Operational) | 4.1 |

## **Conclusion**

The current backup configuration, despite its well-intentioned controls, is critically vulnerable to modern ransomware attack chains due to its reliance on a "push" architecture and the storage of credentials on endpoints. An attacker who compromises a single client machine can readily steal these credentials, bypass all existing defenses, and gain full control over the backup repository, rendering it useless for recovery.

A fundamental strategic shift is required to achieve genuine ransomware resilience. This involves moving from a perimeter-based defense of a flawed architecture to a defense-in-depth model built on secure principles. By transitioning to a server-initiated "pull" model, isolating the infrastructure within a dedicated security zone, replacing static passwords with automatically managed service accounts, and ensuring data invincibility through immutable and air-gapped copies, the backup system can be transformed from a primary target into a true last line of defense. The implementation of the prioritized recommendations in this report will significantly harden the backup infrastructure, ensuring its ability to facilitate rapid and reliable recovery, thereby safeguarding business continuity in the face of a catastrophic cyberattack.

#### **Works cited**

1. Credentials from Password Stores: Windows Credential Manager ..., accessed October 17, 2025, [https://attack.mitre.org/techniques/T1555/004/](https://attack.mitre.org/techniques/T1555/004/)  
2. A Conversation about Backup Strategies and PBS | Proxmox ..., accessed October 17, 2025, [https://forum.proxmox.com/threads/a-conversation-about-backup-strategies-and-pbs.106601/](https://forum.proxmox.com/threads/a-conversation-about-backup-strategies-and-pbs.106601/)  
3. Protection against Ransomware by Pulling Backups \- Feature \- Duplicacy Forum, accessed October 17, 2025, [https://forum.duplicacy.com/t/protection-against-ransomware-by-pulling-backups/2852](https://forum.duplicacy.com/t/protection-against-ransomware-by-pulling-backups/2852)  
4. Backups Are Under Attack: How to Protect Your Backups, accessed October 17, 2025, [https://thehackernews.com/2025/06/how-to-protect-your-backups-from-ransomware-attacks.html](https://thehackernews.com/2025/06/how-to-protect-your-backups-from-ransomware-attacks.html)  
5. \#StopRansomware: Ghost (Cring) Ransomware \- CISA, accessed October 17, 2025, [https://www.cisa.gov/news-events/cybersecurity-advisories/aa25-050a](https://www.cisa.gov/news-events/cybersecurity-advisories/aa25-050a)  
6. \#StopRansomware Guide | CISA, accessed October 17, 2025, [https://www.cisa.gov/stopransomware/ransomware-guide](https://www.cisa.gov/stopransomware/ransomware-guide)  
7. Storm-0501's evolving techniques lead to cloud-based ransomware | Microsoft Security Blog, accessed October 17, 2025, [https://www.microsoft.com/en-us/security/blog/2025/08/27/storm-0501s-evolving-techniques-lead-to-cloud-based-ransomware/](https://www.microsoft.com/en-us/security/blog/2025/08/27/storm-0501s-evolving-techniques-lead-to-cloud-based-ransomware/)  
8. Ransomware: Types, Examples & Removal Tactics \- Fortinet, accessed October 17, 2025, [https://www.fortinet.com/resources/cyberglossary/ransomware](https://www.fortinet.com/resources/cyberglossary/ransomware)  
9. Ransomware Protection Solutions: Email, App, Data | Barracuda Networks, accessed October 17, 2025, [https://www.barracuda.com/solutions/ransomware](https://www.barracuda.com/solutions/ransomware)  
10. A Guide To Preventing Ransomware Attacks for SMBs | Spin.AI, accessed October 17, 2025, [https://spin.ai/blog/ransomware-attacks-for-smbs/](https://spin.ai/blog/ransomware-attacks-for-smbs/)  
11. Dancing with Windows Credential Manager | by S12 \- 0x12Dark ..., accessed October 17, 2025, [https://medium.com/@s12deff/dancing-with-windows-credential-manager-1e16d1b20354](https://medium.com/@s12deff/dancing-with-windows-credential-manager-1e16d1b20354)  
12. Credential Dumping: How ransomware gangs steal login data and how to detect it, accessed October 17, 2025, [https://www.threatdown.com/blog/credential-dumping-how-ransomware-gangs-steal-login-data-and-how-to-detect-it/](https://www.threatdown.com/blog/credential-dumping-how-ransomware-gangs-steal-login-data-and-how-to-detect-it/)  
13. Ransomware Attack \- What is it and How Does it Work? \- Check Point Software, accessed October 17, 2025, [https://www.checkpoint.com/cyber-hub/threat-prevention/ransomware/](https://www.checkpoint.com/cyber-hub/threat-prevention/ransomware/)  
14. \#StopRansomware: Blacksuit (Royal) Ransomware | CISA, accessed October 17, 2025, [https://www.cisa.gov/news-events/cybersecurity-advisories/aa23-061a](https://www.cisa.gov/news-events/cybersecurity-advisories/aa23-061a)  
15. What is a Lateral Movement? Prevention and Detection Methods \- Fortinet, accessed October 17, 2025, [https://www.fortinet.com/resources/cyberglossary/lateral-movement](https://www.fortinet.com/resources/cyberglossary/lateral-movement)  
16. CISA Alerts on Actively Exploited Windows Improper Access Control Flaw \- GBHackers, accessed October 17, 2025, [https://gbhackers.com/cisa-alerts-on-windows-improper-access-control-flaw/](https://gbhackers.com/cisa-alerts-on-windows-improper-access-control-flaw/)  
17. Two New Windows Zero-Days Exploited in the Wild — One Affects Every Version Ever Shipped \- The Hacker News, accessed October 17, 2025, [https://thehackernews.com/2025/10/two-new-windows-zero-days-exploited-in.html](https://thehackernews.com/2025/10/two-new-windows-zero-days-exploited-in.html)  
18. Windows Credential Manager | The Hacker Recipes, accessed October 17, 2025, [https://www.thehacker.recipes/ad/movement/credentials/dumping/windows-credential-manager](https://www.thehacker.recipes/ad/movement/credentials/dumping/windows-credential-manager)  
19. Immutable Backups & Their Role in Cyber Resilience \- Veeam, accessed October 17, 2025, [https://www.veeam.com/blog/immutable-backup.html](https://www.veeam.com/blog/immutable-backup.html)  
20. Lateral Movement in Ransomware: Cybersecurity Definition | Halcyon.ai, accessed October 17, 2025, [https://www.halcyon.ai/faqs/what-is-lateral-movement](https://www.halcyon.ai/faqs/what-is-lateral-movement)  
21. Creating backup process with a pi, should I use push or pull? \- Super User, accessed October 17, 2025, [https://superuser.com/questions/1772610/creating-backup-process-with-a-pi-should-i-use-push-or-pull](https://superuser.com/questions/1772610/creating-backup-process-with-a-pi-should-i-use-push-or-pull)  
22. Building a Ransomware Resilient Architecture \- eSecurity Planet, accessed October 17, 2025, [https://www.esecurityplanet.com/networks/building-a-ransomware-resilient-architecture/](https://www.esecurityplanet.com/networks/building-a-ransomware-resilient-architecture/)  
23. Top 10 Network Segmentation Best Practices | NinjaOne, accessed October 17, 2025, [https://www.ninjaone.com/blog/network-segmentation-best-practices/](https://www.ninjaone.com/blog/network-segmentation-best-practices/)  
24. Network Segmentation Security Best Practices \- Check Point Software, accessed October 17, 2025, [https://www.checkpoint.com/cyber-hub/network-security/what-is-network-segmentation/network-segmentation-security-best-practices/](https://www.checkpoint.com/cyber-hub/network-security/what-is-network-segmentation/network-segmentation-security-best-practices/)  
25. Segmentation | Veeam Backup & Replication Security Best Practice Guide, accessed October 17, 2025, [https://bp.veeam.com/security/Design-and-implementation/Hardening/Segmentation.html](https://bp.veeam.com/security/Design-and-implementation/Hardening/Segmentation.html)  
26. Least privilege for Cloud Functions using Cloud IAM | Google Cloud Blog, accessed October 17, 2025, [https://cloud.google.com/blog/products/application-development/least-privilege-for-cloud-functions-using-cloud-iam](https://cloud.google.com/blog/products/application-development/least-privilege-for-cloud-functions-using-cloud-iam)  
27. Principle of least privilege \- Wikipedia, accessed October 17, 2025, [https://en.wikipedia.org/wiki/Principle\_of\_least\_privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege)  
28. What is the Principle of Least Privilege (PoLP)? \- ARMO, accessed October 17, 2025, [https://www.armosec.io/glossary/principle-of-least-privilege-polp/](https://www.armosec.io/glossary/principle-of-least-privilege-polp/)  
29. Principle of Least Privilege Examples | With Diagrams \- Delinea, accessed October 17, 2025, [https://delinea.com/blog/principle-of-least-privilege-examples](https://delinea.com/blog/principle-of-least-privilege-examples)  
30. Service Accounts in Windows Server | Microsoft Learn, accessed October 17, 2025, [https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts)  
31. How to Manage and Secure Service Accounts: Best Practices \- BeyondTrust, accessed October 17, 2025, [https://www.beyondtrust.com/blog/entry/how-to-manage-and-secure-service-accounts-best-practices](https://www.beyondtrust.com/blog/entry/how-to-manage-and-secure-service-accounts-best-practices)  
32. Local System Account and Windows Credential Manager \- Stack Overflow, accessed October 17, 2025, [https://stackoverflow.com/questions/58719473/local-system-account-and-windows-credential-manager](https://stackoverflow.com/questions/58719473/local-system-account-and-windows-credential-manager)  
33. www.nccoe.nist.gov, accessed October 17, 2025, [https://www.nccoe.nist.gov/sites/default/files/legacy-files/msp-protecting-data-extended.pdf](https://www.nccoe.nist.gov/sites/default/files/legacy-files/msp-protecting-data-extended.pdf)  
34. The 3-2-1 Backup Rule: A Proven Strategy for Data Protection | Cohesity, accessed October 17, 2025, [https://www.cohesity.com/glossary/321-backup-rule/](https://www.cohesity.com/glossary/321-backup-rule/)  
35. How to Implement the 3-2-1 Backup Rule for Cloud Data | CO- by ..., accessed October 17, 2025, [https://www.uschamber.com/co/run/technology/3-2-1-backup-rule](https://www.uschamber.com/co/run/technology/3-2-1-backup-rule)  
36. Achieving Ransomware Resilience | Commvault, accessed October 17, 2025, [https://www.commvault.com/explore/ransomware-resilience](https://www.commvault.com/explore/ransomware-resilience)  
37. 6 components of a ransomware backup strategy \- ConnectWise, accessed October 17, 2025, [https://www.connectwise.com/blog/6-components-of-a-ransomware-backup-strategy](https://www.connectwise.com/blog/6-components-of-a-ransomware-backup-strategy)  
38. Immutable Backup | Defined and Explained \- Cohesity, accessed October 17, 2025, [https://www.cohesity.com/glossary/immutable-backup/](https://www.cohesity.com/glossary/immutable-backup/)  
39. Immutable Backups: How It Works, Pros/Cons, and Best Practices \- N2W Software, accessed October 17, 2025, [https://n2ws.com/blog/immutable-backups-how-it-works-pros-cons-and-best-practices](https://n2ws.com/blog/immutable-backups-how-it-works-pros-cons-and-best-practices)  
40. How Immutable Backups Protect Against Data Loss \- TierPoint, accessed October 17, 2025, [https://www.tierpoint.com/blog/immutable-backups/](https://www.tierpoint.com/blog/immutable-backups/)  
41. 7 Ways to Secure Backup Data \- Adivi, accessed October 17, 2025, [https://adivi.com/blog/ways-to-secure-backup-data/](https://adivi.com/blog/ways-to-secure-backup-data/)  
42. What is an Air Gap Backup? | IBM, accessed October 17, 2025, [https://www.ibm.com/think/topics/air-gap-backup](https://www.ibm.com/think/topics/air-gap-backup)  
43. Air Gap: A Critical Security Measure for Data Protection \- Cohesity, accessed October 17, 2025, [https://www.cohesity.com/glossary/air-gap/](https://www.cohesity.com/glossary/air-gap/)  
44. Air Gap Backup | Explore \- Commvault, accessed October 17, 2025, [https://www.commvault.com/explore/air-gap-backup](https://www.commvault.com/explore/air-gap-backup)  
45. Air Gap Backup: A Complete Guide | CloudAlly, accessed October 17, 2025, [https://www.cloudally.com/glossary/what-is-air-gap-backup-complete-guide/](https://www.cloudally.com/glossary/what-is-air-gap-backup-complete-guide/)  
46. Backing up your data \- NCSC.GOV.UK, accessed October 17, 2025, [https://www.ncsc.gov.uk/collection/top-tips-for-staying-secure-online/always-back-up-your-most-important-data](https://www.ncsc.gov.uk/collection/top-tips-for-staying-secure-online/always-back-up-your-most-important-data)  
47. Principles for ransomware-resistant cloud backups \- NCSC.GOV.UK, accessed October 17, 2025, [https://www.ncsc.gov.uk/collection/ransomware-resistant-backups/principles-for-ransomware-resistant-cloud-backups](https://www.ncsc.gov.uk/collection/ransomware-resistant-backups/principles-for-ransomware-resistant-cloud-backups)  
48. 10 Essential Steps to Secure Your Data Backups \- CommandLink, accessed October 17, 2025, [https://www.commandlink.com/10-steps-to-secure-your-data-backups/](https://www.commandlink.com/10-steps-to-secure-your-data-backups/)  
49. Backup & Secure | U.S. Geological Survey \- USGS.gov, accessed October 17, 2025, [https://www.usgs.gov/data-management/backup-secure](https://www.usgs.gov/data-management/backup-secure)