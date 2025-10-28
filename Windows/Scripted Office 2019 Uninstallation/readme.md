
# **Enterprise Guide to Scripted Uninstallation of Office 2019 Using the Office Deployment Tool**

## **Introduction**

The lifecycle management of enterprise software extends beyond initial deployment to include consistent, reliable, and automated removal. For Microsoft Office 2019, which is deployed using the Office Deployment Tool (ODT), the uninstallation process is not a separate function but an integral part of the same configuration management framework. This report provides a definitive, expert-level guide for system administrators and IT professionals on performing a complete, unattended uninstallation of Office 2019\. The methodology leverages the same tools used for installation—the ODT executable (setup.exe) and a custom configuration.xml file—ensuring a consistent and vendor-supported approach.

The core premise of this guide is that uninstallation is simply another configuration state to be applied to a target system. The primary challenge in automating this process lies in achieving a truly "unattended" execution, which requires anticipating and programmatically mitigating common blockers such as running Office applications, incorrect product targeting, or insufficient permissions. This report serves as a comprehensive playbook, detailing the creation, execution, and verification of a scripted Office 2019 uninstallation. It covers the standard procedures for reliable mass deployment and provides robust contingency plans and alternative strategies for addressing non-standard scenarios, such as corrupted installations, ensuring a high success rate across any enterprise environment.

## **Section 1: Foundational Concepts for Office 2019 Uninstallation**

A successful scripted uninstallation strategy depends on a clear understanding of the underlying technology and terminology. Misconceptions about the Office Deployment Tool's function and the architecture of Office 2019 are the primary source of failed automation attempts. This section establishes the critical technical context necessary for building a reliable removal process.

### **The Office Deployment Tool's Role Beyond Initial Installation**

The Office Deployment Tool (setup.exe) is fundamentally a configuration engine, not merely an installer. Its sole purpose is to read a manifest file—configuration.xml—and reconcile the state of the local machine with the desired state defined within that manifest.1 This paradigm is crucial for understanding uninstallation. When executing the ODT, an administrator is not running a traditional "uninstaller" with command-line switches. Instead, they are applying a configuration that specifies the *absence* of a particular Office product. The ODT then performs the necessary actions (in this case, removal) to make the system's state match the XML file's instructions.2

### **Critical Distinction: Click-to-Run (C2R) vs. Windows Installer (MSI) Architecture**

Understanding the installation technology of the target software is paramount. Microsoft Office 2019 is distributed exclusively as a Click-to-Run (C2R) product.3 This modern installation architecture is fundamentally different from the legacy Windows Installer (MSI) technology used for Office 2016 and earlier versions. This distinction directly impacts the structure of the configuration.xml file.

The ODT's XML schema contains separate, mutually exclusive elements to manage these two architectures:

* **The \<Remove\> Element:** This element is used exclusively to manage C2R products. To uninstall Office 2019, this is the correct and only element to use within the configuration file.2  
* **The \<RemoveMSI /\> Element:** This element is designed *only* to remove older, MSI-based versions of Office. It is typically used during an upgrade or migration scenario, for example, when installing a C2R version of Office 365 or Office 2019 and simultaneously cleaning up a legacy Office 2013 MSI installation.6

Using the \<RemoveMSI /\> element in an attempt to uninstall a C2R-based Office 2019 installation will have no effect. The ODT will execute without error but will not perform any action on the Office 2019 suite because the element targets the wrong installation technology. This is one of the most common points of failure for administrators new to C2R management, often stemming from experience with older MSI-based Office products. Official documentation explicitly clarifies that \<RemoveMSI\> does not uninstall C2R versions and that the \<Remove\> element must be used for that purpose.7

### **The Configuration.xml File as the Central Control Manifest**

Every action performed by the ODT is dictated by the contents of the configuration.xml file. For a successful unattended uninstallation, this file must be meticulously crafted to contain only the necessary removal instructions and the parameters that enforce silent execution. The file must be well-formed XML; any syntax errors, misplaced tags, or incorrect Product IDs will cause the process to fail, often with little to no user-facing feedback. The only evidence of failure may be cryptic entries in the ODT's log files, making a correct initial configuration essential.

## **Section 2: The Primary Method: Unattended Uninstallation via ODT and XML Configuration**

