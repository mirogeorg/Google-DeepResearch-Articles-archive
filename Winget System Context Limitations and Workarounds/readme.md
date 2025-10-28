
# **Mastering WinGet in the Enterprise: A Deep Dive into SYSTEM Context Execution, Official Advancements, and Automation Strategies**

## **The Architectural Impasse: Why WinGet and the SYSTEM Context Historically Collide**

The Windows Package Manager (WinGet) has emerged as a powerful command-line tool for streamlining software management on Windows. However, its adoption within enterprise environments has been historically hindered by a fundamental architectural challenge: its inability to natively operate under the NT AUTHORITY\\SYSTEM security context. This limitation is not a simple bug but a direct consequence of WinGet's design philosophy and delivery mechanism, which were initially tailored for interactive, user-centric scenarios rather than the non-interactive, machine-wide automation required by enterprise management tools like Microsoft Intune and various Remote Monitoring and Management (RMM) platforms.1 Understanding this foundational conflict is crucial to appreciating the evolution of community workarounds and the eventual release of an official, supported solution from Microsoft.

### **WinGet's Origin: A User-Centric Tool Delivered as an MSIX Package**

The root of the SYSTEM context issue lies in WinGet's distribution method. The winget.exe client is not a standalone executable in the traditional sense; it is delivered as a core component of the "App Installer," an MSIX package distributed and updated via the Microsoft Store.3 The MSIX packaging format is a modern application deployment technology designed primarily for user-based installations. These packages are registered on a per-user basis, ensuring that applications are cleanly installed, updated, and removed without leaving behind residual system modifications, a stark contrast to the machine-wide scope typically managed by the SYSTEM account.5

This user-centric model introduces a critical dependency that complicates automated deployments. WinGet's availability on a system is often contingent upon an asynchronous registration process initiated by the Microsoft Store, which typically occurs only after a user has logged in for the first time.3 This creates a significant timing problem and a potential race condition for automation scripts, particularly in pre-login provisioning scenarios such as those executed during the "Device setup" phase of an Intune Autopilot deployment. A script running as SYSTEM on a freshly provisioned device may fail not only because of context-related issues but because the underlying Windows Package Manager service has not yet been registered by the Store, rendering WinGet completely unavailable.7

### **The Role of App Execution Aliases and the Environment PATH**

The second major hurdle stems from how the winget command is made available in a command shell. It is not added to the system-wide PATH environment variable in a conventional location like C:\\Windows\\System32. Instead, WinGet relies on a feature of modern applications known as an "App Execution Alias." This mechanism creates a lightweight link to the actual executable, but it does so within a user-specific directory: %LOCALAPPDATA%\\Microsoft\\WindowsApps.4 When a user opens a command prompt, this path is part of their environment, allowing them to simply type winget and have the system resolve it correctly.

The SYSTEM account, however, operates in a fundamentally different environment. It does not have a standard user profile and therefore lacks the %LOCALAPPDATA% folder structure that a logged-in user possesses. Consequently, the SYSTEM account's PATH variable does not include the WindowsApps directory where the alias resides.9 This is the direct and immediate reason why executing the winget command from a shell running as SYSTEM results in the familiar "The term 'winget' is not recognized as the name of a cmdlet, function, script file, or operable program" error.4 The command is, from the perspective of the SYSTEM account's environment, simply not found.

### **The SYSTEM Account: A Privileged but Isolated Context**

The NT AUTHORITY\\SYSTEM account is one of the most powerful built-in accounts on a Windows system. It is used by the operating system kernel and various system services, granting it extensive privileges over local resources. It operates in a non-interactive environment known as "Session 0," which is isolated from the interactive desktop sessions of logged-in users.1

