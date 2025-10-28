
# **Automating SoftEther VPN Client Deployment: A Technical Guide to Unattended Installation**

## **I. Introduction: The Challenge of Unattended SoftEther Client Deployment**

SoftEther VPN stands as a uniquely powerful and versatile multi-protocol VPN software solution. Developed as part of a master's thesis at the University of Tsukuba, it is offered as free and open-source software for both personal and commercial use.1 Its key strength lies in its comprehensive protocol support, integrating SSL-VPN, OpenVPN, L2TP/IPsec, MS-SSTP, and others into a single server program, making it a powerful alternative to established commercial and open-source VPNs.3 The software runs on a wide array of operating systems, including Windows, Linux, macOS, FreeBSD, and Solaris, further cementing its flexibility.1

Despite its technical prowess and enterprise-grade feature set, the SoftEther VPN Client for Windows presents a significant operational challenge in managed IT environments: the official installer provides no command-line switches or parameters for silent, unattended installation. This omission creates a deployment gap for system administrators who need to deploy the software at scale across numerous workstations. The installer is designed exclusively for interactive, manual setup, a method that is inefficient and impractical for enterprise rollouts.

This lack of silent installation capabilities is not an oversight but rather a reflection of the project's development history and priorities. Originating in an academic setting, the focus was primarily on technological innovation, protocol implementation, and creating a powerful, feature-rich graphical user interface (GUI) for configuration and management.1 The design philosophy catered to an individual user or administrator interacting directly with the software, rather than the automated deployment systems common in corporate environments. This disconnect between the software's powerful backend capabilities and its consumer-grade installation method necessitates an external automation solution.

This report provides a definitive solution to this challenge by leveraging AutoIt, a freeware scripting language designed specifically for automating the Windows GUI.5 AutoIt excels at simulating user input, such as keystrokes and mouse clicks, and can directly manipulate windows and controls, making it the ideal tool to script the interactive SoftEther installation process.5 The objective of this document is to deliver a complete, robust, and heavily annotated AutoIt script that performs a truly unattended installation of the SoftEther VPN Client. This is accompanied by a comprehensive strategic guide covering script implementation, deployment, and long-term maintenance considerations for IT professionals.

## **II. Deconstructing the Manual Installation Process: A Blueprint for Automation**

To create a reliable automation script, one must first establish a precise and repeatable blueprint of the manual process. A thorough analysis of the SoftEther VPN Client installation wizard, as documented in multiple independent guides, reveals a highly consistent and predictable workflow.6 This consistency across versions is the bedrock upon which a robust automation script can be built. The standard installation sequence invariably proceeds through the following stages: Welcome Screen, Component Selection, License Agreement, Important Notice, Installation Directory selection, a progress bar, and finally, a completion screen.

However, a truly "unattended" installation implies more than just copying files to a directory; it requires leaving the software in a functional, ready-to-configure state. A critical observation from the installation process is that upon the first launch of the SoftEther VPN Client Manager, the application immediately prompts the user with a mandatory dialog: "Do you want to create a Virtual Network Adapter?".6 This step is not optional; the client cannot establish any connections without this virtual adapter. Therefore, an effective automation script must extend its scope beyond the installer itself. It must be a two-phase process: first, automate the installer wizard to completion, and second, launch the client manager application and automate the creation of the default virtual network adapter. Failing to address this second phase would result in a non-functional client, defeating the purpose of unattended deployment.

The following table, the "SoftEther Client Installation Automation Map," serves as the core technical specification for the script. It meticulously maps each step of the manual process—including the crucial post-installation phase—to the specific GUI elements and the corresponding AutoIt functions required to automate the interaction. This map is both a design document for the script's creation and an essential maintenance guide for adapting the script to future versions of the SoftEther client.