This section provides a detailed, step-by-step methodology for creating the uninstall.xml file and executing the removal process in a scripted, unattended manner. Each XML element and attribute is analyzed in the context of achieving a silent and reliable uninstallation.

### **Crafting the uninstall.xml File: A Detailed Analysis**

A minimal and effective uninstall.xml file contains three key components: instructions on what to remove, how to handle running applications, and how to manage the user interface.

#### **The \<Configuration\> Root Element**

This is the mandatory root element that encapsulates all other instructions for the ODT. All subsequent elements must be nested within the opening \<Configuration\> and closing \</Configuration\> tags.

#### **The \<Remove\> Element: Targeting the Product**

This element is the core instruction that directs the ODT to perform a removal operation on a C2R product. It can be configured in two primary ways:

1. **Targeting a Specific Product ID:** The most precise method is to specify the exact Product ID of the Office 2019 suite installed on the target machines. A mismatch between the ID in the XML and the ID of the installed product will result in the ODT taking no action. For example, to remove the volume-licensed version of Office Professional Plus 2019, the \<Product\> element would be used as follows 9:  
   XML  
   \<Remove\>  
     \<Product ID\="ProPlus2019Volume" /\>  
   \</Remove\>

2. **Comprehensive Removal with All="TRUE":** A more robust and highly recommended approach for complete removal is to use the All="TRUE" attribute within the \<Remove\> tag. This attribute instructs the ODT to remove *all* installed C2R Office products from the machine, including any standalone installations of Visio or Project.4 This method simplifies scripting for environments with diverse Office product deployments, as it eliminates the need to identify and target each specific Product ID.  
   XML  
   \<Remove All\="TRUE" /\>

#### **Ensuring Silent Execution: The \<Display\> Element**

To achieve a truly unattended process, all user interface elements must be suppressed. This is controlled by the \<Display\> element and its attributes.2

* Level="None": This attribute prevents any progress bars, dialog boxes, or completion notifications from appearing on the user's screen.  
* AcceptEULA="TRUE": This attribute is mandatory for silent execution. It programmatically accepts the End User License Agreement, bypassing the need for manual user interaction.

The complete element should be formatted as:  
\<Display Level="None" AcceptEULA="TRUE" /\>

#### **Mitigating Conflicts: The FORCEAPPSHUTDOWN Property**

One of the most common reasons for uninstallation failure in a live user environment is that Office applications (such as Outlook, Word, or Excel) are running. The ODT cannot remove files that are in use. The FORCEAPPSHUTDOWN property provides a powerful solution to this problem by instructing the ODT to forcibly terminate any blocking Office processes before beginning the removal.5

The syntax for this property is:  
\<Property Name="FORCEAPPSHUTDOWN" Value="TRUE" /\>  
It is critical to note that using this property can lead to data loss if users have unsaved work in the applications that are terminated. Therefore, its use should be aligned with enterprise policies and, if possible, communicated to users in advance. However, for ensuring the highest success rate in a scripted, large-scale deployment, this property is often essential.

### **Complete uninstall.xml Example**

Combining these elements yields a robust and effective configuration file for the complete and silent uninstallation of all Office 2019 products.

XML

\<Configuration\>  
  \<Remove All\="TRUE" /\>  
  \<Property Name\="FORCEAPPSHUTDOWN" Value\="TRUE" /\>  
  \<Display Level\="None" AcceptEULA\="TRUE" /\>  
\</Configuration\>

### **Executing and Verifying the Uninstallation**

With the setup.exe and uninstall.xml files in the same directory, the uninstallation can be initiated from an elevated command prompt or script.

#### **Command-Line Syntax**

The command uses the /configure switch, which instructs setup.exe to apply the settings from the specified XML file.1

setup.exe /configure uninstall.xml

#### **Implementation in Scripts**

For scalable deployment, this command should be incorporated into a script.

* **Batch Script:** Using "%\~dp0" ensures that the script correctly locates setup.exe and uninstall.xml when they are in the same directory as the batch file itself.  
  Code snippet  
  "%\~dp0setup.exe" /configure "%\~dp0uninstall.xml"