Despite its high level of privilege, which includes full read and execute permissions on the C:\\Program Files\\WindowsApps directory where the actual winget.exe binary is stored, the SYSTEM account lacks the necessary contextual framework to interact with MSIX packages as they were designed.5 It cannot register these user-centric packages for its own use, nor can it resolve the App Execution Aliases that depend on a user profile. This creates the central paradox: the account with the most power on the system is unable to use a tool that requires a lower-privileged, but contextually complete, user environment. The demand from enterprise administrators to use WinGet for automated, elevated tasks via tools like Intune directly clashed with this original design, exposing a significant gap between the tool's initial conception and its potential for enterprise-scale management.2

## **Bridging the Gap: Community-Driven Workarounds and Their Inherent Fragility**

Faced with the architectural limitations of WinGet, the IT administrator community did not wait for an official solution. Instead, a series of ingenious workarounds were developed to force winget.exe to run in the SYSTEM context. While these methods demonstrated the strong enterprise demand for this functionality, they were fundamentally flawed, treating a deep-seated context problem as a simple pathing issue. This approach led to solutions that were brittle, inconsistent, and ultimately unsuitable for reliable production use.

### **The Primary Workaround: Dynamic Path Resolution**

The most common workaround involved bypassing the App Execution Alias entirely and calling the winget.exe executable directly. Since the exact path to the executable changes with every update to the App Installer package, hardcoding the path was not a viable long-term strategy.12 To overcome this, administrators developed PowerShell scripts to dynamically locate the executable at runtime.

These scripts typically used cmdlets like Get-ChildItem or Resolve-Path to recursively search the C:\\Program Files\\WindowsApps directory for the winget.exe file. A common implementation would filter for directories matching the pattern Microsoft.DesktopAppInstaller\_\* and then locate the executable within the latest version of that directory.5

A representative script might look like this:

PowerShell

$wingetPath \= (Get-Childitem \-Path "C:\\Program Files\\WindowsApps\\Microsoft.DesktopAppInstaller\_\*" \-Include winget.exe \-Recurse \-ErrorAction SilentlyContinue | Select-Object \-Last 1).FullName  
if ($wingetPath) {  
    & $wingetPath upgrade \-\-all \-\-silent  
}

This approach successfully located the binary, allowing scripts running as SYSTEM to invoke it. However, simply finding the file was only the first step, and this method quickly revealed a host of deeper environmental problems.

### **The Failure Points of Direct Execution**

Invoking winget.exe directly from the SYSTEM context introduced a cascade of reliability issues, demonstrating that the problem was far more complex than a missing PATH entry.

* **Path Instability:** While dynamic resolution worked, it depended on the script's logic being robust enough to handle multiple versions of the App Installer package being present on a system and correctly selecting the latest one.5  
* **Missing Dependencies:** A frequent and critical failure point was the absence of required runtime libraries. When run as SYSTEM, winget.exe would often fail with errors indicating that VCRUNTIME140.dll and MSVCP140.dll could not be found.1 These files are part of the Microsoft Visual C++ Redistributable, which is typically available to a standard user session but is not part of the sparse, minimal environment of the SYSTEM account. This required administrators to create complex dependencies in their deployment scripts, first ensuring the VC++ runtimes were installed before attempting to run WinGet.7  
* **Inconsistent Behavior and Silent Failures:** The most frustrating issue for administrators was the inconsistency of this method. Scripts that worked perfectly on established machines would often fail silently—producing no output and no error code—on freshly installed Windows systems, even after all Windows and Store updates were applied.15 This pointed to subtle, undocumented environmental differences between machines that made debugging nearly impossible.  
* **PowerShell Deadlock Issues:** A more esoteric problem arose from the way winget.exe renders its output. To display progress bars and other dynamic information, it frequently rewrites text to the same line in the console. When standard PowerShell methods were used to capture this output for logging, this non-sequential output could cause the parent PowerShell process to enter a deadlock state, where the script would hang indefinitely. This resulted in a "sometimes it works, sometimes it doesn't" scenario that was extremely difficult to troubleshoot.9

The extensive documentation of these failures across numerous GitHub issues served as a powerful, real-world feedback mechanism for Microsoft.1 The community's persistent efforts and detailed bug reports provided a clear business case and a set of technical requirements that directly justified and guided the development of an official, architecturally sound solution.