| Step | Installer/Application Window Title | User Action | Target Control Identifier (Text or ClassNN) | Primary AutoIt Function |
| :---- | :---- | :---- | :---- | :---- |
| 1 | SoftEther VPN Client Setup Wizard | Click "Next" | Button1 ("\&Next \>") | ControlClick |
| 2 | SoftEther VPN Client Setup Wizard | Select Radio Button | Button3 ("SoftEther VPN Client") | ControlClick |
| 3 | SoftEther VPN Client Setup Wizard | Click "Next" | Button1 ("\&Next \>") | ControlClick |
| 4 | License Agreement | Check "I agree..." | Button1 ("I agree to the End User License Agreement") | ControlClick |
| 5 | License Agreement | Click "Next" | Button2 ("\&Next \>") | ControlClick |
| 6 | Important Notice | Click "Next" | Button1 ("\&Next \>") | ControlClick |
| 7 | Select Install Location | Click "Next" | Button1 ("\&Next \>") | ControlClick |
| 8 | Installation Complete | Click "Finish" | Button1 ("\&Finish") | ControlClick |
| 9 | SoftEther VPN Client Manager | (Wait for prompt) | N/A | WinWaitActive |
| 10 | SoftEther VPN Client Manager | Click "Yes" to create adapter | Button1 ("\&Yes") | ControlClick |
| 11 | Create New Virtual Network Adapter | Click "OK" to confirm name | Button1 ("OK") | ControlClick |

## **III. The AutoIt Solution: A Complete, Annotated Script for Silent Installation**

This section presents the complete, production-ready AutoIt script designed for the silent and unattended installation of the SoftEther VPN Client. The script is heavily annotated to ensure clarity and facilitate future maintenance. Following the script is a detailed functional breakdown that explains the logic and purpose of each code segment.

### **The Complete AutoIt Script: Install-SoftEther.au3**

AutoIt

; \=================================================================================================  
; Script:       Install-SoftEther.au3  
; Author:       Technical Solutions Group  
; Date:         October 26, 2025  
; Version:      1.0  
; Description:  Performs a silent, unattended installation of the SoftEther VPN Client.  
;               This script automates the installer wizard and the initial creation of the  
;               Virtual Network Adapter, leaving the client in a ready-to-configure state.  
;  
; Usage:        1\. Place this script in the same directory as the SoftEther VPN Client installer.  
;               2\. Update the $InstallerExe variable with the correct installer filename.  
;               3\. Compile the script to an.exe using the AutoIt Aut2exe utility.  
;               4\. Deploy the compiled.exe and the installer file together.  
; \=================================================================================================

\#**NoTrayIcon**

; \=================================================================================================  
; SCRIPT CONFIGURATION  
; \=================================================================================================

; \--- Set the filename of the SoftEther VPN Client installer executable.  
; \--- The installer MUST be in the same directory as this script.  
Global $InstallerExe \= "softether-vpnclient-v4.44-9807-rtm-2025.04.16-windows-x86\_x64-intel.exe"

; \--- Set the default installation path for the VPN Client Manager executable.  
Global $VpnManagerPath \= @ProgramFilesDir & "\\SoftEther VPN Client\\vpncmgr\_x64.exe"  
If Not @OSArch \= "X64" Then  
    $VpnManagerPath \= @ProgramFilesDir & "\\SoftEther VPN Client\\vpncmgr.exe"  
EndIf

; \=================================================================================================  
; INITIALIZATION  
; \=================================================================================================

; \--- Set window title matching to partial match mode (substring).  
; \--- This adds robustness in case of minor changes to window titles in future versions.  
Opt("WinTitleMatchMode", 2)

; \--- Check if the installer file exists before starting.  
If Not FileExists($InstallerExe) Then  
    MsgBox(16, "Error", "SoftEther installer not found: " & $InstallerExe & @CRLF & "Please place the installer in the same directory as the script.")  
    Exit 1  
EndIf

; \=================================================================================================  
; PHASE 1: AUTOMATE THE INSTALLER WIZARD  
; \=================================================================================================

