
# **An In-Depth Analysis of PowerShell 7 and UTF-8 Encoding Issues in the Visual Studio Code Terminal**

## **The Labyrinth of Character Encoding in the Windows Ecosystem**

The challenge of ensuring correct character representation in modern development environments, particularly on the Windows platform, is rooted in a long history of evolving standards. The garbled output observed when using PowerShell 7 within the Visual Studio Code terminal is not a simple bug but a symptom of a deep-seated conflict between legacy architecture and modern, cross-platform conventions. To fully diagnose the issue, it is essential to first understand the foundational concepts of character encoding as they pertain to the Windows operating system and its development tools.

### **The Legacy of Code Pages: ANSI and OEM**

For much of its history, the Windows operating system relied on traditional 8-bit character sets, commonly known as code pages, to represent text.1 These code pages use a single byte (8 bits) to represent up to 256 different characters. While the first 128 characters were standardized as 7-bit ASCII, the upper 128 characters were used for language-specific symbols, such as accented letters, currency symbols, and box-drawing characters. This approach led to a proliferation of regional code pages; for example, Windows-1252 became the standard for Western European languages, while other code pages were used for Cyrillic or Central European languages.2

A critical and often confusing aspect of this legacy is the dual-code-page system within Windows. The operating system maintained two distinct default code pages:

1. **The ANSI Code Page:** This code page was used by the graphical user interface (GUI) subsystem and most traditional Windows applications. For systems in North America and Western Europe, this is typically Windows-1252.1  
2. **The OEM Code Page:** This code page was used by the console subsystem, including the Command Prompt (cmd.exe) and legacy MS-DOS applications. The OEM code page (e.g., Code Page 437 in the US) often differed from the ANSI page, particularly in its use of the upper character range for block graphics and line-drawing symbols.1

This duality meant that text data could be interpreted differently depending on whether it was displayed in a standard application window or a console window. This system's fundamental limitation is its inability to represent characters from multiple languages simultaneously. A file created on a system using a Western European code page would display incorrectly on a system configured for a Cyrillic language. This inherent ambiguity is a primary source of modern encoding problems, as many older Windows components and applications, including Windows PowerShell, will default to assuming text is encoded using the system's active ANSI or OEM code page if no other information is provided.2

### **The Rise of Unicode and the UTF-8 Standard**

To overcome the limitations of code pages, the Unicode standard was developed. Unicode provides a unique numerical value—a code point—for every character, regardless of the platform, program, or language.1 This universal character set is the foundation for modern text processing. However, the Unicode standard itself does not define how these code points are to be represented as bytes in memory or on disk. This is the role of encoding formats, the most important of which are UTF-16 and UTF-8.

* **UTF-16:** Historically, many Microsoft products, including the.NET Framework and the core Windows APIs, adopted UTF-16. In this format, most common characters are represented by two bytes. In older Microsoft documentation, this encoding is often referred to simply as "Unicode".1 Windows PowerShell, for instance, defaults to writing files with UTF-16 Little Endian (LE) encoding when using redirection operators.1  
* **UTF-8:** In contrast, UTF-8 has emerged as the de facto standard for the internet, Linux-based systems, and the broader world of cross-platform development.3 Its popularity stems from several key advantages: it uses a variable-width encoding, representing the most common English characters and symbols (7-bit ASCII) in a single byte, making it highly efficient for ASCII-heavy text. It can represent every character in the Unicode standard, and it does so without the byte-order complexities that can affect UTF-16.

The central conflict in the Windows ecosystem arises from the tension between these standards. The Unix and Linux worlds operate on an implicit assumption that text is encoded in UTF-8. The legacy Windows world operates on an implicit assumption that text is encoded in a regional ANSI or OEM code page. Modern tools like PowerShell 7 and Visual Studio Code attempt to bridge this gap by adopting the UTF-8 convention, but they must function within an operating system environment that is still deeply influenced by its legacy assumptions. This clash of implicit defaults is where most encoding errors originate.

### **The Byte Order Mark (BOM): A Critical Signifier**

To resolve the ambiguity between legacy code pages and UTF-8, the Byte Order Mark (BOM) plays a crucial role, especially within the Windows ecosystem. The BOM is a special sequence of bytes—specifically 0xEF, 0xBB, 0xBF—placed at the very beginning of a text file.10 Its presence acts as an unambiguous, explicit signal to any program reading the file that its contents are encoded in UTF-8.

In the context of Windows, the BOM's primary function is to serve as a compatibility shim. When a program like Windows PowerShell encounters a text file, it must guess the encoding. If it finds no BOM, it falls back to its default behavior, which is to assume the file is encoded using the system's legacy ANSI code page.2 If the file actually contains UTF-8-encoded characters, this leads to their misinterpretation and the appearance of garbled text. The presence of a UTF-8 BOM forces even these legacy-minded applications to correctly interpret the file as UTF-8.

