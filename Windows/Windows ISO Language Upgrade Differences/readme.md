
# **An In-Depth Analysis of Windows 11 ISO Variants, Localization, and Upgrade Compatibility**

### **Executive Summary**

This report provides a comprehensive analysis of the differences between the standard English (United States) Windows 11 installation media and its international counterparts, addressing the critical question of cross-language upgrade compatibility. The investigation begins by deconstructing the specific ISO filename en-us\_windows\_11\_consumer\_editions\_version\_25h2\_x64\_dvd\_9934ee4c.iso, revealing that each component of the name is a precise data descriptor for its language, edition bundle, version, and architecture. The analysis clarifies that there is no singular "international" version of Windows; rather, Microsoft produces dozens of distinct, fully localized installation media for different global markets, each with pre-configured regional settings and, in some cases, regulatory-driven feature variations.

A central finding of this report is the critical and often misunderstood distinction between the operating system's foundational "base language," which is immutably set during the initial installation, and the superficial "display language," which can be changed by the user post-installation via language packs. Core system components, including the Windows Recovery Environment and certain built-in account names, are permanently tied to this base language.

Consequently, the report establishes the cardinal rule of in-place upgrades: to preserve installed applications and settings, the base language of the upgrade media must exactly match the base language of the existing Windows installation. Attempting an upgrade with mismatched language media will result in the installer disabling the option to keep applications, forcing a choice between a partial migration that preserves only user files or a complete clean installation. The report concludes with strategic recommendations for both IT administrators and individual users, emphasizing the importance of standardizing on a single base language for managed environments and the necessity of verifying the system's true base language before initiating any upgrade to prevent unintended data loss.

## **Section 1: Deconstructing the Windows 11 Installation Media**

The foundation of understanding Windows 11 distribution lies in decoding the information embedded within the installation media's filename. This structured naming convention is not arbitrary; it is a form of metadata that precisely defines the media's contents, intended audience, and technical specifications.

### **1.1 Analysis of the ISO Filename: A Structured Data String**

The filename en-us\_windows\_11\_consumer\_editions\_version\_25h2\_x64\_dvd\_9934ee4c.iso serves as a detailed manifest. Each segment provides essential information about the package. The deliberate separation of ISO types, such as consumer\_editions from business\_editions, is the first indication of Microsoft's channel-based distribution strategy. This approach segments the market and controls access to volume-licensed SKUs like Windows 11 Enterprise, which are typically distributed through dedicated portals such as the Microsoft 365 admin center or the Visual Studio Subscriptions portal.1 An organization cannot use a publicly downloaded consumer ISO to deploy Windows 11 Enterprise because the necessary image is not included in the installation file. The filename, therefore, is the first gate in the licensing and deployment process.

---

**Table 1: ISO Filename Component Breakdown**

| Component | Description | Significance | Example Values |
| :---- | :---- | :---- | :---- |
| en-us | Language-Region Code | Sets the immutable base language of the OS, defining core UI elements and default regional settings. | de-DE, fr-FR, en-GB, ja-JP |
| windows\_11 | Product | Identifies the major operating system product line. | windows\_10 |
| consumer\_editions | SKU Bundle | A "multi-edition" package containing SKUs for the consumer market, such as Home and Pro. | business\_editions |
| version\_25h2 | Feature Update | Specifies the OS version. "25" indicates the target year (2025) and "H2" the second-half release.3 | 23H2, 24H2 |
| x64 | Architecture | Designates the 64-bit instruction set for modern Intel/AMD processors.1 | Arm64 |
| dvd | Media Format | A legacy descriptor for a disk image (.iso) file. | N/A |
| 9934ee4c | Release Hash | A unique identifier for this specific build, used to verify file integrity via checksums.1 | Varies per release |

---

### **1.2 Inside the ISO: The install.wim and Edition Indexing**

The functional core of the Windows installation media resides in the \\sources\\ directory within a single, large file. This file is typically named install.wim (Windows Imaging Format) or install.esd (Electronic Software Download), with the latter being a more highly compressed version of the former.7 This file is not a monolithic image but a container holding multiple, distinct images of the Windows operating system.