* **PowerShell Script:** The Start-Process cmdlet is recommended, particularly with the \-Wait parameter, which forces the script to pause until the ODT process has completed before moving to the next command. This is crucial for sequential tasks and accurate logging.  
  PowerShell  
  Start-Process \-FilePath ".\\setup.exe" \-ArgumentList "/configure uninstall.xml" \-Wait \-NoNewWindow

#### **Verification and Logging**

Troubleshooting ODT operations relies heavily on its log files.

* **Log File Location:** The ODT automatically generates detailed log files in the user's temporary directory (%temp%) and the system's temp directory (C:\\Windows\\Temp).12 The log file is typically named in the format MachineName-YYYYMMDD-HHMMSS.log.  
* **Successful Execution:** A successful operation will conclude with a log entry stating "Products configured successfully".12  
* **Programmatic Verification:** For automated verification, scripts can check for the absence of key artifacts, such as the primary Office installation directory (C:\\Program Files\\Microsoft Office\\root\\Office16) or specific registry keys associated with the C2R installation, like HKEY\_LOCAL\_MACHINE\\SOFTWARE\\Microsoft\\Office\\ClickToRun.5

| Table 1: Common Office 2019 Product IDs for ODT |  |
| :---- | :---- |
| **Product Name** | **Product ID** |
| Office Professional Plus 2019 (Volume License) | ProPlus2019Volume |
| Office Standard 2019 (Volume License) | Standard2019Volume |
| Office Professional Plus 2019 (Retail) | ProPlus2019Retail |
| Office Home & Business 2019 (Retail) | HomeBusiness2019Retail |
| Office Home & Student 2019 (Retail) | HomeStudent2019Retail |
| Project Professional 2019 (Volume License) | ProjectPro2019Volume |
| Project Standard 2019 (Volume License) | ProjectStd2019Volume |
| Visio Professional 2019 (Volume License) | VisioPro2019Volume |
| Visio Standard 2019 (Volume License) | VisioStd2019Volume |

| Table 2: Key Configuration.xml Elements and Properties for Uninstallation |  |  |
| :---- | :---- | :---- |
| **Element / Property** | **Attribute / Value** | **Function in Unattended Uninstallation** |
| \<Remove\> | All="TRUE" | Instructs the ODT to remove all installed Click-to-Run Office products. This is the recommended approach for a complete cleanup. |
| \<Product\> | ID="Product ID" | Nested within \<Remove\>, this targets a specific Office product (e.g., ProPlus2019Volume) for removal. Use when All="TRUE" is not desired. |
| \<Display\> | Level="None" | Suppresses all user interface elements, including progress bars and completion dialogs, ensuring the process is completely silent. |
| \<Display\> | AcceptEULA="TRUE" | Programmatically accepts the license agreement, which is a mandatory requirement for any unattended ODT operation. |
| \<Property\> | Name="FORCEAPPSHUTDOWN" Value="TRUE" | Forcibly closes any running Office applications that would otherwise block the uninstallation process. Essential for reliability but may cause user data loss. |

## **Section 3: Advanced Scenarios and Strategic Customization**

The Office Deployment Tool's flexibility extends beyond simple, all-encompassing removal. By customizing the configuration.xml file, administrators can execute more complex and granular software management tasks, such as reclaiming licenses for specific applications or performing seamless in-place upgrades.

### **Granular Control: Selective Uninstallation of Individual Products**

In scenarios where the goal is not to remove the entire Office suite but to reclaim licenses for specific applications, the ODT provides precise control. Instead of using the All="TRUE" attribute, an administrator can specify one or more \<Product\> elements within the \<Remove\> block. This approach allows for the surgical removal of applications like Visio or Project while leaving the core Office suite (Word, Excel, Outlook, etc.) untouched.9

For example, to create a script that only uninstalls the volume-licensed version of Visio Professional 2019 from a user's machine, the uninstall.xml would be configured as follows:

XML

\<Configuration\>  
  \<Remove\>  
    \<Product ID\="VisioPro2019Volume" /\>  
  \</Remove\>  
  \<Display Level\="None" AcceptEULA\="TRUE" /\>  
\</Configuration\>

This targeted approach is highly valuable for software asset management and license compliance initiatives.

### **Consolidated Operations: Combining Removal and Installation for Upgrades**

