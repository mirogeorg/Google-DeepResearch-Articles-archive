
# **An Analysis of Application Exclusion Policies in the Winget-AutoUpdate Framework**

### **Executive Summary: The Principle of Proactive Exclusion in Automated Patching**

The inclusion of remote desktop applications within the excluded\_apps.txt file of the Winget-AutoUpdate (WAU) project is not an oversight but a deliberate and necessary risk mitigation strategy. This decision represents a core design philosophy centered on ensuring the stability and reliability of automated software patching in enterprise and enthusiast environments. The exclusion is a proactive measure driven by a confluence of technical challenges and strategic considerations that make unattended updates of these applications inherently risky.

The primary causal factors for this exclusion can be summarized as follows:

1. **Execution Context Conflicts:** A fundamental incompatibility exists between WAU's high-privilege SYSTEM execution context, required for comprehensive system-wide updates, and the user-profile-centric installation and operational model of many remote desktop clients.  
2. **Inherent winget Limitations:** The WAU tool, as an automation wrapper, inherits problematic behaviors from the underlying Windows Package Manager (winget), including the creation of duplicate, side-by-side installations instead of performing clean upgrades for certain packages.  
3. **Application Volatility and In-Use Files:** Remote desktop clients have a high probability of being actively in use during scheduled update cycles, leading to locked files, which invariably cause update processes to fail.  
4. **Critical Infrastructure Risk:** Most significantly, remote access tools are mission-critical infrastructure. A failed automated update could sever remote connectivity for administrators and users, creating a high-severity incident that prevents the very access needed to remediate the problem.

Ultimately, the exclusion list serves as an essential instrument of administrative control. It enables a balanced and prudent approach to patch management, combining the efficiency of broad automation for the majority of applications with targeted, manual oversight for software that is vital to operational continuity.

## **Section 1: The Winget-AutoUpdate (WAU) Framework and Control Mechanisms**

To comprehend the rationale behind excluding specific application classes, it is essential to first understand the operational architecture of Winget-AutoUpdate and the integral role of its control mechanisms.

### **1.1 WAU's Operational Architecture**

Winget-AutoUpdate (WAU) is a powerful automation solution built upon PowerShell scripts that leverages Microsoft's native Windows Package Manager (winget) command-line tool.1 Its primary function is to automate the discovery and installation of application updates on a recurring schedule, typically at user logon or on a daily basis, thereby maintaining software currency with minimal administrative intervention.2 The project is part of a broader ecosystem of tools, including the WAU-Settings-GUI, which provides a user-friendly graphical interface for configuring the core WAU engine.1

A critical architectural feature of WAU is its ability to operate in dual execution contexts. The main scheduled task runs with the high privileges of the NT AUTHORITY\\SYSTEM account to manage machine-wide software installations. Concurrently, it can be configured to run in the logged-on user's context to handle applications installed exclusively within a user profile.2 This dual-context design aims for comprehensive update coverage but, as later sections will detail, is also a significant source of technical complexity and potential update failures.

### **1.2 The excluded\_apps.txt File: A Primary Control Surface**

The excluded\_apps.txt file is the principal mechanism through which administrators can exert control over WAU's behavior. Described in the project's documentation as a "BlockList," its explicit purpose is to prevent WAU from attempting to update designated applications.2 This is particularly useful for applications that must be maintained at a specific version for compatibility reasons or for those with their own built-in auto-update mechanisms that could conflict with WAU.2

The file itself is a simple text list of winget application IDs. It supports the use of wildcards (\*) to match multiple package variants with a single entry, such as using Mozilla.Firefox\* to cover all release channels of the Firefox browser.2 For WAU to recognize this list, the file must be placed either in the same directory as the WAU.msi installer during setup or directly within the WAU installation folder post-installation.2