This architecture is what allows a single "multi-edition" ISO to facilitate the installation of various Windows 11 SKUs.1 For the consumer\_editions ISO, this container includes images for Windows 11 Home, Pro, Pro for Workstations, Education, and others targeted at the consumer and small business markets.9 During the setup process, the installer uses the provided product key—or one embedded in the device's firmware—to select and deploy the corresponding image from the container.

The specific editions contained within an install.wim or install.esd file can be inspected using the command-line Deployment Image Servicing and Management (DISM) tool. By mounting the ISO file to a drive letter (e.g., F:), an administrator can run the following command to list all available images:

dism /Get-WimInfo /WimFile:F:\\sources\\install.wim

The output will detail each available Windows edition, assigning it an "Index" number. This index is the internal identifier that the setup process uses to install the correct version of the operating system.7

## **Section 2: The Landscape of "International" Windows 11 ISOs**

The term "international ISO" does not refer to a single, universal version of Windows. Instead, it encompasses a wide array of distinct installation media, each tailored for a specific language and region. These variations range from subtle differences in default settings to significant, legally mandated changes in the included feature set.

### **2.1 Defining "International": A Collection of Localized Builds**

Microsoft produces and distributes dozens of unique, language-specific ISO files to cater to its global user base.9 An "international ISO" is simply any installation media with a language-region code other than en-us. Examples include de-DE for German (Germany), es-MX for Spanish (Mexico), and en-GB for English (United Kingdom).

It is crucial to understand that these are not en-US builds with a language pack applied on top. Each localized ISO is a complete build where the specified language is the foundational, or "base," language of the operating system. This means all core components, from the initial setup screens to the Windows Recovery Environment, are natively in the designated language.

### **2.2 A Tale of Two Englishes: en-US vs. en-GB**

A common point of confusion for English-speaking users is the choice between the "English" (en-US) ISO and the "English International" ISO, which is typically the en-GB (United Kingdom) version. While the core features of the operating system are identical, the two versions differ significantly in their default configurations, reflecting an effort to align the user experience with regional conventions.11 This pre-configuration of a "locale profile" demonstrates that localization is a deep, systemic process that embeds cultural norms for timekeeping, measurement, and communication, rather than a simple cosmetic translation. For a global organization, this implies that deploying a single en-US image and managing regional settings via policy may offer greater consistency than allowing the use of various regional ISOs.

---

**Table 2: Regional Default Settings Comparison: en-US vs. en-GB**

| Setting | en-US Default (English) | en-GB Default (English International) |
| :---- | :---- | :---- |
| Date Format | Month/Day/Year (MM/DD/YYYY) | Day/Month/Year (DD/MM/YYYY) |
| Time Format | 12-hour clock (AM/PM) | 24-hour clock |
| Week Start Day | Sunday | Monday |
| Spelling Example | "Color," "Personalization," "Favorite" | "Colour," "Personalisation," "Favourite" |
| Default Keyboard | US QWERTY | UK QWERTY |
| Currency Symbol | $ (Dollar) | £ (Pound Sterling) |

---

### **2.3 Specialized Regional Variants: The "N" and "KN" Editions**

Beyond localization, Microsoft also produces special editions of Windows to comply with regional legal and regulatory requirements. These editions differ not in language but in their included feature set.

* **Windows N Editions:** These versions are specifically for the European market, created in response to a 2004 European Commission antitrust ruling.13 The "N" stands for "Not with Media Player." These editions (e.g., Windows 11 Pro N) are identical to their standard counterparts but exclude certain media-related technologies. Key omissions include Windows Media Player, Groove Music, Movies & TV, and various audio and video codecs.15 The absence of this underlying media infrastructure can also affect other functionalities, such as file transfers to MTP devices.14 Users can restore the missing functionality by installing the free "Media Feature Pack" from Microsoft as an optional feature.17  
* **Windows KN Editions:** A similar variant was created for the Korean market, which also has media-related technologies removed.14

## **Section 3: The Foundational Language Layer: Base Install vs. Language Packs**