However, the BOM is not a universal solution and comes with significant trade-offs. In the Unix/Linux world, where UTF-8 is the assumed default, the BOM is often unnecessary and can cause problems. Many Unix-heritage utilities and shell environments do not recognize the BOM and may treat it as visible, garbage characters at the beginning of the file.1 Therefore, while the BOM is essential for ensuring backward compatibility with tools like Windows PowerShell and the PowerShell ISE, it can hinder cross-platform compatibility.2 This makes the choice of whether to use a BOM a critical decision based on the target environments for a given script or file.

## **Deconstructing PowerShell's Encoding Behavior**

PowerShell's handling of character encoding is a complex and multifaceted topic, marked by a significant philosophical shift between its legacy, Windows-only version and its modern, cross-platform successor. Understanding the specific mechanisms PowerShell provides for managing encoding, and the stark differences in their default behaviors, is fundamental to diagnosing and resolving the issues encountered in Visual Studio Code.

### **The Great Divide: Windows PowerShell 5.1 vs. PowerShell 7+**

The transition from Windows PowerShell 5.1 to the modern PowerShell (version 7 and later) represents one of the most important changes in the platform's history, particularly concerning character encoding. The two versions operate on fundamentally different default assumptions, which is a common source of confusion and unexpected behavior.

* **Windows PowerShell (version 5.1 and earlier):** This version is deeply integrated with the traditional Windows environment and its encoding defaults are inconsistent and legacy-oriented.  
  * **Script Interpretation:** When reading a script file (.ps1) that lacks a BOM, Windows PowerShell falls back to the system's active ANSI code page (e.g., Windows-1252).1 This is the most frequent cause of errors when scripts with non-ASCII characters, created in modern editors, are run in older PowerShell environments.  
  * **File Output:** By default, redirection operators (\> and \>\>) and the Out-File cmdlet create files using UTF-16LE encoding, which is often labeled as "Unicode" in PowerShell's \-Encoding parameter options.1  
  * **Communication with External Programs:** The $OutputEncoding variable, which controls the encoding used to send string data to native executables, defaults to ASCII.9 This can lead to data loss when piping text containing non-ASCII characters to external command-line tools.  
* **PowerShell (version 7 and later):** As a cross-platform framework, modern PowerShell has standardized its behavior on BOM-less UTF-8, aligning with the prevailing convention on Linux and macOS.  
  * **Script Interpretation:** PowerShell 7 assumes script files without a BOM are encoded in UTF-8.1  
  * **File Output:** All standard file output mechanisms, including redirection operators and cmdlets like Out-File and Set-Content, now default to creating BOM-less UTF-8 files.1  
  * **Communication with External Programs:** The $OutputEncoding variable now defaults to BOM-less UTF-8, ensuring that Unicode characters are preserved when interacting with modern, UTF-8-aware external utilities.9

This deliberate and comprehensive shift to UTF-8 in PowerShell 7 is a significant improvement for cross-platform development. However, it also means that scripts and environments must be managed with an awareness of which version is being used, especially when backward compatibility with Windows PowerShell is a requirement.

| Feature | Windows PowerShell 5.1 Default | PowerShell 7+ Default |
| :---- | :---- | :---- |
| **Reading .ps1 files (no BOM)** | System ANSI Code Page (e.g., Windows-1252) | UTF-8 (no BOM) |
| **File Redirection (\>)** | UTF-16LE ("Unicode") | UTF-8 (no BOM) |
| **Out-File** | UTF-16LE ("Unicode") | UTF-8 (no BOM) |
| **Set-Content** | System ANSI Code Page (e.g., Windows-1252) | UTF-8 (no BOM) |
| **$OutputEncoding** | ASCII | UTF-8 (no BOM) |

### **The Three Pillars of PowerShell Encoding Configuration**

Resolving encoding issues in PowerShell requires understanding that there is no single "master" setting. Instead, control is distributed across three distinct mechanisms, each with a specific domain of influence. Applying the wrong tool to a problem—for example, changing $OutputEncoding with the expectation that it will affect file redirection—is a common mistake.