The configuration.xml file is capable of holding multiple instructions that the ODT will execute in a predefined sequence. A single XML file can contain both \<Remove\> and \<Add\> elements, enabling a highly efficient, single-operation process for upgrades or platform transitions. When both elements are present, the ODT always processes the \<Remove\> instructions first, clearing the specified products before proceeding with the \<Add\> instructions.3

This is the standard method for performing an in-place upgrade, such as migrating a machine from Office 2019 to Microsoft 365 Apps. The configuration file would instruct the ODT to remove the Office 2019 product and then immediately install the desired Microsoft 365 Apps suite. This consolidated approach streamlines deployment, reduces script complexity, and minimizes machine downtime.

### **Best Practices: Using the Office Customization Tool (OCT) for Error-Free XML Generation**

For creating any configuration.xml file, the recommended starting point is the web-based Office Customization Tool (OCT), available at config.office.com.1 This tool provides a graphical user interface for selecting products, architecture, update channels, and other settings, which then generates a syntactically correct and validated XML file. Using the OCT significantly reduces the risk of manual typing errors that can cause ODT failures.

However, there is a critical consideration when using the OCT for uninstallation tasks. The tool's graphical interface is primarily designed for installation and migration scenarios. While it provides a clear option to add the \<RemoveMSI /\> element for cleaning up legacy Office versions, it does not offer a direct, user-facing option to generate the \<Remove\> element needed for C2R product uninstallation.5 This represents a functional gap in the tool's GUI.

This limitation necessitates a hybrid workflow for creating an uninstall.xml file:

1. Navigate to config.office.com and configure basic settings, such as architecture (e.g., 64-bit) and any logging preferences.  
2. Export the configuration.xml file from the OCT.  
3. Open the downloaded XML file in a text editor and manually insert the required removal elements. This typically involves adding \<Remove All="TRUE" /\> and \<Property Name="FORCEAPPSHUTDOWN" Value="TRUE" /\> within the \<Configuration\> tags.

This hybrid approach leverages the OCT for generating a valid base structure while requiring manual editing to implement the specific uninstallation logic. This underscores the importance for administrators to understand the underlying XML schema directly, as they cannot rely on the GUI tooling alone for this fundamental lifecycle management task.

## **Section 4: Alternative and Complementary Uninstallation Strategies**

While the Office Deployment Tool is the primary and recommended method for managing C2R installations, it operates on the assumption of a healthy, uncorrupted Office installation. In real-world enterprise environments, installations can become damaged, preventing the standard ODT removal process from completing successfully. A robust software management strategy must include contingency plans and alternative tools for these remediation scenarios.

### **Microsoft Support and Recovery Assistant (SaRA): The Deep-Cleaning Approach**

The Microsoft Support and Recovery Assistant (SaRA) is the official, powerful troubleshooting utility designed to diagnose, fix, and completely remove problematic Office installations.6 When the standard ODT removal method fails, SaRA is the recommended next step. It performs a significantly more thorough "scrub" of the system than the ODT, removing residual files, registry keys, and configuration settings that can be left behind or cause conflicts.

Crucially for enterprise automation, SaRA is not limited to its graphical interface. It includes a command-line executable, SaRAcmd.exe, which is designed specifically for scripted execution.6 This allows administrators to invoke SaRA's powerful cleaning capabilities in an unattended manner.

The syntax for using SaRA to silently uninstall Office 2019 is:

SaRAcmd.exe \-S OfficeScrubScenario \-AcceptEula \-OfficeVersion 2019

The \-S OfficeScrubScenario switch initiates the removal process, \-AcceptEula handles the license agreement for silent operation, and \-OfficeVersion 2019 targets the specific version. The tool also supports an \-OfficeVersion All parameter to remove all detected versions of Office from a machine, making it a versatile cleanup utility.8 The existence of this robust, scriptable "scrubber" tool is an acknowledgment that standard removal processes can fail and that a more aggressive, dedicated tool is sometimes necessary for remediation.

### **Legacy Toolset: The OffScrub VBS Scripts**

Before the development of SaRA, Microsoft provided a collection of Visual Basic Scripts known as "OffScrub" for deep cleaning Office installations.6 These scripts were the go-to solution for administrators dealing with corrupted Office versions. While SaRA is the modern and preferred replacement, the OffScrub scripts can still be found in enterprise script libraries and may be useful in certain legacy environments or as a last resort if both ODT and SaRA fail.