A core architectural principle of Windows is the distinction between the permanent base language of the installation and the flexible display language used by an individual. This distinction is fundamental to understanding the constraints of system upgrades and the true nature of localization in Windows. The immutability of the base language reveals that Windows is not a truly language-agnostic platform with a simple UI layer on top; rather, language is woven into its foundational fabric. This is a form of technical debt from the deep integration of language strings into the core Windows NT architecture, where components created during setup become permanently stamped. For system administrators, this reality makes standardization on a single base language (like en-US) a superior strategy for ensuring predictability in scripting and automation, which often rely on non-localized identifiers like Security IDs (SIDs) or environment variables.19

### **3.1 The Immutable Core: The Base UI Language**

The language of the ISO used for the initial clean installation of Windows establishes the system's "base UI language" or "install language." This is a foundational setting that cannot be fully altered without reinstalling the operating system.19 Several critical, low-level components are permanently set in this base language:

* **Built-in User Accounts:** The names of well-known security principals, such as the default Administrator account, are created in the base language and will not change even if a different display language is applied later.19  
* **Core System Paths:** While File Explorer may display localized folder names (e.g., Benutzer for Users in German), the underlying physical paths on the disk remain in English (e.g., C:\\Program Files, C:\\Users). Critically, the system's environment variables (%ProgramFiles%, %SystemRoot%) always point to these non-localized English paths, ensuring script and software compatibility across different language installations.19  
* **Windows Recovery Environment (WinRE):** The language used in the advanced startup and recovery tools is tied directly to the base installation language. A system installed from a French ISO will have a French-language WinRE, which cannot be changed by installing a user-level language pack.21  
* **Registry and Services:** Many fundamental registry keys and the names of core system services are written during setup using the base language and remain unchanged.

### **3.2 The Superficial Layer: Language Packs and Display Language**

After Windows is installed, users can add other languages through the Settings app (Settings \> Time & language \> Language & region).23 Installing a "language pack" allows a user to change their "Windows display language." This provides a localized experience for that user's session, translating the Start menu, File Explorer, settings panels, dialog boxes, and modern applications.22

However, this change is superficial. It does not alter the underlying base UI language of the system. A user can be working in a fully German interface, but if the operating system was originally installed from an en-US ISO, the core system remains English. This can sometimes lead to a mixed-language experience, where deep system error messages, boot screens, or legacy components may still appear in the original base language.19 Microsoft provides full Language Packs (LPs) for comprehensive translation and smaller Language Interface Packs (LIPs), which offer partial localization and require a parent language pack to function.21

## **Section 4: In-Place Upgrade Mechanics and Cross-Language Constraints**

The process of performing an in-place upgrade of Windows is governed by strict rules, particularly concerning language. These constraints are not arbitrary; they are a critical risk-mitigation feature embedded within the Windows Setup engine. The engine prioritizes post-upgrade system stability over user convenience. An in-place upgrade involves a complex migration of applications, drivers, and settings. If the base languages differ, the setup engine cannot reliably map the registry structures and system identifiers from the old installation to the new one. Attempting such a migration could lead to a corrupted, unstable, or unbootable system. Therefore, the setup engine identifies a language mismatch as an unacceptable risk and defaults to a "fail-safe" path—wiping applications—to guarantee a clean and stable OS.

### **4.1 The Cardinal Rule of In-Place Upgrades**

To perform an in-place upgrade while preserving all user applications, files, and settings (the "Keep personal files and apps" option), there is a non-negotiable requirement: **The base language of the installation media must be an exact match for the base language of the currently installed operating system**.5

This rule applies to all forms of in-place upgrades, including major version jumps (e.g., Windows 10 to Windows 11\) and feature updates within Windows 11 that require full media (e.g., upgrading from a version older than 24H2 to 25H2).25 For example, a system that was initially installed using an en-US ISO can only be upgraded with another en-US ISO if the goal is to keep all applications intact.

### **4.2 Consequences of a Language Mismatch**

When an in-place upgrade is initiated with mismatched languages—for instance, using en-GB installation media on a system with an en-US base language—the Windows Setup program will detect the conflict during its initial checks. The consequences are immediate and restrictive:

* The option to **"Keep personal files and apps" will be disabled** and greyed out in the setup wizard.20  
* A message will typically be displayed, informing the user that applications cannot be kept because the installation language differs from the system's current language.20  
* The user will be forced to select one of two less desirable outcomes 27:  
  1. **Keep personal files only:** This option preserves the contents of the user's profile folders (Documents, Pictures, etc.) but removes all installed applications and resets all Windows settings.  
  2. **Nothing:** This option performs a clean installation of the new operating system, wiping the system partition entirely. All applications, user files, and settings are deleted.