## **Microsoft's Strategic Shift: The Microsoft.WinGet.Client PowerShell Module**

After years of community workarounds and persistent enterprise demand, Microsoft introduced an official, supported solution for running WinGet in non-interactive, privileged contexts. This solution, the Microsoft.WinGet.Client PowerShell module, represents a fundamental architectural shift. Instead of attempting to fix the problems associated with running the user-facing winget.exe in an unsupported environment, Microsoft decoupled WinGet's core functionality from its command-line interface, providing a stable, programmatic API for automation.

### **The Official Solution: A PowerShell Module Built on COM APIs**

The Microsoft.WinGet.Client module is the sanctioned method for interacting with the Windows Package Manager service from the SYSTEM context.1 Its key innovation is that it does not "shell out" to winget.exe. Instead, it communicates directly with the underlying package manager service using "In Process" Component Object Model (COM) APIs.1

This architectural decision elegantly sidesteps every major issue that plagued the direct execution workarounds:

* **No Path Resolution Needed:** Because the module communicates with the registered COM service, it doesn't need to know the location of winget.exe. This eliminates the fragility of dynamic path-finding scripts.  
* **No App Execution Alias Dependency:** The COM interface is independent of the user-centric alias mechanism.  
* **No Missing Runtimes:** The module interacts with the service in-process, resolving dependencies correctly without requiring manual modifications to the SYSTEM account's environment.  
* **No Console Output Deadlocks:** The module returns structured PowerShell objects instead of parsing console text, completely avoiding the deadlock issues associated with capturing winget.exe's output.

By exposing the core service via a programmatic interface, Microsoft has effectively made winget.exe just one of many possible clients. The PowerShell module is another, and this design allows for the creation of future clients that can interact with the package manager in a stable and supported manner, regardless of the execution context.

### **Prerequisites and Implementation Details**

While the PowerShell module provides a robust solution, its use in an enterprise context comes with a specific set of prerequisites that administrators must manage.

* **PowerShell 7 Requirement:** The module is designed for and requires PowerShell 7 for SYSTEM context execution. It is not fully compatible with the legacy Windows PowerShell 5.1 that is included with Windows by default.18 This introduces a new dependency management challenge for organizations, as they must first establish a reliable method for deploying and maintaining PowerShell 7 across their endpoints before they can leverage the official WinGet solution.19  
* **Multi-Threaded Apartment (-MTA) Switch:** The COM interactions required by the module necessitate that the PowerShell 7 host process be launched in a Multi-Threaded Apartment state. This is accomplished by starting the executable with the \-MTA command-line switch (e.g., pwsh.exe \-MTA). The default Single-Threaded Apartment (-STA) state will cause the cmdlets to fail or hang when run as SYSTEM.1  
* **Module Installation:** The module itself is installed from the official PowerShell Gallery using a standard command, which can be incorporated into deployment scripts 3:  
  PowerShell  
  Install-Module \-Name Microsoft.WinGet.Client \-Force \-Scope AllUsers

### **Known Limitations and Ongoing Development**

The Microsoft.WinGet.Client module is a significant advancement, but it is still an evolving solution. Administrators should be aware of several limitations:

* **Incomplete COM Migration:** As of its initial releases, not all WinGet functionality had been migrated to the COM backend. Some cmdlets, such as Get-WinGetVersion, still relied on calling winget.exe and would therefore fail in the SYSTEM context.17 This requires administrators to test their scripts carefully and be aware that certain operations may not yet be supported.  
* **Installer and Manifest Quality:** The module provides a stable way to *initiate* an installation, but the ultimate success of that installation still depends heavily on the quality of the target application's installer and the accuracy of its WinGet manifest. Well-constructed Microsoft Installer (MSI) packages with properly defined properties are the most reliable. Traditional executable installers (.exe) and Nullsoft Scriptable Install System (NSIS) packages can be unpredictable, especially with silent installation switches, leading to varied results.1 Similarly, APPX and MSIX packages are often user-targeted and may not be suitable for SYSTEM context deployment.1