For C2R installations like Office 2019, the relevant script is OffScrubc2r.vbs. Executing it requires using the Windows Script Host's command-line engine, cscript.exe, with flags to ensure silent operation 6:

cscript.exe OffScrubc2r.vbs ALL /Quiet /NoCancel /Force

### **Direct System Interrogation and Removal**

For highly customized or targeted scripting, it is possible to interact with the system's application management components directly, though these methods are generally more fragile than using the official tools.

* **Registry Uninstall Strings:** The Windows Registry stores the specific uninstall command for every installed application. This string can be found under the key HKEY\_LOCAL\_MACHINE\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Uninstall\\.4 For Office C2R products, this command often invokes OfficeClickToRun.exe directly with a long series of specific parameters that identify the exact product to remove.10 While this method offers surgical precision, it is brittle and can break if the product version or system configuration changes.  
* **PowerShell for Microsoft Store Installations:** In the less common scenario where Office was installed via the Microsoft Store, the uninstallation must be handled using PowerShell's modern application management cmdlets. The Get-AppxPackage and Remove-AppxPackage commands are used to identify and remove these types of applications.15 This is a distinct architecture from standard C2R and requires a different toolset.

A comprehensive enterprise strategy should therefore be tiered: attempt uninstallation first with the standard ODT method. If that fails, escalate the process by using the more powerful SaRAcmd.exe for remediation.

## **Section 5: Best Practices, Troubleshooting, and Strategic Recommendations**

Synthesizing the technical details into a strategic framework is essential for successful enterprise-wide deployment. This section provides checklists, troubleshooting guides, and a decision-making matrix to ensure reliability and efficiency in the Office 2019 uninstallation process.

### **Pre-Deployment Checklist for Uninstallation Scripts**

Before deploying an uninstallation script to a production environment, a series of checks can prevent widespread failures.

1. **Confirm Product ID:** If not using All="TRUE", verify the exact Product ID of the target Office 2019 installation. A discrepancy is a common and silent point of failure.  
2. **Use the Latest ODT:** Always download the most recent version of the Office Deployment Tool from the Microsoft Download Center. Older versions may lack support for newer features or product versions.12  
3. **Test on a Representative Sample:** Execute the script and uninstall.xml on a small group of test machines that mirror the production environment. Check for successful removal and review the ODT log files for any warnings or errors.  
4. **Establish User Communication and Data Policy:** The uninstallation process itself does not remove user-created documents or files.16 However, the use of FORCEAPPSHUTDOWN can cause the loss of unsaved work. Determine the policy for handling this risk and communicate with end-users if the deployment will occur during work hours.

### **Troubleshooting Common Failure Points**

When an unattended uninstallation fails, the ODT log file in the %temp% directory is the primary diagnostic tool. The following are common failure scenarios and their solutions.

* **Failure Cause: Incorrect Product ID or No Matching Product**  
  * **Symptom:** The setup.exe process executes and terminates very quickly. No changes are made to the system, and no errors are displayed.  
  * **Solution:** Examine the ODT log file. It will contain entries indicating that it scanned for products to remove but found no match for the criteria specified in the configuration.xml. Correct the \<Product ID\> in the XML file or switch to the more reliable \<Remove All="TRUE" /\> attribute.  
* **Failure Cause: Running Office Applications**  
  * **Symptom:** The uninstallation process hangs, times out, or fails. The log file contains errors related to locked files or processes that could not be stopped.  
  * **Solution:** Ensure the \<Property Name="FORCEAPPSHUTDOWN" Value="TRUE" /\> line is present and correctly formatted in the uninstall.xml file. This is the most effective way to handle application conflicts in an automated script.  
* **Failure Cause: Insufficient Permissions**  
  * **Symptom:** The log file contains "Access Denied" errors or indicates a failure to write to system directories or registry hives.  
  * **Solution:** The uninstallation script must be executed with administrative privileges. When running manually, use the "Run as administrator" option. When deploying through a management system like SCCM or Intune, ensure the deployment is configured to run under the SYSTEM account.  