---

**Table 3: In-Place Upgrade Language Compatibility Matrix**

| Installed Base Language | Upgrade Media Language | "Keep Apps & Files" Available? | Outcome / Recommendation |
| :---- | :---- | :---- | :---- |
| en-US | en-US | Yes | Supported path. Upgrade will proceed while preserving applications, files, and settings. |
| en-US | en-GB | No | Mismatch detected. The option to keep applications will be disabled. Abort and download matching en-US media. |
| de-DE | de-DE | Yes | Supported path. Upgrade will proceed while preserving applications, files, and settings. |
| de-DE | en-US | No | Mismatch detected. The option to keep applications will be disabled. Abort and download matching de-DE media. |

---

### **4.3 Verification and Prevention: The Definitive Check**

A user's display language, as configured in the Settings app, can be misleading and is not a reliable indicator of the system's true base language. A system can be configured to display "English (United Kingdom)" while its underlying base language is actually en-US. To avoid catastrophic data loss by mistakenly using the wrong upgrade media, it is essential to verify the base language definitively before proceeding.

This can be accomplished with a single command run from an administrative Command Prompt or PowerShell terminal 20:

DISM /online /get-intl

The output of this command includes a line for "Default system UI language," which displays the language-region code (e.g., en-US) of the immutable base installation. This is the language that the upgrade media must match.

## **Section 5: Strategic Recommendations for System Deployment and Management**

Based on the technical analysis of Windows 11 installation media, localization, and upgrade constraints, the following strategic recommendations are provided for both professional IT environments and individual power users.

### **5.1 For System Administrators and IT Professionals**

* **Standardize on a Single Base Image:** In multilingual corporate or educational environments, the most effective management strategy is to standardize all system deployments on a single base language. en-US is often the preferred choice due to its status as the de facto industry standard and the typical speed at which documentation, support resources, and security patches are released for it.  
* **Deploy Language Packs Centrally:** Rather than maintaining multiple language-specific images, a single en-US base image should be deployed to all endpoints. Required language packs and features can then be deployed centrally to user devices via management tools like Microsoft Intune or Microsoft Configuration Manager.28 This approach creates a homogenous and predictable underlying environment that is significantly easier to service and upgrade, while still providing end-users with their preferred localized display language.  
* **Simplify Servicing and Upgrades:** This standardized model drastically simplifies the feature update process. Only a single en-US upgrade package or ISO is needed to service the entire fleet of devices, reducing the complexity of testing, validation, and deployment.

### **5.2 For Power Users and Individuals**

* **Choose Your Initial ISO Carefully:** The language of the ISO used for the very first installation of Windows on a machine defines its core identity permanently. It is critical to select the ISO that best matches the primary, long-term language and region for the device to ensure the most consistent and seamless user experience.  
* **Always Verify Before Upgrading:** Before committing to a multi-gigabyte download for an in-place upgrade, always verify the system's true base language. Open an administrative Command Prompt or PowerShell and run the command DISM /online /get-intl.20 Note the language code provided for the "Default system UI language."  
* **Download the Correct Matching ISO:** Use the language code verified in the previous step to select the exact corresponding language from the dropdown menu on the official Microsoft Windows 11 software download page.1 This simple, one-minute check is the most effective way to prevent the loss of installed applications and the hours of work required to reinstall and reconfigure them.

#### **Works cited**