; \--- Run the installer and wait for it to close before continuing.  
; \--- Using RunWait ensures the script pauses until the installation is fully complete.  
RunWait($InstallerExe)

; \--- Step 1: Welcome Screen  
WinWaitActive("SoftEther VPN Client Setup Wizard")  
ControlClick("SoftEther VPN Client Setup Wizard", "\&Next \>", "Button1")

; \--- Step 2: Select Software Components  
WinWaitActive("SoftEther VPN Client Setup Wizard")  
; Select the "SoftEther VPN Client" radio button  
ControlClick("SoftEther VPN Client Setup Wizard", "SoftEther VPN Client", "Button3")  
ControlClick("SoftEther VPN Client Setup Wizard", "\&Next \>", "Button1")

; \--- Step 3: License Agreement  
WinWaitActive("License Agreement")  
ControlClick("License Agreement", "I agree to the End User License Agreement", "Button1")  
ControlClick("License Agreement", "\&Next \>", "Button2")

; \--- Step 4: Important Notice  
WinWaitActive("Important Notice")  
ControlClick("Important Notice", "\&Next \>", "Button1")

; \--- Step 5: Select Install Location  
WinWaitActive("Select Install Location")  
ControlClick("Select Install Location", "\&Next \>", "Button1")

; \--- The installer will now copy files. We wait for the "Installation Complete" window.  
WinWaitActive("Installation Complete", "completed", 30) ; Wait up to 30 seconds

; \--- Step 6: Finish Installation  
ControlClick("Installation Complete", "\&Finish", "Button1")

; \=================================================================================================  
; PHASE 2: AUTOMATE VIRTUAL NETWORK ADAPTER CREATION  
; \=================================================================================================

; \--- Wait a moment for the system to settle before launching the client manager.  
Sleep(2000)

; \--- Launch the VPN Client Manager to trigger the initial setup prompt.  
Run($VpnManagerPath)

; \--- Step 7: Handle the "Create Virtual Network Adapter" prompt.  
; \--- This window title is the main application window, but the prompt appears within it.  
WinWaitActive("SoftEther VPN Client Manager", "Do you want to create a Virtual Network Adapter?", 30)  
ControlClick("SoftEther VPN Client Manager", "\&Yes", "Button1")

; \--- Step 8: Confirm the name of the new adapter.  
WinWaitActive("Create New Virtual Network Adapter")  
ControlClick("Create New Virtual Network Adapter", "OK", "Button1")

; \--- Wait for the main manager window to become active again after the dialogs close.  
WinWaitActive("SoftEther VPN Client Manager")

; \--- Close the VPN Client Manager application.  
WinClose("SoftEther VPN Client Manager")

; \=================================================================================================  
; SCRIPT COMPLETION  
; \=================================================================================================

Exit 0

### **Script Anatomy and Functional Breakdown**

This script is logically divided into two primary phases, mirroring the manual process identified previously. Its reliability hinges on a few key AutoIt functions and principles.