* **Failure Cause: Corrupted Office Installation**  
  * **Symptom:** The ODT process fails with unexpected or cryptic error codes in the log file, even when the XML is correct and permissions are sufficient.  
  * **Solution:** This is the primary indicator that the standard removal process cannot handle the state of the installation. Escalate to the remediation strategy outlined in Section 4: use the Microsoft Support and Recovery Assistant via the command line (SaRAcmd.exe) to perform a deep scrub of the installation.

### **Strategic Recommendations and Final Summary**

The choice of uninstallation tool and method should be guided by the specific scenario. The following matrix provides a clear decision-making framework for administrators.

| Table 3: Uninstallation Method Recommendation Matrix |  |  |  |  |
| :---- | :---- | :---- | :---- | :---- |
| **Method** | **Target Scenario** | **Scripting Complexity** | **Thoroughness** | **Vendor Recommendation** |
| **ODT with XML** | Standard, healthy installations; scripted enterprise removal; selective product removal. | Low | Standard | **Primary Method** |
| **SaRAcmd.exe** | Corrupted or damaged installations; ODT failures; need for a complete system scrub. | Low-Medium | High / Scrub | **Primary Remediation Tool** |
| **OffScrub VBS** | Legacy environments; last resort when ODT and SaRA both fail. | Medium | High / Scrub | Legacy |
| **Registry/PowerShell** | Highly specific, custom scripting; Microsoft Store app removal. | High | Variable | Niche / Situational |

## **Conclusion**

The unattended uninstallation of Microsoft Office 2019 is a core competency for modern IT administration, achievable through a systematic and well-understood application of the Office Deployment Tool. The key to success lies in recognizing the ODT as a configuration engine and crafting a precise configuration.xml manifest that defines the desired end state—a system with Office 2019 removed.

The primary strategy for all scripted removals should be the ODT, utilizing a manually verified uninstall.xml that specifies complete removal with the All="TRUE" attribute, enforces silent execution via the \<Display\> element, and ensures reliability with the FORCEAPPSHUTDOWN property. This approach provides a consistent, vendor-supported, and scalable solution for enterprise-wide software lifecycle management.

For the inevitable instances of installation corruption where the standard ODT process fails, the Microsoft Support and Recovery Assistant, executed via SaRAcmd.exe, serves as the essential remediation tool. By maintaining both the ODT and SaRA in their administrative toolset, IT professionals can build a resilient, tiered strategy capable of handling both standard uninstallation tasks and complex cleanup scenarios with confidence and efficiency.

#### **Works cited**