1. Download Windows 11 \- Microsoft, accessed October 28, 2025, [https://www.microsoft.com/en-us/software-download/windows11](https://www.microsoft.com/en-us/software-download/windows11)  
2. Windows 11 Enterprise | Microsoft Evaluation Center, accessed October 28, 2025, [https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise)  
3. Windows 11 Home and Pro \- Microsoft Lifecycle, accessed October 28, 2025, [https://learn.microsoft.com/en-us/lifecycle/products/windows-11-home-and-pro](https://learn.microsoft.com/en-us/lifecycle/products/windows-11-home-and-pro)  
4. Windows 11 editions explained: Versions, SKUs, and Home vs. Pro ..., accessed October 28, 2025, [https://www.zdnet.com/article/windows-11-editions-explained-versions-skus-and-home-vs-pro/](https://www.zdnet.com/article/windows-11-editions-explained-versions-skus-and-home-vs-pro/)  
5. Download Windows 11 \- Microsoft, accessed October 28, 2025, [https://www.microsoft.com/en-gb/software-download/windows11](https://www.microsoft.com/en-gb/software-download/windows11)  
6. Download Windows 11 Arm64 \- Microsoft, accessed October 28, 2025, [https://www.microsoft.com/en-us/software-download/windows11arm64](https://www.microsoft.com/en-us/software-download/windows11arm64)  
7. How to Find the Windows 11 or Windows 10 Version Number Using the ISO File | Dell US, accessed October 28, 2025, [https://www.dell.com/support/kbdoc/en-us/000113418/how-to-find-the-windows-10-version-number-using-the-iso-file](https://www.dell.com/support/kbdoc/en-us/000113418/how-to-find-the-windows-10-version-number-using-the-iso-file)  
8. How to Find the Windows 11 or Windows 10 Version Number Using the ISO File \- Dell, accessed October 28, 2025, [https://www.dell.com/support/kbdoc/en-ed/000113418/how-to-find-the-windows-10-version-number-using-the-iso-file](https://www.dell.com/support/kbdoc/en-ed/000113418/how-to-find-the-windows-10-version-number-using-the-iso-file)  
9. Download the official Windows 11 version 25H2 RTM ISOs here \- Windows Central, accessed October 28, 2025, [https://www.windowscentral.com/microsoft/windows-11/microsofts-official-windows-11-version-25h2-rtm-iso-media-is-now-available-download-all-28-languages-here-for-x64-or-arm64](https://www.windowscentral.com/microsoft/windows-11/microsofts-official-windows-11-version-25h2-rtm-iso-media-is-now-available-download-all-28-languages-here-for-x64-or-arm64)  
10. Windows 11 Download | MAS \- Microsoft Activation Scripts, accessed October 28, 2025, [https://massgrave.dev/windows\_11\_links](https://massgrave.dev/windows_11_links)  
11. Windows 11 (and 10\) English vs. English International \- Pureinfotech, accessed October 28, 2025, [https://pureinfotech.com/windows-10-english-vs-english-international/](https://pureinfotech.com/windows-10-english-vs-english-international/)  
12. Difference between English and English International for Windows 10 .ISO? \- Super User, accessed October 28, 2025, [https://superuser.com/questions/1115981/difference-between-english-and-english-international-for-windows-10-iso](https://superuser.com/questions/1115981/difference-between-english-and-english-international-for-windows-10-iso)  
13. Licence key differences between win 11 home and win 11 home N \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/answers/questions/1338769/licence-key-differences-between-win-11-home-and-wi](https://learn.microsoft.com/en-us/answers/questions/1338769/licence-key-differences-between-win-11-home-and-wi)  
14. What Are the Windows N Editions, and Should You Use Them? \- MakeUseOf, accessed October 28, 2025, [https://www.makeuseof.com/windows-n-editions-guide/](https://www.makeuseof.com/windows-n-editions-guide/)  
15. support.microsoft.com, accessed October 28, 2025, [https://support.microsoft.com/en-us/topic/media-feature-pack-list-for-windows-n-editions-c1c6fffa-d052-8338-7a79-a4bb980a700a\#:\~:text=N%20editions%20of%20Windows%20include,to%20restore%20these%20excluded%20technologies.](https://support.microsoft.com/en-us/topic/media-feature-pack-list-for-windows-n-editions-c1c6fffa-d052-8338-7a79-a4bb980a700a#:~:text=N%20editions%20of%20Windows%20include,to%20restore%20these%20excluded%20technologies.)  
16. Windows 11 Pro vs Pro N \- Which version is superior? \- RoyalCDKeys, accessed October 28, 2025, [https://royalcdkeys.com/blogs/news/windows-11-pro-vs-pro-n-which-version-is-superior](https://royalcdkeys.com/blogs/news/windows-11-pro-vs-pro-n-which-version-is-superior)  
17. Windows N editions | NTLite Forums, accessed October 28, 2025, [https://www.ntlite.com/community/index.php?threads/windows-n-editions.4367/](https://www.ntlite.com/community/index.php?threads/windows-n-editions.4367/)  
18. Media Feature Pack list for Windows N editions \- Microsoft Support, accessed October 28, 2025, [https://support.microsoft.com/en-us/topic/media-feature-pack-list-for-windows-n-editions-c1c6fffa-d052-8338-7a79-a4bb980a700a](https://support.microsoft.com/en-us/topic/media-feature-pack-list-for-windows-n-editions-c1c6fffa-d052-8338-7a79-a4bb980a700a)  
19. Installing Windows 11 Pro via EN ISO is equivalent to install from a localized version patched with Language Pack EN? \- Super User, accessed October 28, 2025, [https://superuser.com/questions/1888805/installing-windows-11-pro-via-en-iso-is-equivalent-to-install-from-a-localized-v](https://superuser.com/questions/1888805/installing-windows-11-pro-via-en-iso-is-equivalent-to-install-from-a-localized-v)  
20. Windows in-place upgrade won't keep apps due to different ..., accessed October 28, 2025, [https://support42.co.uk/windows-in-place-upgrade-wont-keep-apps-due-to-different-language/](https://support42.co.uk/windows-in-place-upgrade-wont-keep-apps-due-to-different-language/)  
21. Languages overview | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/languages-overview?view=windows-11](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/languages-overview?view=windows-11)  
22. Localize | Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/localize-windows?view=windows-11](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/localize-windows?view=windows-11)  
23. Language packs for Windows \- Microsoft Support, accessed October 28, 2025, [https://support.microsoft.com/en-us/windows/language-packs-for-windows-a5094319-a92d-18de-5b53-1cfc697cfca8](https://support.microsoft.com/en-us/windows/language-packs-for-windows-a5094319-a92d-18de-5b53-1cfc697cfca8)  
24. Windows 11 In Place Upgrade Encountering Multiple Languages : r/SCCM \- Reddit, accessed October 28, 2025, [https://www.reddit.com/r/SCCM/comments/1f8nt82/windows\_11\_in\_place\_upgrade\_encountering\_multiple/](https://www.reddit.com/r/SCCM/comments/1f8nt82/windows_11_in_place_upgrade_encountering_multiple/)  
25. Get ready for Windows 11, version 25H2 \- Windows IT Pro Blog \- Microsoft Community Hub, accessed October 28, 2025, [https://techcommunity.microsoft.com/blog/windows-itpro-blog/get-ready-for-windows-11-version-25h2/4426437](https://techcommunity.microsoft.com/blog/windows-itpro-blog/get-ready-for-windows-11-version-25h2/4426437)  
26. Can I upgrade to Windows 11? \- Microsoft Support, accessed October 28, 2025, [https://support.microsoft.com/en-us/windows/can-i-upgrade-to-windows-11-14c25efc-ecb7-4ce6-a3dd-7e2e24476997](https://support.microsoft.com/en-us/windows/can-i-upgrade-to-windows-11-14c25efc-ecb7-4ce6-a3dd-7e2e24476997)  
27. Ways to install Windows 11 \- Microsoft Support, accessed October 28, 2025, [https://support.microsoft.com/en-us/windows/ways-to-install-windows-11-e0edbbfb-cfc5-4011-868b-2ce77ac7c70e](https://support.microsoft.com/en-us/windows/ways-to-install-windows-11-e0edbbfb-cfc5-4011-868b-2ce77ac7c70e)  
28. Install language packs on Windows 11 Enterprise VMs in Azure Virtual Desktop \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/azure/virtual-desktop/windows-11-language-packs](https://learn.microsoft.com/en-us/azure/virtual-desktop/windows-11-language-packs)  
29. Windows in-place upgrade \- Configuration Manager \- Microsoft Learn, accessed October 28, 2025, [https://learn.microsoft.com/en-us/intune/configmgr/osd/deploy-use/upgrade-windows-to-the-latest-version](https://learn.microsoft.com/en-us/intune/configmgr/osd/deploy-use/upgrade-windows-to-the-latest-version)