* **Initialization and Prerequisites:** The script begins by setting global variables for the installer executable's filename and the path to the VPN Client Manager. It intelligently checks the operating system architecture (@OSArch) to select the correct manager executable (vpncmgr\_x64.exe or vpncmgr.exe). Crucially, it sets Opt("WinTitleMatchMode", 2), which allows AutoIt to find windows based on a partial title match. This adds a layer of resilience, as the script will not fail if a future SoftEther update adds a version number to a window title.  
* **Launching the Installer (RunWait):** The automation of the installer is initiated with the RunWait function.11 This function is deliberately chosen over the simpler Run command because it forces the script to pause its own execution until the installer process ($InstallerExe) has completely terminated. This creates a synchronous workflow, ensuring that Phase 2 of the script (Virtual Adapter Creation) does not begin until Phase 1 (File Installation) is verifiably complete.  
* **Synchronizing with the GUI (WinWaitActive):** The cornerstone of the script's reliability is the WinWaitActive function.11 GUI automation scripts can easily fail if they try to interact with a window or control that has not yet been drawn on the screen. WinWaitActive acts as a synchronization point, pausing the script indefinitely (or until a specified timeout) until a window with a matching title is present and is the active, focused window. Each step of the script waits for the expected window before attempting to interact with it, creating a robust, event-driven sequence.  
* **Interacting with Controls (ControlClick):** All user interactions, such as clicking "Next" or selecting a radio button, are performed by the ControlClick function.11 This function sends a click command directly to a control without needing to simulate mouse movement, making it faster and more reliable than mouse-based automation. The parameters used for title, text, and controlID are taken directly from the "Automation Map" table, which was populated using tools like the AutoIt Window Info tool.  
* **Handling the Post-Installation Steps:** After RunWait confirms the installer has finished, the script proceeds to Phase 2\. It launches the VPN Client Manager (vpncmgr\_x64.exe) and uses the same WinWaitActive and ControlClick pattern to navigate the "Create Virtual Network Adapter" dialogs. Once these actions are complete, it closes the manager application, leaving the system with a fully installed and initialized SoftEther VPN Client, ready for configuration.

## **IV. Implementation and Deployment Guide**

This section provides actionable instructions for system administrators to compile, deploy, and verify the automated installation of the SoftEther VPN Client using the provided AutoIt script.

### **Prerequisites**

Before deployment, two components are required: the SoftEther VPN Client installer and the AutoIt scripting environment.

1. **Acquiring the SoftEther Installer:** Navigate to the official SoftEther Download Center.12 In the selection menus, choose:  
   * **Select Software:** SoftEther VPN (Freeware)  
   * **Select Component:** SoftEther VPN Client  
   * **Select Platform:** Windows  
   * From the resulting list, download the latest stable version. Stable releases are typically marked with "RTM" (Release to Manufacturing).  
2. **Setting up the AutoIt Environment:** Download the full AutoIt installation package from the official website.5 The full package is recommended as it includes not only the script interpreter but also the SciTE Script Editor and, most importantly, the Aut2exe script compiler.

### **Compiling the Script for Portability**

For enterprise deployment, it is highly recommended to compile the .au3 script into a standalone .exe executable. This provides several key advantages:

* **No Dependencies:** The compiled .exe contains the entire AutoIt interpreter and can run on any target Windows machine without requiring AutoIt to be pre-installed.  
* **Simplified Distribution:** A single executable is easier to manage and deploy than a script file that requires an interpreter.  
* **Code Protection:** It prevents end-users or other administrators from easily viewing or modifying the automation logic.

To compile the script:

1. Save the script code as Install-SoftEther.au3.  
2. Place the downloaded SoftEther installer .exe in the same directory.  
3. Ensure the $InstallerExe variable in the script matches the filename of the installer exactly.  
4. Launch the Aut2exe utility (Compile Script to.exe) from the AutoIt Start Menu group.  
5. In the Aut2exe interface, browse to select Install-SoftEther.au3 as the source file. The destination file will be populated automatically.  
6. Click the "Compile Script" button. A new file, Install-SoftEther.exe, will be created in the same directory.

### **Deployment Strategies**

The compiled Install-SoftEther.exe and the original SoftEther installer .exe must be deployed together to the target machines (e.g., in the same temporary directory). The automation can then be triggered by executing the compiled script. This package can be deployed using any standard enterprise software distribution method, including:

* **Group Policy (GPO):** Use a computer startup script to execute Install-SoftEther.exe.  
* **Microsoft Endpoint Configuration Manager (MECM/SCCM):** Create an Application or Package that copies both files to the client and runs the command Install-SoftEther.exe.  
* **Third-Party Deployment Tools (e.g., PDQ Deploy, Ivanti):** Create a deployment package that includes both files and specifies Install-SoftEther.exe as the installation command.