## **A Case Study in Automation: Deconstructing the Winget-AutoUpdate (WAU) Implementation**

While Microsoft was developing its official PowerShell module, the community project Winget-AutoUpdate (WAU), created by GitHub user Romanitho, had already become a de facto standard for automated WinGet patching in the enterprise. The project's success lies not in a secret method for running winget.exe, but in a clever and robust orchestration layer built on top of native Windows components. A deep dive into its implementation reveals a sophisticated approach to solving the context problem.

### **The Core Mechanism: Orchestration via the Windows Task Scheduler**

The "magic" behind WAU's ability to reliably run as SYSTEM is its use of the Windows Task Scheduler.21 Rather than being an ad-hoc script pushed from a management tool, WAU is a self-contained solution that, upon installation, configures the local system to perform updates on a schedule.

The process, orchestrated by the Winget-AutoUpdate-Install.ps1 script, follows a clear and effective pattern 22:

1. **Installation:** The installer script (Winget-AutoUpdate-Install.ps1) is executed with administrative privileges. It copies the necessary PowerShell scripts and configuration files to a stable location on the system, typically C:\\ProgramData\\Winget-AutoUpdate.  
2. **Task Creation:** The script then programmatically creates a new scheduled task, commonly named Winget-AutoUpdate.  
3. **Principal Configuration:** This is the most critical step. The script defines the task's principal—the account under which the task will run—as NT AUTHORITY\\SYSTEM. This is achieved by specifying the account's well-known Security Identifier (SID), S-1-5-18, and setting the run level to "Highest" to ensure it executes with full privileges.22  
4. **Action Definition:** The action for the scheduled task is configured to launch powershell.exe with arguments to bypass the execution policy and run WAU's main update script.

By leveraging the Task Scheduler, WAU ensures that its update logic is always executed in a consistent, predictable, and highly privileged non-interactive environment, abstracting away the complexities and inconsistencies of being launched directly from an external management tool.

### **The Dual-Context Strategy: A Comprehensive Solution**

Perhaps the most sophisticated feature of WAU is its recognition that a SYSTEM-only approach is incomplete for managing a modern Windows endpoint. Many applications, such as Google Chrome or Visual Studio Code, can be installed either for all users (machine scope) or just for the current user (user scope). A process running as SYSTEM is effectively blind to applications installed within a specific user's profile (e.g., in %LOCALAPPDATA%).11

To solve this, WAU implements a dual-context strategy by creating two distinct scheduled tasks 23:

1. **The SYSTEM Context Task:** The primary task, as described above, runs as NT AUTHORITY\\SYSTEM. It is responsible for scanning for and upgrading applications installed with machine-wide scope.  
2. **The User Context Task:** A second scheduled task, often named Winget-AutoUpdate-UserContext, is configured to run under the context of the currently logged-on user. This task is typically triggered by a user logon event.22 It performs a separate scan and update cycle, but because it runs as the user, it can correctly identify and manage applications installed exclusively within that user's profile.

This dual-context model allows WAU to provide comprehensive update coverage for both machine- and user-scoped applications, a critical capability for managing the diverse software landscape on a typical enterprise device. This is a feature that a simple implementation of the official Microsoft.WinGet.Client module does not provide out of the box; an administrator would need to build this complex orchestration logic themselves.

### **Underlying Execution and Robustness**

It is important to note that under the hood, WAU's PowerShell scripts still rely on the same dynamic path resolution techniques pioneered by the community to find and execute winget.exe.25 It does not use a private API or a different method of invocation.

However, WAU's significant value proposition is the highly robust framework it wraps around this brittle core function. The project includes a wealth of features that transform the unreliable workaround into a dependable automation tool:

* **Extensive Logging:** Separate log files are created for privileged (SYSTEM) and user-context executions, simplifying troubleshooting.24  
* **Rich Configuration:** Administrators can control behavior through simple text files (excluded\_apps.txt, included\_apps.txt) or MSI installation parameters, allowing for easy blacklisting or whitelisting of applications.23  
* **Extensibility:** A "mods" folder allows for the execution of custom pre- and post-install scripts for specific applications, enabling complex deployment workflows.21  
* **Error Handling:** The scripts contain logic to handle common issues, such as applications with "Unknown" versions that could cause repeated failed update attempts.24