1. **$PSDefaultParameterValues:** This is a built-in preference variable that allows users to define default values for the parameters of any cmdlet.1 Its primary role in encoding management is to control the default behavior of cmdlets that have an \-Encoding parameter, such as Out-File, Set-Content, and Export-Csv. By adding a line to a PowerShell profile script, one can change the default file output encoding for an entire session. For instance, the command $PSDefaultParameterValues\['\*:Encoding'\] \= 'utf8' sets the default encoding to UTF-8 for all cmdlets that accept an \-Encoding parameter.7 In PowerShell 7, this simply reinforces the existing default, but in Windows PowerShell, it is a powerful way to override the legacy defaults.  
2. **$OutputEncoding:** This variable has a very specific and often misunderstood purpose: it dictates the character encoding that PowerShell uses when it converts its internal.NET strings into a byte stream to be piped to an *external, native executable*.4 It has absolutely no effect on the output of PowerShell's own cmdlets or file redirection operators (\>). For example, in the command Get-Process | findstr.exe "chrome", $OutputEncoding determines how the text data from Get-Process is encoded before being sent to the findstr.exe process. As noted, this defaults to ASCII in Windows PowerShell, a significant limitation, but was changed to UTF-8 in PowerShell 7\.15  
3. \*\*The Class:\*\* This mechanism operates at the lowest level, interacting directly with the console host (the terminal window itself). The.NET class exposes static properties, \[Console\]::InputEncoding and \[Console\]::OutputEncoding, which control how the.NET runtime interprets byte streams coming from standard input (e.g., keyboard) and how it encodes strings being sent to standard output.15 Changing these properties is the programmatic equivalent of using the chcp command to change the console's code page, but it is more reliable from within a PowerShell session because.NET can cache the console's encoding at startup.4 This is the critical layer that determines how characters are ultimately rendered in the terminal display. If \[Console\]::OutputEncoding is set to a legacy OEM code page, even correctly formed Unicode strings from within PowerShell will be corrupted upon being written to the console.

| Setting | Scope of Influence | Default (PS 5.1) | Default (PS 7+) |
| :---- | :---- | :---- | :---- |
| **$PSDefaultParameterValues\['\*:Encoding'\]** | Default \-Encoding parameter for PowerShell cmdlets (e.g., Out-File, Set-Content). | Varies by cmdlet (e.g., UTF-16LE for Out-File, ANSI for Set-Content). | UTF-8 (no BOM) |
| **$OutputEncoding** | Encoding of string data piped to *external native executables*. | ASCII | UTF-8 (no BOM) |
| **\[Console\]::OutputEncoding** | Encoding used by the.NET runtime to write strings to the console host (terminal display). | System OEM Code Page (e.g., 437\) | System OEM Code Page (e.g., 437\) |

The analysis of these three pillars reveals a crucial point: PowerShell 7's sensible defaults for its own operations (file I/O and inter-process communication) do not automatically extend to the console host environment itself. The \`\` layer remains tied to the operating system's legacy defaults. This disconnect is the central clue to understanding why a modern shell, running in a modern terminal editor, can still produce garbled output. The problem lies at the boundary where PowerShell hands off its data to the underlying console subsystem for display.

## **The VS Code Integrated Terminal: A Modern Façade on a Complex Foundation**

Visual Studio Code provides a highly integrated and customizable terminal experience, which has become the preferred environment for many PowerShell developers. However, this terminal is not a monolithic entity. It is a layered system comprising the editor's own settings, the terminal emulator, and the specific shell profile being run. The interaction between these layers directly influences how character encoding is handled and is a key factor in the problem being investigated.

### **VS Code's File Encoding Settings (settings.json)**

Visual Studio Code itself operates with a modern, cross-platform mindset. By default, it creates and saves all new files using UTF-8 without a BOM.3 This default is a sensible choice for web and cross-platform development, but as previously discussed, it is the direct cause of script interpretation errors when those files are run with Windows PowerShell 5.1.

VS Code provides robust mechanisms for controlling file encoding via its settings.json configuration file.21 Users can modify this behavior both globally and on a per-language basis.

* **Global Configuration:** The files.encoding setting can be changed to a new global default. For example, setting "files.encoding": "utf8bom" would make VS Code save all new files as UTF-8 with a BOM.3 While effective, this is a blunt instrument that may be undesirable for projects involving non-Windows platforms.  
* **Language-Specific Configuration:** A more precise and recommended approach is to use language-specific settings. This allows developers to enforce a particular encoding for PowerShell scripts without affecting other file types. The following configuration in settings.json ensures that all PowerShell files are saved as UTF-8 with a BOM, maximizing compatibility with both modern PowerShell and legacy Windows PowerShell environments 3:  
  JSON  
  "\[powershell\]": {  
      "files.encoding": "utf8bom",  
      "files.autoGuessEncoding": true  
  }

The files.autoGuessEncoding setting instructs VS Code to attempt to detect a file's encoding upon opening it, which can be helpful but is not a substitute for an explicit and consistent encoding strategy.3 These settings are crucial for controlling how script *files* are written to disk, but they do not directly control the runtime behavior of the terminal itself.

### **The Integrated Terminal vs. The PowerShell Integrated Console**

A critical distinction must be made within the VS Code environment. The term "integrated terminal" can refer to two different things, and their encoding behaviors differ significantly:

1. **The Generic Integrated Terminal:** When a user opens a new terminal (Ctrl+Shift+), VS Code launches the system's default shell (e.g., PowerShell 7, Command Prompt, or WSL Bash on Windows).26 This generic terminal session inherits its environment, including its active code page, from the operating system's defaults.\[27, 28\] On a standard Windows installation, this means a PowerShell 7 session started this way will likely run with the legacy OEM code page (e.g., 437\) active, as reported by chcp\`.4  
2. **The PowerShell Integrated Console:** When a user runs a PowerShell script using the PowerShell extension (e.g., by pressing F5), a special, customized terminal session is launched. This is not a generic terminal; it is the "PowerShell Integrated Console".19 A key behavior of the PowerShell extension is that it *proactively modifies the environment* of this console. Specifically, it programmatically sets the \[Console\]::OutputEncoding property to UTF-8.28

This intervention by the PowerShell extension is a deliberate attempt to create a more modern, UTF-8-friendly environment out of the box. It aims to solve the problem of garbled output from external, UTF-8-aware command-line tools. However, this helpful step is incomplete. The extension typically only sets the *output* encoding, leaving \[Console\]::InputEncoding untouched.28 This creates a hybrid, partially configured state that can lead to its own subtle and confusing behaviors. The environment expects to receive UTF-8 from external processes but may not be fully configured to handle all UTF-8 interactions correctly, and this partial configuration can be overridden by a user's own profile scripts, leading to unpredictable results.

### **Configuring Terminal Profiles and Startup Behavior**

VS Code's settings.json allows for detailed configuration of terminal profiles, enabling users to customize how different shells are launched.26 This feature can be leveraged to enforce specific encoding settings upon startup. For example, a profile could be created to launch PowerShell with arguments that execute a custom initialization script 26:

JSON

"terminal.integrated.profiles.windows": {  
    "PowerShell UTF-8": {  
        "path": "pwsh.exe",  
        "args": \[  
            "-noexit",  
            "-file",  
            "C:\\\\path\\\\to\\\\my\\\\utf8-init.ps1"  
        \]  
    }  
},  
"terminal.integrated.defaultProfile.windows": "PowerShell UTF-8"

This approach provides a powerful way to ensure that any terminal session, whether generic or for debugging, starts with a consistently configured encoding environment. While it is possible to use shell arguments to run commands like chcp 65001 directly 29, this method is less effective for PowerShell due to.NET's startup caching of the console encoding. Using a profile script that sets the \`\` properties is the more reliable method.

## **Diagnosis of the Mismatch: The Root Cause of Garbled Output**

Synthesizing the behaviors of the Windows ecosystem, the PowerShell engine, and the Visual Studio Code environment allows for a precise diagnosis of the character encoding problem. Despite PowerShell 7's sensible UTF-8 defaults, garbled output occurs because of a fundamental misalignment at a specific boundary: the interface between the.NET runtime and the underlying Windows console host. The VS Code terminal, though fully UTF-8 capable, is merely a passive display for an already corrupted data stream.

### **Tracing the Data Flow: From Script to Screen**

To pinpoint the failure, it is instructive to trace the lifecycle of a non-ASCII character from its definition in a script file to its final rendering on the screen.

1. **Step 1: The Script File.** A developer creates a PowerShell script (.ps1) in Visual Studio Code. The file contains a string with non-ASCII characters, for example, Write-Host "Résumé". By default, VS Code saves this file using UTF-8 encoding without a BOM.3  
2. **Step 2: Engine Interpretation.** The script is executed in a PowerShell 7 session within the VS Code integrated terminal. The PowerShell 7 engine, which defaults to assuming BOM-less files are UTF-8, correctly reads the bytes from the file and interprets them.1 The string "Résumé" is successfully created in memory. Within the.NET runtime, all strings are handled internally as a sequence of UTF-16 code units, ensuring that the character data is preserved accurately at this stage.1  
3. **Step 3: The Output Command.** The Write-Host "Résumé" command is executed. PowerShell's role is to take the correct, in-memory string and pass it to the appropriate subsystem for display. This involves invoking the underlying.NET console APIs.  
4. **Step 4: The Console Host Bottleneck.** This is the critical point of failure. To display the string in the console, the.NET runtime must convert the internal UTF-16 string into a stream of bytes that the console host can understand. The encoding used for this conversion is determined by the ::OutputEncoding property.3 Unless it has been explicitly changed, this property defaults to the system's legacy OEM code page (e.g., Code Page 437).28 The character "é" (Unicode code point U+00E9) does not exist in Code Page 437\. The.NET runtime, forced to encode the character using this incompatible code page, converts it into a fallback character, typically a question mark (?), or a byte sequence that is meaningless in a UTF-8 context. The Unicode data is now corrupted.  
5. **Step 5: Terminal Rendering.** The VS Code integrated terminal, which is a modern terminal emulator fully capable of rendering UTF-8 characters 30, receives this corrupted byte stream from the PowerShell process. It has no way of knowing that the original data was "Résumé"; it only sees the bytes representing "R?sum?". It then faithfully renders the corrupted string it was given.

This step-by-step analysis demonstrates that the problem is not with the file saving (VS Code), the script interpretation (PowerShell 7 engine), or the final display technology (VS Code terminal). The corruption occurs exclusively during the encoding translation performed by the.NET console API, which is governed by a legacy-bound default setting.

### **The Core Problem: A Misaligned Console Encoding**

The ultimate root cause of the issue is the persistent default to a legacy code page within the Windows console subsystem, a default that is not automatically overridden by PowerShell 7's modern internal settings. The Windows console environment is fundamentally "opt-in" for UTF-8.

PowerShell 7 has successfully opted in for its own file I/O operations and for its communication with external processes via $OutputEncoding. However, it does not, by default, force the console host itself to opt in. The final act of rendering text to the screen is delegated to the \`\`.NET class, which remains tethered to the system's default OEM code page unless it is explicitly reconfigured.

While the PowerShell extension for VS Code attempts to mitigate this by setting \[Console\]::OutputEncoding to UTF-8 for its PowerShell Integrated Console, this solution is not foolproof. It can be overridden by a user's own PowerShell profile script ($PROFILE), or it may be insufficient for complex scenarios involving mixed-encoding inputs and outputs.18 The core of the problem remains: there is a misalignment between the encoding used within the PowerShell process and the encoding that the process assumes is expected by the terminal it is writing to.

## **A Strategic Framework for Resolution: From Tactical Fixes to Systemic Harmony**

Resolving the UTF-8 output issue requires a deliberate and correctly targeted approach. A durable solution involves aligning the encoding expectations across all layers of the stack: the file system, the PowerShell runtime environment, and the console host. The following strategies are presented in order of increasing scope and permanence, from simple file-level workarounds to a comprehensive reconfiguration of the development environment.

### **File-Level Remediation: The UTF-8 with BOM Workaround**

This is the most direct, tactical fix, primarily aimed at ensuring script files are interpreted correctly, especially in environments that might also use the legacy Windows PowerShell.

* **Action:** In Visual Studio Code, with the PowerShell script open, click on the encoding label (e.g., "UTF-8") in the bottom status bar. From the menu that appears, select "Save with Encoding," and then choose "UTF-8 with BOM".5  
* **Mechanism:** This action prepends the three-byte UTF-8 Byte Order Mark (0xEF, 0xBB, 0xBF) to the start of the file. This BOM serves as an unambiguous signal to any application, including Windows PowerShell 5.1, that the file's content is UTF-8 encoded, overriding any default assumption of a legacy ANSI code page.2  
* **Pros:** It is simple to perform and highly effective for ensuring script *portability* between PowerShell 7 and older Windows PowerShell environments.  
* **Cons:** This method only solves the problem of the PowerShell engine *reading* the script file correctly. It does nothing to address the root cause of garbled *output* to the console, which is a runtime configuration issue. Furthermore, the presence of a BOM can cause compatibility problems with tools and platforms in the broader Unix/Linux ecosystem that do not expect it.1

### **Environment Configuration (VS Code settings.json)**

This approach establishes a "best practice" policy for file encodings within the development editor, making the use of a compatible encoding the default behavior for PowerShell projects.

* **Action:** Open the VS Code settings file (settings.json) and add a language-specific configuration for PowerShell. This ensures that all new PowerShell files are created with the desired encoding and that VS Code will attempt to interpret existing files correctly.  
  JSON  
  "\[powershell\]": {  
      "files.encoding": "utf8bom",  
      "files.autoGuessEncoding": true  
  }

* **Mechanism:** This setting instructs VS Code to default to saving files with the .ps1 extension as UTF-8 with a BOM.3 This automates the file-level remediation described above, creating a set of "guard rails" for development.  
* **Pros:** Enforces a consistent and safe encoding standard for PowerShell projects within a team, reducing the likelihood of script interpretation errors.  
* **Cons:** Like the manual file-level fix, this only addresses the encoding of the script file on disk. It does not configure the runtime environment of the terminal and therefore does not solve the console output problem. Its effects are limited to files edited within VS Code.

### **The Definitive Solution: Harmonizing the PowerShell Profile ($PROFILE)**

This is the most robust and highly recommended solution for developers, as it directly addresses the root cause of the problem by configuring the PowerShell runtime environment itself for every session.

* **Action:** Edit the current user's PowerShell profile script. This can be easily opened in VS Code by running the command code $PROFILE from within a PowerShell terminal. Add the following code block to the profile file.  
  PowerShell  
  \#  
  \# Comprehensive UTF-8 Environment Configuration for PowerShell  
  \#  
  \# This block aligns the console host's encoding with PowerShell's modern defaults,  
  \# ensuring correct handling of Unicode characters for display, inter-process  
  \# communication, and default file output.  
  \#

  \# Set the console's input and output encodings to UTF-8.  
  \# This ensures the terminal correctly displays Unicode characters sent by PowerShell  
  \# and correctly interprets Unicode characters received from external sources.

::InputEncoding \=::UTF8  
::OutputEncoding \=::UTF8

\# Set the encoding for piping data to native/external executables to UTF-8.  
$OutputEncoding \=::UTF8

\# Set the default encoding for cmdlets with an \-Encoding parameter (e.g., Out-File)  
\# to BOM-less UTF-8. This reinforces PowerShell 7's default behavior.  
$PSDefaultParameterValues\['\*:Encoding'\] \= 'utf8'  
\`\`\`

* **Mechanism:** This script block comprehensively configures all three pillars of PowerShell encoding for every new session. It sets the.NET console APIs to expect UTF-8, aligns the encoding for external process communication, and reinforces the UTF-8 default for file-writing cmdlets.4 This harmonizes the entire environment, ensuring that what PowerShell 7 handles internally as UTF-8 is also treated as UTF-8 at every boundary—console display, external pipes, and file I/O.  
* **Pros:** This is a complete and persistent solution that fixes the root cause of the problem. It is user-specific and portable, as the profile script can be managed as part of a developer's dotfiles.  
* **Cons:** It requires a basic understanding of the PowerShell profile mechanism. The settings are applied per-user and must be configured on each development machine.

### **System-Wide Unification: The Windows UTF-8 Beta Feature**

This is the most far-reaching solution, altering the default behavior of the entire Windows operating system. It should be approached with caution.

* **Action:** Navigate to the Windows Region settings (can be opened by running intl.cpl). Go to the "Administrative" tab, click "Change system locale...", and check the box labeled "Beta: Use Unicode UTF-8 for worldwide language support." A system restart is required.4  
* **Mechanism:** This experimental feature changes the system's default ANSI and OEM code pages to 65001 (UTF-8). This makes UTF-8 the default assumption for all console applications and even for many legacy non-Unicode GUI applications across the entire operating system.  
* **Pros:** It provides a true system-wide fix that eliminates the need for per-application or per-shell configuration. It creates a Windows environment that behaves more like Linux or macOS with respect to default encoding.  
* **Cons:** This is a significant and potentially breaking change. Older applications that are not Unicode-aware and rely on a specific legacy code page for their functionality may fail or produce incorrect data.4 It is designated as a "Beta" feature and should only be enabled by advanced users on dedicated development machines who understand and accept the potential risks of system-wide incompatibility.

## **Executive Summary and Recommendations**

The investigation into garbled UTF-8 output from PowerShell 7 in the Visual Studio Code terminal reveals a complex interaction between modern development tools and legacy operating system architecture. The problem is not a flaw in either PowerShell 7 or VS Code but is instead a consequence of a misaligned encoding configuration at the boundary between the.NET runtime and the Windows console host.

### **Recapitulation of the Core Problem**

PowerShell 7 has commendably standardized on BOM-less UTF-8 for its internal operations, including script interpretation, file I/O, and communication with external programs. Visual Studio Code similarly defaults to BOM-less UTF-8 for file creation. The conflict arises because the underlying.NET console APIs, which PowerShell uses to render text to the terminal, do not automatically inherit these modern defaults. Instead, they remain bound to the system's legacy OEM code page. When a Unicode string from PowerShell is passed to these APIs for display, it is incorrectly transcoded using the legacy code page, resulting in data corruption before it ever reaches the UTF-8-capable VS Code terminal. The terminal then correctly renders the garbled byte stream it receives.

### **Prioritized Action Plan**

Based on a thorough analysis of the issue and the available solutions, the following prioritized action plan is recommended to achieve consistent, correct UTF-8 behavior.

* Recommendation 1 (Most Users): Implement the PowerShell Profile ($PROFILE) Solution.  
  This is the most balanced, powerful, and non-disruptive method for creating a reliable UTF-8 development environment. By adding the comprehensive configuration block from Section 5.3 to the $PROFILE script, a user ensures that every PowerShell session—whether in VS Code, Windows Terminal, or another console—is fully aligned for UTF-8 handling across console display, external process communication, and file output. This approach fixes the root cause at the runtime level without making risky system-wide changes.  
* Recommendation 2 (For Mixed PowerShell 5.1/7.0 Environments): Configure VS Code settings.json in Addition to the Profile.  
  For developers who must ensure their scripts are backward-compatible with Windows PowerShell 5.1, it is advisable to supplement the profile solution by configuring VS Code's settings.json to save PowerShell files as UTF-8 with BOM, as detailed in Section 5.2. This provides the necessary BOM for legacy environments to correctly interpret the script files, while the profile configuration ensures correct runtime behavior in modern PowerShell.  
* Recommendation 3 (Advanced Users on Dedicated Systems Only): Consider the Windows Beta UTF-8 Feature.  
  For advanced users working on a dedicated development machine, enabling the system-wide "Use Unicode UTF-8 for worldwide language support" feature described in Section 5.4 can provide the most seamless experience. This aligns the entire operating system with modern encoding standards. However, this action should only be taken after a careful consideration of the potential for breaking legacy, non-Unicode-aware applications.

### **Final Best Practices**

To prevent encoding issues in the future, developers should adopt the following best practices:

* **Be Explicit:** Whenever a PowerShell cmdlet supports an \-Encoding parameter, be explicit (e.g., Out-File \-Path.\\data.txt \-Encoding utf8) to remove ambiguity, especially in scripts intended for distribution.  
* **Understand Your Environment:** Recognize the distinction between the generic VS Code integrated terminal and the specialized PowerShell Integrated Console, and be aware of how profile scripts and extensions can alter their behavior.  
* **Prioritize Portability:** When writing scripts that will be shared or run in unknown environments, either restrict string content to the 7-bit ASCII character set or save files as UTF-8 with BOM to ensure the widest possible compatibility.

#### **Works cited**

1. about\_Character\_Encoding \- PowerShell | Microsoft Learn, accessed October 30, 2025, [https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about\_character\_encoding?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_character_encoding?view=powershell-7.5)  
2. PowerShell file encoding \- ScriptRunner Documentation, accessed October 30, 2025, [https://support.scriptrunner.com/articles/coding-gen7/file-encoding](https://support.scriptrunner.com/articles/coding-gen7/file-encoding)  
3. Understanding file encoding in VS Code and PowerShell ..., accessed October 30, 2025, [https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/vscode/understanding-file-encoding?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/vscode/understanding-file-encoding?view=powershell-7.5)  
4. Using UTF-8 Encoding (CHCP 65001\) in Command Prompt ..., accessed October 30, 2025, [https://stackoverflow.com/questions/57131654/using-utf-8-encoding-chcp-65001-in-command-prompt-windows-powershell-window](https://stackoverflow.com/questions/57131654/using-utf-8-encoding-chcp-65001-in-command-prompt-windows-powershell-window)  
5. UTF8 Script in PowerShell outputs incorrect characters \- Stack Overflow, accessed October 30, 2025, [https://stackoverflow.com/questions/14482253/utf8-script-in-powershell-outputs-incorrect-characters](https://stackoverflow.com/questions/14482253/utf8-script-in-powershell-outputs-incorrect-characters)  
6. Encoding issue with VS Code and Powershell \- Reddit, accessed October 30, 2025, [https://www.reddit.com/r/PowerShell/comments/hokqyv/encoding\_issue\_with\_vs\_code\_and\_powershell/](https://www.reddit.com/r/PowerShell/comments/hokqyv/encoding_issue_with_vs_code_and_powershell/)  
7. Understanding Character Encoding in PowerShell \- Petri IT Knowledgebase, accessed October 30, 2025, [https://petri.com/understanding-character-encoding-in-powershell/](https://petri.com/understanding-character-encoding-in-powershell/)  
8. Change OutputEncoding and code page \- Super User, accessed October 30, 2025, [https://superuser.com/questions/1558443/change-outputencoding-and-code-page](https://superuser.com/questions/1558443/change-outputencoding-and-code-page)  
9. Changing PowerShell's default output encoding to UTF-8 \- Stack Overflow, accessed October 30, 2025, [https://stackoverflow.com/questions/40098771/changing-powershells-default-output-encoding-to-utf-8](https://stackoverflow.com/questions/40098771/changing-powershells-default-output-encoding-to-utf-8)  
10. Need the get-content command output in UTF8 no bom \- PowerShell Forums, accessed October 30, 2025, [https://forums.powershell.org/t/need-the-get-content-command-output-in-utf8-no-bom/23718](https://forums.powershell.org/t/need-the-get-content-command-output-in-utf8-no-bom/23718)  
11. Debugger Does Not Handle Unicode/UTF-8 Characters Properly \#1392 \- GitHub, accessed October 30, 2025, [https://github.com/PowerShell/vscode-powershell/issues/1392](https://github.com/PowerShell/vscode-powershell/issues/1392)  
12. PowerShell file encoding \- ScriptRunner Documentation, accessed October 30, 2025, [https://support.scriptrunner.com/articles/\#\!coding/file-encoding](https://support.scriptrunner.com/articles/#!coding/file-encoding)  
13. Powershell Terminal encoding in VSCode \- Stack Overflow, accessed October 30, 2025, [https://stackoverflow.com/questions/53848860/powershell-terminal-encoding-in-vscode](https://stackoverflow.com/questions/53848860/powershell-terminal-encoding-in-vscode)  
14. Default PowerShell to emitting UTF-8 instead of UTF-16? \- Super User, accessed October 30, 2025, [https://superuser.com/questions/327492/default-powershell-to-emitting-utf-8-instead-of-utf-16](https://superuser.com/questions/327492/default-powershell-to-emitting-utf-8-instead-of-utf-16)  
15. Make console windows fully UTF-8 by default on Windows, in line with the behavior on Unix-like platforms \- character encoding, code page · Issue \#7233 · PowerShell/PowerShell \- GitHub, accessed October 30, 2025, [https://github.com/PowerShell/PowerShell/issues/7233](https://github.com/PowerShell/PowerShell/issues/7233)  
16. Character encoding (UTF-8) in PowerShell session \[duplicate\] \- Stack Overflow, accessed October 30, 2025, [https://stackoverflow.com/questions/51933189/character-encoding-utf-8-in-powershell-session](https://stackoverflow.com/questions/51933189/character-encoding-utf-8-in-powershell-session)  
17. Handling Native EXE Output Encoding in UTF8 with No BOM | Keith Hill's Blog, accessed October 30, 2025, [https://rkeithhill.wordpress.com/2010/05/26/handling-native-exe-output-encoding-in-utf8-with-no-bom/](https://rkeithhill.wordpress.com/2010/05/26/handling-native-exe-output-encoding-in-utf8-with-no-bom/)  
18. Why does "Out-File" seem to ignore the "Encoding" parameter? \- Super User, accessed October 30, 2025, [https://superuser.com/questions/1721178/why-does-out-file-seem-to-ignore-the-encoding-parameter](https://superuser.com/questions/1721178/why-does-out-file-seem-to-ignore-the-encoding-parameter)  
19. Weird encoding issues in VS Code. \- PowerShell \- Reddit, accessed October 30, 2025, [https://www.reddit.com/r/PowerShell/comments/y3rnab/weird\_encoding\_issues\_in\_vs\_code/](https://www.reddit.com/r/PowerShell/comments/y3rnab/weird_encoding_issues_in_vs_code/)  
20. PS7 doesn't display utf-8 characters : r/PowerShell \- Reddit, accessed October 30, 2025, [https://www.reddit.com/r/PowerShell/comments/lvn84x/ps7\_doesnt\_display\_utf8\_characters/](https://www.reddit.com/r/PowerShell/comments/lvn84x/ps7_doesnt_display_utf8_characters/)  
21. User and workspace settings \- Visual Studio Code, accessed October 30, 2025, [https://code.visualstudio.com/docs/configure/settings](https://code.visualstudio.com/docs/configure/settings)  
22. Change file encoding in VS Code \- DeveloperF1.com, accessed October 30, 2025, [https://developerf1.com/snippet/change-file-encoding-in-vs-code](https://developerf1.com/snippet/change-file-encoding-in-vs-code)  
23. Change the encoding of a file in Visual Studio Code \- Stack Overflow, accessed October 30, 2025, [https://stackoverflow.com/questions/30082741/change-the-encoding-of-a-file-in-visual-studio-code](https://stackoverflow.com/questions/30082741/change-the-encoding-of-a-file-in-visual-studio-code)  
24. Using Visual Studio Code for PowerShell Development \- Microsoft Learn, accessed October 30, 2025, [https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/vscode/using-vscode?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/vscode/using-vscode?view=powershell-7.5)  
25. How to change the File Encoding in Visual Studio Code | bobbyhadz, accessed October 30, 2025, [https://bobbyhadz.com/blog/vscode-change-file-encoding](https://bobbyhadz.com/blog/vscode-change-file-encoding)  
26. Terminal Profiles \- Visual Studio Code, accessed October 30, 2025, [https://code.visualstudio.com/docs/terminal/profiles](https://code.visualstudio.com/docs/terminal/profiles)  
27. powershell \- Different CLI InputEncoding between Console and VS ..., accessed October 30, 2025, [https://stackoverflow.com/questions/70476030/different-cli-inputencoding-between-console-and-vs-code-console](https://stackoverflow.com/questions/70476030/different-cli-inputencoding-between-console-and-vs-code-console)  
28. How to set Integrated Terminal default encoding to UTF-8? · Issue \#19837 · microsoft/vscode, accessed October 30, 2025, [https://github.com/microsoft/vscode/issues/19837](https://github.com/microsoft/vscode/issues/19837)  
29. Visual Studio Code on Windows, accessed October 30, 2025, [https://code.visualstudio.com/docs/setup/windows](https://code.visualstudio.com/docs/setup/windows)  
30. Terminal Advanced \- Visual Studio Code, accessed October 30, 2025, [https://code.visualstudio.com/docs/terminal/advanced](https://code.visualstudio.com/docs/terminal/advanced)  
31. Fix Windows Emoji & UTF-8 Encoding Issues for VS Code, PowerShell, Git Bash & Claude Code \- GitHub Gist, accessed October 30, 2025, [https://gist.github.com/killerapp/6c53b69639ceef5139b276995879a6ac](https://gist.github.com/killerapp/6c53b69639ceef5139b276995879a6ac)  
32. Change default code page of Windows console to UTF-8 \- Super User, accessed October 30, 2025, [https://superuser.com/questions/269818/change-default-code-page-of-windows-console-to-utf-8](https://superuser.com/questions/269818/change-default-code-page-of-windows-console-to-utf-8)