The script is designed to run silently (due to the \#NoTrayIcon directive) and will exit with a code of 0 on success or 1 if the installer file is not found.

### **Verification of Successful Installation**

Administrators can verify a successful deployment through several methods:

* **File System:** Check for the existence of the installation directory at C:\\Program Files\\SoftEther VPN Client.  
* **Windows Services:** Verify that the "SoftEther VPN Client" service has been installed and is set to start automatically.  
* **Network Adapters:** In Device Manager or the Network Connections control panel, confirm the presence of a new adapter named "VPN Client Adapter \- VPN". The presence of this adapter is the definitive sign that both phases of the automation script completed successfully.

## **V. Advanced Considerations and Future Automation Strategy**

While the provided AutoIt script effectively solves the problem of unattended installation, a truly robust enterprise deployment strategy requires looking beyond the initial setup. This involves considerations for error handling, long-term maintenance, and understanding the proper division between installation and configuration automation.

### **Error Handling and Script Robustness**

GUI automation is inherently more fragile than command-line or API-based automation because it depends on a visual interface that can change between software versions. To enhance the script's robustness, administrators can add timeouts to the WinWaitActive functions. For example, WinWaitActive("Window Title", "", 10\) will cause the script to wait a maximum of 10 seconds for the window to appear. If it does not, the function will time out and set an error flag. This allows for the creation of logic that can gracefully exit and return a specific error code to the deployment system, preventing the script from hanging indefinitely if an unexpected window or error appears.

### **The Critical Distinction: Installation vs. Configuration**

The most important strategic consideration is understanding the difference between *installation* and *configuration* and using the right tool for each task.

* **Installation** is the process of getting the software's files onto the system and performing initial, one-time setup tasks. This process, as demonstrated, is linear and identical for every machine, making it a suitable task for GUI automation with AutoIt.  
* **Configuration** is the process of creating VPN connection profiles, which involves setting variable data such as server hostnames, virtual hub names, usernames, and authentication methods. This data is often unique per user or department.

Using AutoIt to automate the entry of this variable data into GUI text fields is possible but highly inadvisable. It is brittle, difficult to maintain, and insecure if credentials are hardcoded into the script. Fortunately, the SoftEther project provides superior, purpose-built tools for this exact task. The software suite includes a powerful command-line administration utility, vpncmd.exe, and a comprehensive JSON-RPC API for programmatic management.3 These tools are the developer-intended methods for scripting configuration and management.

Therefore, the optimal, most scalable, and maintainable deployment methodology is a two-stage process:

1. **Stage 1 (Installation):** Use the provided Install-SoftEther.exe (compiled AutoIt script) to handle the GUI-based installation and the creation of the virtual network adapter. This stage is universal for all deployments.  
2. **Stage 2 (Configuration):** After the installation is complete, use a separate script (e.g., PowerShell or a simple batch file) to call vpncmd.exe to programmatically create and define the required VPN connection settings.

This hybrid approach leverages the strengths of each tool: AutoIt for the rigid, unchangeable installer GUI, and vpncmd for the flexible, data-driven configuration.

### **Post-Installation Configuration with vpncmd**

To illustrate the second stage, a conceptual batch script could be used to create a new VPN connection account. The vpncmd utility can be scripted by feeding it commands. For example, to create a connection named "Corporate VPN" connecting to "vpn.mycompany.com", the commands would look something like this:

Code snippet

@echo off  
SET VPNCMD="C:\\Program Files\\SoftEther VPN Client\\vpncmd\_x64.exe"  
SET HOST=localhost:9933

(  
echo AccountCreate "Corporate VPN"  
echo AccountServerNameSet "Corporate VPN" /SERVER:vpn.mycompany.com:443  
echo AccountHubNameSet "Corporate VPN" /HUB:PRODUCTION  
echo AccountUsernameSet "Corporate VPN" /USERNAME:MyUser  
echo AccountPasswordSet "Corporate VPN" /PASSWORD:MyPassword /TYPE:standard  
) | %VPNCMD% /CLIENT /CMD

This command-line approach is far more robust and adaptable for deploying varied connection settings than attempting to automate the same steps in the GUI.

### **Script Maintenance and Future-Proofing**

Finally, it is crucial to recognize that the AutoIt script's reliability is directly tied to the SoftEther VPN Client's GUI. Before deploying a new version of the SoftEther client across the enterprise, the automation script must be re-validated in a test environment. Administrators should run the script against the new installer to ensure that window titles, control identifiers, and the overall workflow have not changed in a way that would break the automation. This simple testing step is the key to maintaining a successful and reliable unattended deployment strategy over the long term.

#### **Works cited**

1. SoftEther VPN \- Wikipedia, accessed October 22, 2025, [https://en.wikipedia.org/wiki/SoftEther\_VPN](https://en.wikipedia.org/wiki/SoftEther_VPN)  
2. SoftEther VPN Project, accessed October 22, 2025, [https://www.softether.org/](https://www.softether.org/)  
3. SoftEtherVPN/SoftEtherVPN: Cross-platform multi-protocol VPN software. Pull requests are welcome. The stable version is available at https://github.com/SoftEtherVPN/SoftEtherVPN\_Stable. \- GitHub, accessed October 22, 2025, [https://github.com/SoftEtherVPN/SoftEtherVPN](https://github.com/SoftEtherVPN/SoftEtherVPN)  
4. SoftEtherVPN/SoftEtherVPN\_Stable: Cross-platform multi-protocol VPN software. This repository is officially managed by Daiyuu Nobori, the founder of the project. Pull requests should be sent to the master repository at https://github.com/SoftEtherVPN/SoftEtherVPN. \- GitHub, accessed October 22, 2025, [https://github.com/SoftEtherVPN/SoftEtherVPN\_Stable](https://github.com/SoftEtherVPN/SoftEtherVPN_Stable)  
5. AutoIT \- the Language of Choice \- \- OpenHoldem Documentation, accessed October 22, 2025, [https://documentation.help/OpenHoldem/AutoIT.html](https://documentation.help/OpenHoldem/AutoIT.html)  
6. How to Set Up a VPN Connection Using SoftEther VPN Client, accessed October 22, 2025, [https://www.winserver.net/blog/how-to-setup-softether-vpn-client/](https://www.winserver.net/blog/how-to-setup-softether-vpn-client/)  
7. How to Setup Softether on Windows 10 \- hide.me, accessed October 22, 2025, [https://hide.me/en/knowledgebase/setup-windows10-softether/](https://hide.me/en/knowledgebase/setup-windows10-softether/)  
8. How to setup and install SoftEther VPN client in your windows system? \- WebScoot Support, accessed October 22, 2025, [https://support.webscoot.io/portal/en/kb/articles/how-to-setup-and-install-softether-vpn-client-in-windows](https://support.webscoot.io/portal/en/kb/articles/how-to-setup-and-install-softether-vpn-client-in-windows)  
9. How to set up SoftEther VPN on Windows | CactusVPN, accessed October 22, 2025, [https://www.cactusvpn.com/tutorials/how-to-set-up-softether-vpn-on-windows/](https://www.cactusvpn.com/tutorials/how-to-set-up-softether-vpn-on-windows/)  
10. How to Install and Use SoftEther VPN Client on Windows | VPS Tutorial \- YouTube, accessed October 22, 2025, [https://www.youtube.com/watch?v=1w-54Yp-NRA](https://www.youtube.com/watch?v=1w-54Yp-NRA)  
11. Functions \- AutoIt, accessed October 22, 2025, [https://www.autoitscript.com/autoit3/docs/functions.htm](https://www.autoitscript.com/autoit3/docs/functions.htm)  
12. SoftEther Download Center, accessed October 22, 2025, [https://www.softether-download.com/](https://www.softether-download.com/)