In essence, WAU's innovation is not in *how* it calls WinGet, but in the sophisticated and reliable *orchestration layer* it builds around that call.

## **Synthesis and Strategic Recommendations for Enterprise Deployment**

The journey of running WinGet in the SYSTEM context has evolved from fragile, community-hacked workarounds to an officially supported, architecturally sound PowerShell module. Concurrently, third-party tools like Winget-AutoUpdate have matured into comprehensive management solutions. For the enterprise administrator, choosing the right approach depends on the specific use case, balancing factors of official support, implementation complexity, and the required scope of management.

### **Comparative Analysis of Execution Methods**

The three primary methods for executing WinGet operations as SYSTEM each present a distinct set of advantages and disadvantages. The following table provides a comparative analysis to guide decision-making. This summary distills the report's technical findings into a practical tool, allowing administrators to quickly assess the trade-offs and select the method best suited to their organizational requirements and technical constraints. For instance, an organization with a strict policy requiring officially supported tools would immediately gravitate toward the PowerShell module, whereas an organization prioritizing a complete, automated patching solution for both user and machine applications would find the capabilities of WAU more compelling.

**Table 1: Comparison of Methods for Executing WinGet in SYSTEM Context**

| Feature | Direct winget.exe Invocation | Microsoft.WinGet.Client Module | Winget-AutoUpdate (WAU) |
| :---- | :---- | :---- | :---- |
| **Underlying Mechanism** | Direct process call via dynamically resolved path | In-process COM API calls to the Windows Package Manager service | Orchestration via Windows Task Scheduler, which then calls winget.exe |
| **Official Support** | Unsupported; considered a brittle workaround | Officially supported by Microsoft | Third-party community project; no official Microsoft support |
| **Reliability & Stability** | Low; prone to silent failures, dependency issues, and deadlocks | High; bypasses the instabilities of the command-line executable | High; provides a robust framework with extensive logging and error handling |
| **Key Prerequisites** | Robust path-finding script; manual installation of dependencies (e.g., VC++ Runtimes) | PowerShell 7; must be launched with the \-MTA switch | WAU must be installed on the endpoint |
| **Handles User-Scope Apps** | No; blind to applications installed in user profiles | Limited; runs only as SYSTEM and cannot see user-scope applications | Yes; employs a dual-context strategy with separate tasks for SYSTEM and the logged-on user |
| **Ease of Implementation** | Complex; requires significant custom scripting and error handling | Moderate; requires managing the PowerShell 7 dependency and understanding cmdlet syntax | Simple; provides a "set-and-forget" solution after initial deployment |

### **Strategic Recommendations**

Based on this analysis, the following strategic recommendations can be made for enterprise environments:

* **For Targeted, Ad-Hoc SYSTEM Tasks:** The **Microsoft.WinGet.Client PowerShell module** is the recommended approach. For tasks such as including a single application installation in a larger Intune deployment script or performing a specific, one-time action, the official module provides the highest degree of stability and forward compatibility. It is the architecturally correct method for programmatic interaction with the Windows Package Manager. The primary consideration for its adoption is the organization's ability to deploy and manage the PowerShell 7 prerequisite.  
* **For Comprehensive, Automated Patch Management:** A dedicated wrapper solution like **Winget-AutoUpdate** is the superior choice. For the ongoing, scheduled task of keeping all third-party applications on endpoints up-to-date, WAU's feature set is far more complete. Its robust orchestration via the Task Scheduler, mature configuration options (blacklisting, notifications, logging), and, most importantly, its critical **dual-context support** for managing both machine- and user-scoped applications, make it a more holistic and practical "set-and-forget" solution for fleet-wide application lifecycle management.  
* **Deprecation of Direct Execution:** The method of dynamically resolving the path to winget.exe and invoking it directly from a script should now be considered **deprecated and unsuitable for production enterprise environments**. While it was a necessary and innovative workaround in the absence of an official solution, its inherent fragility and the availability of superior, more reliable alternatives render it obsolete.