Furthermore, WAU employs a concept of a *default* exclusion list, which is curated by the project maintainers based on community feedback and known problematic software. A user-provided excluded\_apps.txt file overrides or supplements this default list.6 This explains reports from users who observe that the final, effective exclusion list on a client machine is larger than the one they personally configured, as it represents a merger of the default and custom lists.8 For enterprise deployments, WAU provides sophisticated management options, allowing this list to be defined and distributed centrally via Group Policy Objects (GPO) or a URI specified in the ListPath parameter, which can point to a file on a local network share or a web server.2

The prominence of this feature within the documentation and the extensive development effort dedicated to its flexible management underscore its importance. The exclusion list is not a minor configuration tweak but a fundamental pillar of the WAU design. Its existence demonstrates a core understanding by the developers that a one-size-fits-all, fully automated patching solution is neither feasible nor desirable in real-world environments. They anticipated that administrators would require a robust and adaptable method to opt-out of updating certain applications. This elevates the excluded\_apps.txt file from a simple ignore list to a strategic tool for ensuring the stability of the entire automation process. The default exclusion of remote desktop applications is a direct and logical application of this foundational design philosophy.

## **Section 2: The Technical Rationale: A Multi-Layered Analysis of Update Failures**

The decision to exclude remote desktop applications by default is not based on a single issue but on a convergence of multiple, distinct technical failure modes. These layers of complexity collectively make such applications exceptionally poor candidates for unattended automated updates.

### **2.1 The "In-Use File" Paradox**

Remote desktop clients, by their very function, are frequently left running for extended periods to maintain active sessions for users and administrators. Automated update tools like WAU typically operate on a fixed schedule, creating a high probability that an update attempt will coincide with the application being in use. When an installer attempts to modify or replace binary files that are locked by a running process, the operating system denies access, and the update fails. This exact failure pattern is observed in a parallel scenario involving the PowerShell App Deployment Toolkit (PSADT), where an update to Microsoft Remote Desktop fails because the RdClient.UpdateLib.dll file is locked by the active client process.11 Although the tool is different, the underlying OS behavior is identical and directly applicable. This leads to predictable and recurring update failures in WAU logs, creating administrative noise and leaving critical connectivity tools vulnerable and unpatched.

### **2.2 Execution Context Collision: SYSTEM vs. User Profile**

A primary driver of update failures stems from a conflict in execution contexts. Many modern applications, including the Microsoft Remote Desktop client, are designed to install into the user's local application data folder (e.g., C:\\Users\\username\\AppData\\Local\\Programs) rather than the system-wide Program Files directory. However, WAU's main scheduled task executes as the non-interactive NT AUTHORITY\\SYSTEM account to ensure it can perform privileged operations regardless of which user is logged in.2 This collision leads to several failure modes:

* **Permissions and Ownership:** When a SYSTEM process installs software into a user's profile, it can incorrectly assign file and folder ownership to the Administrators group instead of the user. This can subsequently break the application's own internal update or repair mechanisms, which assume they have write permissions within the user's profile.11  
* **Non-Interactive Session:** The SYSTEM account operates in a non-interactive context (Session 0). The winget tool itself has known limitations when run this way, particularly when managing AppX or MSIX-based packages, which can lead to silent failures or unexpected behavior.12  
* **User-Specific Data:** An update process may require access to registry keys or configuration files located within the user's profile (e.g., under HKEY\_CURRENT\_USER), which are not loaded or accessible within the SYSTEM context.

This context mismatch is a well-understood problem in application packaging and deployment, and it makes user-profile-based applications like many remote desktop clients inherently fragile when managed by a system-level service.

### **2.3 Anomalies in winget Package Handling**

Because WAU is an automation layer built on top of winget, it is entirely subject to the behavior, bugs, and limitations of the underlying package manager. Several documented issues with winget's handling of remote desktop packages make their automated update unreliable:

* **Side-by-Side Installations:** In a reported issue, an attempt to update Microsoft.RemoteDesktopClient via winget resulted not in an in-place upgrade but in the installation of the new version *alongside* the old one.13 This is a known bug in winget's handling of certain package types, and it leads to software inventory pollution and potential application conflicts. The user who encountered this ultimately resolved it by blacklisting the application.  
* **Update Loops:** The same issue thread describes how WAU, driven by incorrect version detection from winget, would get caught in a persistent loop, repeatedly attempting to "update" the Remote Desktop client on every cycle, even after the process had already run.13  
* **Locale Mismatches:** An update for Devolutions.RemoteDesktopManager was observed to fail because winget could not reconcile the installer's locale (en-US) with the system's required locale, erroring out with "Installer locale does not match required locale".14 While this can be resolved with a manual command-line argument, WAU's automated process cannot accommodate such per-package exceptions, resulting in a hard failure.

These are not flaws in WAU's logic but in the winget tool it depends on. From the perspective of the WAU developers, the only stable and reliable solution is to proactively exclude applications known to trigger these winget anomalies.

### **2.4 The Fragility of Critical Infrastructure Tools**

Beyond the technical failures, a strategic risk assessment compels the exclusion of remote desktop clients. These applications are mission-critical infrastructure, providing the essential connectivity for administrators to manage servers and for users to access their work environments. A failed automated update is not a minor inconvenience; it is a potential high-severity incident. If a faulty update incapacitates the RDP client across an enterprise, it could sever all remote access, preventing administrators from fixing the very problem the automation created.

Furthermore, automated updates can conflict with organizational policies. For complex clients like Devolutions Remote Desktop Manager, an application update may require a corresponding backend database update. An administrator forum post reveals a desire to *prevent* winget from upgrading the client to avoid violating version management policies and breaking connectivity to the data source.15 In this context, an unattended update is actively harmful. The risk associated with the automated failure of a critical connectivity tool far outweighs the benefit of keeping it on the latest version via an aggressive, fully automated process.

These factors combine to create a "failure cascade" where a single application can be subject to multiple failure modes simultaneously. For example, the Microsoft Remote Desktop client is often in use (file locking), installs to the user profile (context collision), and is known to have problematic winget upgrade behavior. Any one of these issues would be sufficient cause for exclusion; their combination makes the case overwhelming. This reality necessitates a form of application readiness assessment before enabling auto-updates. The default exclusion of remote desktop applications in WAU serves as a pre-packaged, community-vetted "failed" assessment, saving administrators from discovering these critical failure modes in their production environments.

## **Section 3: Corroborating Evidence from Community and Developer Insights**

The technical rationale for exclusion is strongly supported by real-world experiences documented by the WAU user community and the guidance provided by the project's maintainers. This collective knowledge confirms that excluding remote access tools is a recognized best practice.

### **3.1 User-Reported Failures and Workarounds**

Multiple threads within the project's GitHub repository contain firsthand accounts of failures involving remote desktop clients, with exclusion emerging as the consensus solution.