1. Office 2016 \- How to remove individual apps? \- Super User, accessed October 24, 2025, [https://superuser.com/questions/988002/office-2016-how-to-remove-individual-apps](https://superuser.com/questions/988002/office-2016-how-to-remove-individual-apps)  
2. A way to remove Office components silently : r/sysadmin \- Reddit, accessed October 24, 2025, [https://www.reddit.com/r/sysadmin/comments/11i4drc/a\_way\_to\_remove\_office\_components\_silently/](https://www.reddit.com/r/sysadmin/comments/11i4drc/a_way_to_remove_office_components_silently/)  
3. Worklet to first Uninstall Microsoft Office 2019 then install MS Office 2021 | Community, accessed October 24, 2025, [https://community.automox.com/find-share-worklets-12/worklet-to-first-uninstall-microsoft-office-2019-then-install-ms-office-2021-2410](https://community.automox.com/find-share-worklets-12/worklet-to-first-uninstall-microsoft-office-2019-then-install-ms-office-2021-2410)  
4. How to uninstall silently Office2019 via "ODT" : r/Lansweeper \- Reddit, accessed October 24, 2025, [https://www.reddit.com/r/Lansweeper/comments/taz4qs/how\_to\_uninstall\_silently\_office2019\_via\_odt/](https://www.reddit.com/r/Lansweeper/comments/taz4qs/how_to_uninstall_silently_office2019_via_odt/)  
5. Remove Any Installed Version of Microsoft Office When Deploying M365 Desktop Apps, accessed October 24, 2025, [https://smbtothecloud.com/remove-any-installed-version-of-microsoft-office-when-deploying-m365-desktop-apps/](https://smbtothecloud.com/remove-any-installed-version-of-microsoft-office-when-deploying-m365-desktop-apps/)  
6. How to Completely Uninstall Previous Versions of Office with Removal Scripts, accessed October 24, 2025, [https://woshub.com/complete-uninstall-previous-office-versions/](https://woshub.com/complete-uninstall-previous-office-versions/)  
7. Remove existing MSI versions of Office when upgrading to Microsoft ..., accessed October 24, 2025, [https://learn.microsoft.com/en-us/microsoft-365-apps/deploy/upgrade-from-msi-version](https://learn.microsoft.com/en-us/microsoft-365-apps/deploy/upgrade-from-msi-version)  
8. Plan an upgrade from older versions of Office to Microsoft 365 Apps, accessed October 24, 2025, [https://learn.microsoft.com/en-us/microsoft-365-apps/end-of-support/plan-upgrade-older-versions-office](https://learn.microsoft.com/en-us/microsoft-365-apps/end-of-support/plan-upgrade-older-versions-office)  
9. Uninstall Microsoft 365, 2021 and 2019 using Endpoint Central ..., accessed October 24, 2025, [https://www.manageengine.com/products/desktop-central/uninstall-microsoft365-2021-2019-apps.html](https://www.manageengine.com/products/desktop-central/uninstall-microsoft365-2021-2019-apps.html)  
10. Office 365 Uninstall : r/sysadmin \- Reddit, accessed October 24, 2025, [https://www.reddit.com/r/sysadmin/comments/14vph44/office\_365\_uninstall/](https://www.reddit.com/r/sysadmin/comments/14vph44/office_365_uninstall/)  
11. Uninstalling Microsoft Office 2019 using Endpoint Central \- ManageEngine, accessed October 24, 2025, [https://www.manageengine.com/products/desktop-central/ms-office-uninstall.html](https://www.manageengine.com/products/desktop-central/ms-office-uninstall.html)  
12. Overview of the Office Deployment Tool \- Microsoft 365 Apps, accessed October 24, 2025, [https://learn.microsoft.com/en-us/microsoft-365-apps/deploy/overview-office-deployment-tool](https://learn.microsoft.com/en-us/microsoft-365-apps/deploy/overview-office-deployment-tool)  
13. Office Deployment Tool 2021: Configuration and Setup Guide, accessed October 24, 2025, [https://www.wps.com/blog/office-deployment-tool-2021-configuration-and-setup-guide/](https://www.wps.com/blog/office-deployment-tool-2021-configuration-and-setup-guide/)  
14. Custom deployment of MS Office fails: "configuration file wasn't specified" \- Reddit, accessed October 24, 2025, [https://www.reddit.com/r/sysadmin/comments/1bofr85/custom\_deployment\_of\_ms\_office\_fails/](https://www.reddit.com/r/sysadmin/comments/1bofr85/custom_deployment_of_ms_office_fails/)  
15. Uninstall Microsoft 365 or Office from a PC, accessed October 24, 2025, [https://support.microsoft.com/en-us/office/uninstall-microsoft-365-from-a-pc-9dd49b83-264a-477a-8fcc-2fdf5dbf61d8](https://support.microsoft.com/en-us/office/uninstall-microsoft-365-from-a-pc-9dd49b83-264a-477a-8fcc-2fdf5dbf61d8)  
16. Uninstall Microsoft 365 or Office from a PC, accessed October 24, 2025, [https://support.microsoft.com/en-us/office/uninstall-microsoft-365-or-office-from-a-pc-9dd49b83-264a-477a-8fcc-2fdf5dbf61d8](https://support.microsoft.com/en-us/office/uninstall-microsoft-365-or-office-from-a-pc-9dd49b83-264a-477a-8fcc-2fdf5dbf61d8)  
17. Scenario: Office Uninstall \- Microsoft 365, accessed October 24, 2025, [https://learn.microsoft.com/en-us/troubleshoot/microsoft-365/admin/miscellaneous/assistant-office-uninstall](https://learn.microsoft.com/en-us/troubleshoot/microsoft-365/admin/miscellaneous/assistant-office-uninstall)  
18. How to Uninstall Microsoft Office Completely \- ALI TAJRAN, accessed October 24, 2025, [https://www.alitajran.com/uninstall-microsoft-office/](https://www.alitajran.com/uninstall-microsoft-office/)