### **The Future: A Converging Ecosystem**

The current landscape represents a transitional phase in WinGet's maturation as an enterprise tool. Microsoft has provided the stable, powerful *engine* with the Microsoft.WinGet.Client module and its underlying COM API. The community, through projects like Winget-AutoUpdate, has built a complete and feature-rich *vehicle* around the older, less reliable engine.

The logical next step in this evolution is the convergence of these two streams. It is highly probable that future versions of tools like Winget-AutoUpdate will re-architect their backend to use the official Microsoft.WinGet.Client module instead of directly calling winget.exe. This would create the ideal solution: the stability and official support of the COM API combined with the sophisticated orchestration, dual-context logic, and user-friendly configuration features of the third-party wrapper. When this convergence occurs, WinGet will have fully realized its potential as a true enterprise-grade package management framework for the Windows ecosystem.

#### **Works cited**

1. System Context · Issue \#3049 · microsoft/winget-cli \- GitHub, accessed October 28, 2025, [https://github.com/microsoft/winget-cli/issues/3049](https://github.com/microsoft/winget-cli/issues/3049)  
2. Winget for dummies... : r/sysadmin \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/sysadmin/comments/1cwqjlb/winget\_for\_dummies/](https://www.reddit.com/r/sysadmin/comments/1cwqjlb/winget_for_dummies/)  
3. Use WinGet to install and manage applications | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows/package-manager/winget/](https://learn.microsoft.com/en-us/windows/package-manager/winget/)  
4. Debugging and troubleshooting issues with the WinGet tool \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows/package-manager/winget/troubleshooting](https://learn.microsoft.com/en-us/windows/package-manager/winget/troubleshooting)  
5. Usage with System Account · microsoft winget-cli · Discussion \#962 ..., accessed October 28, 2025, [https://github.com/microsoft/winget-cli/discussions/962](https://github.com/microsoft/winget-cli/discussions/962)  
6. Post-install winget not working | NTLite Forums, accessed October 28, 2025, [https://www.ntlite.com/community/index.php?threads/post-install-winget-not-working.3539/](https://www.ntlite.com/community/index.php?threads/post-install-winget-not-working.3539/)  
7. Install Software With WinGet As System (UPDATE 2024\) \- David Just, accessed October 28, 2025, [https://davidjust.com/post/intune-install-software-with-winget-updated-2024/](https://davidjust.com/post/intune-install-software-with-winget-updated-2024/)  
8. Winget Command \- LIVEcommunity \- 594907 \- Palo Alto Networks, accessed October 28, 2025, [https://live.paloaltonetworks.com/t5/cortex-xdr-discussions/winget-command/td-p/594907](https://live.paloaltonetworks.com/t5/cortex-xdr-discussions/winget-command/td-p/594907)  
9. winget for Intune (and SCCM) \- Asher Jebbink, accessed October 28, 2025, [https://asherjebbink.medium.com/winget-for-intune-and-sccm-361b88ca0eb](https://asherjebbink.medium.com/winget-for-intune-and-sccm-361b88ca0eb)  
10. Running Windows Package Manager (Winget) in the SYSTEM context \- njen.com, accessed October 28, 2025, [https://nialljen.wordpress.com/2023/05/14/running-windows-package-manager-winget-in-the-system-context/](https://nialljen.wordpress.com/2023/05/14/running-windows-package-manager-winget-in-the-system-context/)  
11. Add support for SYSTEM account when using Winget PowerShell ..., accessed October 28, 2025, [https://github.com/microsoft/winget-cli/issues/4422](https://github.com/microsoft/winget-cli/issues/4422)  
12. Is it possible to run Winget in the USER context without window pop-up? \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/PowerShell/comments/1c3wezi/is\_it\_possible\_to\_run\_winget\_in\_the\_user\_context/](https://www.reddit.com/r/PowerShell/comments/1c3wezi/is_it_possible_to_run_winget_in_the_user_context/)  
13. winget is not recognized as the name of cmdlet \- Microsoft Q\&A, accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/1305372/winget-is-not-recognized-as-the-name-of-cmdlet](https://learn.microsoft.com/en-us/answers/questions/1305372/winget-is-not-recognized-as-the-name-of-cmdlet)  
14. Winget Deployment with PSAppDeployToolkit and Intune \- scloud, accessed October 28, 2025, [https://scloud.work/winget-deployment-with-psappdeploytoolkit-and-intune/](https://scloud.work/winget-deployment-with-psappdeploytoolkit-and-intune/)  
15. winget does not respond on fresh Windows 10 installations in system context, works on existing installations : r/sysadmin \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/sysadmin/comments/1bpp9me/winget\_does\_not\_respond\_on\_fresh\_windows\_10/](https://www.reddit.com/r/sysadmin/comments/1bpp9me/winget_does_not_respond_on_fresh_windows_10/)  
16. winget does not respond on fresh Windows 10 installations in system context, works on existing installations \#4321 \- GitHub, accessed October 28, 2025, [https://github.com/microsoft/winget-cli/issues/4321](https://github.com/microsoft/winget-cli/issues/4321)  
17. Winget Powershell Cmdlets don't work from SYSTEM context (\#3409) \- GitHub, accessed October 28, 2025, [https://github.com/microsoft/winget-cli/issues/3409](https://github.com/microsoft/winget-cli/issues/3409)  
18. Script to update applications through WinGet is not running through GPO \- Server Fault, accessed October 28, 2025, [https://serverfault.com/questions/1174342/script-to-update-applications-through-winget-is-not-running-through-gpo](https://serverfault.com/questions/1174342/script-to-update-applications-through-winget-is-not-running-through-gpo)  
19. Deploy and automatically update WinGet apps in Intune using PowerShell without Remediation or 3rd party tools, accessed October 28, 2025, [https://powershellisfun.com/2025/05/16/deploy-and-automatically-update-winget-apps-in-intune-using-powershell-without-remediation-or-3rd-party-tools/](https://powershellisfun.com/2025/05/16/deploy-and-automatically-update-winget-apps-in-intune-using-powershell-without-remediation-or-3rd-party-tools/)  
20. Winget PowerShell module \- Andrew Taylor, accessed October 28, 2025, [https://andrewstaylor.com/2023/11/28/winget-powershell-module/](https://andrewstaylor.com/2023/11/28/winget-powershell-module/)  
21. Winget-AutoUpdate download | SourceForge.net, accessed October 28, 2025, [https://sourceforge.net/projects/winget-autoupdate.mirror/](https://sourceforge.net/projects/winget-autoupdate.mirror/)  
22. Winget-AutoUpdate-Install.ps1 · main · klient / WAU · GitLab, accessed October 28, 2025, [https://code.miun.se/klient/wau/-/blob/main/Winget-AutoUpdate-Install.ps1?ref\_type=heads](https://code.miun.se/klient/wau/-/blob/main/Winget-AutoUpdate-Install.ps1?ref_type=heads)  
23. CanyonITS/PS-Winget-AutoUpdate: WAU daily updates ... \- GitHub, accessed October 28, 2025, [https://github.com/CanyonITS/PS-Winget-AutoUpdate](https://github.com/CanyonITS/PS-Winget-AutoUpdate)  
24. Romanitho/Winget-AutoUpdate: WAU daily updates apps as system and notify connected users. (Allowlist and Blocklist support) \- GitHub, accessed October 28, 2025, [https://github.com/Romanitho/Winget-AutoUpdate](https://github.com/Romanitho/Winget-AutoUpdate)  
25. Winget Automation : r/PowerShell \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/PowerShell/comments/1b0w9ay/winget\_automation/](https://www.reddit.com/r/PowerShell/comments/1b0w9ay/winget_automation/)