* **Case Study: Microsoft Remote Desktop (Issue \#551):** One user explicitly reports that WAU is "constantly popping up about updating Remote Desktop, says it's updated but it hasn't," perfectly describing the update loop phenomenon.13 Another contributor in the same thread diagnoses the root cause as winget's poor support for side-by-side packages and concludes, "But I guess I'll blacklist it as you say".13 This demonstrates the community independently identifying the problem and arriving at exclusion as the only practical workaround.  
* **Case Study: Devolutions RDM (Issue \#763 & Forum Post):** A GitHub issue details the technical failure of winget updates due to a locale mismatch.14 Separately, a post on the vendor's own forum shows an administrator actively seeking to block winget updates to comply with internal version control policies, highlighting a business case for exclusion beyond just technical failure.15  
* **Case Study: General Remote Access Tools (TeamViewer, Citrix):** It is common to see a variety of remote access tools in user-configured exclusion lists. One user's Intune deployment blacklist includes TeamViewer.TeamViewer and Citrix.Workspace alongside Microsoft.Teams.Classic and Romanitho.Winget-AutoUpdate itself.16 In another discussion, a maintainer advises that Microsoft Teams is "highly recommended to be excluded" because it is "not well handled by Winget in the system context".6 This links the exclusion of remote access and communication tools directly to the systemic problem of managing user-context applications with a system-level service.

### **3.2 Maintainer Guidance and Established Best Practices**

The project maintainers and key contributors consistently guide users toward using the exclusion list as the intended solution for problematic applications. Their responses and development priorities reinforce that stability through administrative control is favored over attempting universal compatibility.

The explicit recommendation to exclude Microsoft Teams due to its behavior in the SYSTEM context establishes a clear principle that applies equally to many remote desktop clients installed in a similar manner.6 When users report unresolvable update issues with specific applications, the guidance from maintainers and experienced community members is not to pursue a code fix within WAU, but rather to leverage the existing and robust exclusion functionality. This pattern solidifies exclusion as the designed-in resolution for such edge cases. The developers' focus on enhancing control mechanisms, such as robust GPO and ListPath support for managing blocklists, further indicates a design philosophy that prioritizes empowering administrators to manage exceptions over trying to make the core tool flawlessly handle every problematic application in the winget repository.7

This consistent pattern of user reports, community workarounds, and developer guidance transforms the exclusion of remote desktop clients from a series of individual fixes into a recognized, community-vetted best practice. The default exclusion list provided with WAU is the formal codification of this collective knowledge, designed to prevent new users from re-learning these hard-won lessons through service-impacting failures.

## **Section 4: Strategic Guidance for System Administrators**

The evidence clearly indicates that a successful WAU implementation requires a strategic approach to application management rather than universal enablement. The following framework and recommendations provide actionable guidance for administrators.

### **4.1 A Decision Framework for Application Exclusion**

Administrators should not treat WAU as a "set it and forget it" solution for their entire software catalog. Instead, a simple triage process should be applied to any application before it is allowed to be managed by WAU:

1. **Assess Criticality:** Is this application mission-critical for business operations or IT administration (e.g., remote access clients, ERP software, core LOB apps)? If the impact of a failed update is high, the application should be a strong candidate for exclusion.  
2. **Determine Installation Scope:** Does the application install system-wide (e.g., to C:\\Program Files) or on a per-user basis (e.g., to %LOCALAPPDATA%)? Per-user installations are at a significantly higher risk of execution context collisions and should be scrutinized carefully or excluded by default.  
3. **Evaluate In-Use Probability:** Is the application likely to be running during the scheduled update window? Applications with high uptime, such as browsers, communication tools, and remote desktop clients, are prone to file-locking failures and may require a more controlled update strategy.  
4. **Check winget Community Health:** Before trusting WAU to update an application, administrators should search the official microsoft/winget-pkgs GitHub repository for open issues related to its package manifest. A history of problematic or failing updates is a major red flag indicating the package may be unsuitable for automation.

### **4.2 Alternative Management Strategies for Excluded Applications**

Excluding an application from WAU does not mean it should go unpatched. It means a more appropriate and robust tool should be used. Superior methods include:

* **Microsoft Intune (Win32 Apps):** For enterprise environments, packaging critical applications as Win32 apps in Intune offers granular control. This method allows for custom install/uninstall commands, sophisticated detection logic, and a managed user experience (e.g., displaying notifications to close an app before an update). This is the recommended path for most critical enterprise software.  
* **Vendor-Provided Tools:** Many enterprise applications, such as TeamViewer and Devolutions RDM, come with their own central management consoles or policy mechanisms designed specifically for controlling updates across an organization.15 These tools are purpose-built and respect the application's unique requirements.  
* **Configuration Management Systems:** More powerful platforms like Microsoft Configuration Manager (SCCM) or Ansible provide advanced logic for handling complex scenarios, such as scripting pre-update actions (e.g., closing a running process) and post-update validation.  
* **Controlled Manual Updates:** For the most sensitive infrastructure components, updates should be tested in a staging environment and deployed manually or in controlled deployment rings as part of a formal change management process.

### **4.3 Reference Table: Common Remote Desktop Clients and Recommended Update Strategies**

The following table summarizes the known issues and recommended management strategies for common remote desktop clients, serving as a quick-reference guide for administrators.

| Application Name | Winget Package ID | Known Issues with WAU / winget | Recommended Management Strategy |
| :---- | :---- | :---- | :---- |
| Microsoft Remote Desktop | Microsoft.RemoteDesktop | Creates side-by-side installations instead of upgrading; can cause update loops.13 Installs per-user, risking context collision.11 | **Exclude in WAU.** Manage via Microsoft Store policies or deploy as a required Win32 app in Intune for version control. |
| Devolutions RDM | Devolutions.RemoteDesktopManager | winget update can fail due to locale mismatches.14 Automated updates can bypass internal version control policies required for database compatibility.15 | **Exclude in WAU.** Manage versions centrally using the data source settings as recommended by the vendor. |
| TeamViewer | TeamViewer.TeamViewer | Has a robust built-in auto-updater that can conflict with WAU. Often included in user-defined exclusion lists for stability.16 | **Exclude in WAU.** Manage updates and settings centrally via policies in the TeamViewer Management Console. |
| RustDesk | RustDesk.RustDesk | Designed to run without installation, which can complicate winget's version detection and upgrade logic.12 | **Exclude in WAU.** Perform manual updates or use custom scripts for deployment to ensure consistent installation behavior. |
| Citrix Workspace | Citrix.Workspace | Complex enterprise client with many dependencies and configurations. Frequently appears in administrator exclusion lists.16 | **Exclude in WAU.** Deploy and manage using official Citrix administrative packages and Intune/SCCM for precise configuration. |

## **Conclusion: Balancing Automated Efficiency with Administrative Prudence**

The comprehensive analysis of technical failure modes, community reports, and developer guidance confirms that the inclusion of remote desktop applications in WAU's default exclusion list is a deliberate feature, not a flaw. It reflects a mature and pragmatic approach to automation that acknowledges the inherent limitations and risks of a universal, unattended patching model. No single tool can safely and reliably update every application in a diverse software environment.

The most effective automation strategies are those that provide administrators with robust tools for control, visibility, and risk management. The excluded\_apps.txt file is precisely such a tool, allowing for a nuanced policy that separates low-risk, easily updated applications from high-risk, critical infrastructure. By leveraging WAU for the broad majority of software while handling sensitive applications like remote desktop clients with more specialized, controlled methods, administrators can achieve a powerful and sustainable balance: maximizing the efficiency of automation while maintaining the stability, security, and operational integrity of their IT environment.

#### **Works cited**

1. Göran Axel Johannesson KnifMelti \- GitHub, accessed October 29, 2025, [https://github.com/KnifMelti](https://github.com/KnifMelti)  
2. Romanitho/Winget-AutoUpdate: WAU daily updates apps as system and notify connected users. (Allowlist and Blocklist support) \- GitHub, accessed October 29, 2025, [https://github.com/Romanitho/Winget-AutoUpdate](https://github.com/Romanitho/Winget-AutoUpdate)  
3. CanyonITS/PS-Winget-AutoUpdate: WAU daily updates apps as system and notify connected users. (Allowlist and Blocklist support) \- GitHub, accessed October 29, 2025, [https://github.com/CanyonITS/PS-Winget-AutoUpdate](https://github.com/CanyonITS/PS-Winget-AutoUpdate)  
4. Install WAU Settings GUI with winget \- winstall, accessed October 29, 2025, [https://winstall.app/apps/KnifMelti.WAU-Settings-GUI](https://winstall.app/apps/KnifMelti.WAU-Settings-GUI)  
5. \[Bug\]: Handling excluded apps when installing via WinGet · Issue, accessed October 29, 2025, [https://github.com/Romanitho/Winget-AutoUpdate/issues/982](https://github.com/Romanitho/Winget-AutoUpdate/issues/982)  
6. default\_excluded\_apps.txt · Romanitho Winget-AutoUpdate · Discussion \#742 \- GitHub, accessed October 29, 2025, [https://github.com/Romanitho/Winget-AutoUpdate/discussions/742](https://github.com/Romanitho/Winget-AutoUpdate/discussions/742)  
7. \[Bug\]: ADMX files installed in wrong registry location (Intune) · Issue \#748 · Romanitho/Winget-AutoUpdate \- GitHub, accessed October 29, 2025, [https://github.com/Romanitho/Winget-AutoUpdate/issues/748](https://github.com/Romanitho/Winget-AutoUpdate/issues/748)  
8. \[Bug\]: how do i controll the excludelist in Winget · Issue \#803 \- GitHub, accessed October 29, 2025, [https://github.com/Romanitho/Winget-AutoUpdate/issues/803](https://github.com/Romanitho/Winget-AutoUpdate/issues/803)  
9. how to add or update excluded\_apps.txt if use winget to install WAU ? \#731 \- GitHub, accessed October 29, 2025, [https://github.com/Romanitho/Winget-AutoUpdate/issues/731](https://github.com/Romanitho/Winget-AutoUpdate/issues/731)  
10. Winget-AutoUpdate-Intune exclude apps via admx \- Reddit, accessed October 29, 2025, [https://www.reddit.com/r/Intune/comments/16c8bvy/wingetautoupdateintune\_exclude\_apps\_via\_admx/](https://www.reddit.com/r/Intune/comments/16c8bvy/wingetautoupdateintune_exclude_apps_via_admx/)  
11. Deploy Microsoft Remote Desktop \- Auto Update \- PSAppDeployToolkit Community, accessed October 29, 2025, [https://discourse.psappdeploytoolkit.com/t/deploy-microsoft-remote-desktop-auto-update/4352](https://discourse.psappdeploytoolkit.com/t/deploy-microsoft-remote-desktop-auto-update/4352)  
12. Windows Installer · Issue \#100 · rustdesk/rustdesk \- GitHub, accessed October 29, 2025, [https://github.com/rustdesk/rustdesk/issues/100](https://github.com/rustdesk/rustdesk/issues/100)  
13. \[Bug\]: WAU tries to upgrade Windows Desktop Runtime from 6.0.26 to 7.0.15 every day · Issue \#551 · Romanitho/Winget-AutoUpdate \- GitHub, accessed October 29, 2025, [https://github.com/Romanitho/Winget-AutoUpdate/issues/551](https://github.com/Romanitho/Winget-AutoUpdate/issues/551)  
14. \[Bug\]: Installer locale does not match required locale: en-US Required locales: \[\] · Issue \#763 · Romanitho/Winget-AutoUpdate \- GitHub, accessed October 29, 2025, [https://github.com/Romanitho/Winget-AutoUpdate/issues/763](https://github.com/Romanitho/Winget-AutoUpdate/issues/763)  
15. Policy to disable Update still allows to update through Winget, accessed October 29, 2025, [https://forum.devolutions.net/topics/39312/policy-to-disable-update-still-allows-to-update-through-winget](https://forum.devolutions.net/topics/39312/policy-to-disable-update-still-allows-to-update-through-winget)  
16. \[Bug\]: Romanitho.Winget-AutoUpdate is excluded when using Blacklist \#1044 \- GitHub, accessed October 29, 2025, [https://github.com/Romanitho/Winget-AutoUpdate/issues/1044](https://github.com/Romanitho/Winget-AutoUpdate/issues/1044)  
17. MSI Installer · Romanitho Winget-AutoUpdate · Discussion \#582 \- GitHub, accessed October 29, 2025, [https://github.com/Romanitho/Winget-AutoUpdate/discussions/582](https://github.com/Romanitho/Winget-AutoUpdate/discussions/582)