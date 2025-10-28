
# **A Comprehensive Guide to Building and Installing the Harbour Compiler on Windows**

## **I. Introduction: The Principle of Separation in Software Compilation**

In modern software engineering, a foundational principle is the strict separation of source code from build artifacts. Maintaining a pristine source code tree—one that contains only the human-written files necessary to define the project—is paramount for maintainability, collaboration, and reproducibility. When the compilation process generates object files, libraries, and executables directly within the source directories, it introduces several challenges: it complicates version control by mixing tracked source with untracked binaries, makes it difficult to manage multiple build configurations (e.g., a debug build and a release build) from the same source, and clutters the workspace, making it harder for developers to navigate.

The situation described—compiling the Harbour core repository on Windows and finding the source directory intermingled with the output—is a direct consequence of an "in-source" build. This is a common default behavior for many build systems. The solution is to perform an "out-of-source" build, where all generated files are directed to a separate, dedicated directory. This approach not only resolves the aforementioned issues but also streamlines the process of creating clean, distributable software packages.

This report provides a definitive, step-by-step guide to mastering the Harbour build system on Windows. It begins with the immediate solution to redirecting build output, then delves into the architecture of the build system itself. It offers a comprehensive reference for the key environment variables that control compilation, provides a prescriptive walkthrough for the specific goal of building with the Zig compiler, and concludes with post-installation verification and advanced configuration techniques. For clarity, this document pertains to the Harbour programming language, a superset of Clipper, and should not be confused with Harbor, the open-source container image registry.1

## **II. The Immediate Solution: Building Harbour into a Dedicated Directory**

The Harbour build system is powerful and flexible, designed to be configured externally without modifying its source files. Achieving a clean installation into a separate directory requires instructing the build system on two key points: the destination for the final files and the action to perform the installation. This is accomplished by setting a specific environment variable and invoking the correct build target.

### **Defining the Destination with HB\_INSTALL\_PREFIX**

The primary mechanism for specifying the installation directory is the HB\_INSTALL\_PREFIX environment variable. This variable informs the makefile's installation logic of the root directory where all compiled binaries, libraries, header files, and documentation should be placed. Its use is a standard practice across various build configurations, whether for native Windows builds with MinGW or cross-compiling for platforms like Android.2

For the scenario of installing to a directory named dist within a D:\\HarbourInstall folder, the command in a Windows Command Prompt would be:

Code snippet

set HB\_INSTALL\_PREFIX=D:\\HarbourInstall\\dist

It is crucial to provide an absolute path to avoid ambiguity during the build process.4

### **Executing the Installation with the 'install' Target**

Simply setting the HB\_INSTALL\_PREFIX variable is insufficient on its own. It is a piece of configuration that must be acted upon. The user's original command, win-make all, instructs the build system only to compile all necessary components, which it does within the source tree by default.

The correct action is to use the install target. This target is specifically designed to trigger the deployment phase of the build. It typically depends on the all target, meaning it will first compile everything required, and then it will execute the sequence of commands responsible for copying the resulting files from their temporary build locations to the structured directory tree under HB\_INSTALL\_PREFIX.3 The command to invoke this is:

Code snippet

win-make install

The relationship between the variable and the target is the core of the solution. The install target's rules are written to read the value of HB\_INSTALL\_PREFIX from the environment and use it as the destination path. Without the install target, this variable is effectively ignored.

### **The Complete Command Sequence**

Combining these two elements provides the complete and correct command sequence to achieve the user's goal. From within the cloned Harbour source directory (D:\\HarbourCompile\\core), the following commands will compile Harbour using the Zig toolchain and place all final artifacts into D:\\HarbourInstall\\dist, leaving the source directory clean.

Code snippet

cd D:\\HarbourCompile\\core  
set PATH=C:\\zig;%PATH%  
set HB\_COMPILER=zig  
set HB\_INSTALL\_PREFIX=D:\\HarbourInstall\\dist  
win-make \-j8 install

This sequence first sets up the environment by prioritizing the Zig compiler in the PATH and explicitly telling Harbour to use it via HB\_COMPILER. It then defines the installation destination. Finally, it invokes win-make with the \-j8 flag to parallelize the compilation across eight CPU cores for speed, and crucially, uses the install target to ensure the compiled files are moved to the specified prefix.

## **III. A Deeper Dive into the Harbour Build System**

To move from simply executing commands to truly understanding the build process, it is essential to grasp the concepts of make targets and the configuration philosophy that underpins the Harbour project. This knowledge empowers developers to adapt the build to various needs beyond the immediate problem.

### **Understanding Make Targets: all vs. install vs. clean**

The win-make.exe utility provided with Harbour is a Windows port of GNU Make, a tool that automates the build process by executing rules defined in a makefile.6 These rules are grouped into "targets," which represent specific actions or states.

* **all**: This is typically the default target in a makefile. Its purpose is to perform all the necessary steps to compile the source code and generate the final products, such as executables and libraries. However, by convention, it places these artifacts within the source or a temporary build subdirectory. Executing win-make all successfully confirms that the toolchain is configured correctly and the source code is valid, but it does not perform any deployment. This is precisely the behavior the user initially encountered.  
* **install**: This target is designed for deployment. Its primary responsibility is to copy the build artifacts—which are either pre-existing or generated by its dependency on the all target—to a final destination outside the source tree. This destination is almost universally controlled by a prefix variable like HB\_INSTALL\_PREFIX.2 Using the install target is the standard and intended method for creating a clean, usable installation of the software.  
* **clean**: This is a housekeeping target. Its function is to remove all files generated during the compilation process from the source tree. This includes object files (.o), libraries (.lib, .a), and executables (.exe). Running win-make clean is a critical step when needing to force a complete rebuild from scratch, ensuring that no stale or improperly configured artifacts from a previous build attempt interfere with a new one.

### **The Build Philosophy: Configuration Over Modification**

The Harbour build system is a prime example of a robust, cross-platform design that favors external configuration over internal modification. An examination of its usage across different platforms and compilers reveals a consistent pattern: the build is controlled almost entirely through environment variables.2

This design choice is deliberate and highly advantageous. By reading its configuration from the environment, the build system allows a single, unmodified set of makefiles to produce a vast array of different builds. A developer can switch C compilers, target a different operating system, enable or disable features, or change installation paths simply by setting the appropriate variables before running win-make. This avoids the fragile and error-prone practice of editing the project's makefiles, which would create local modifications that conflict with updates from the source repository.

The extensive lists of HB\_\* variables documented for the project serve as a testament to this philosophy.4 They provide hooks to control nearly every facet of the compilation and installation process, making the system exceptionally flexible. The correct way to interact with the Harbour build system is to treat the makefiles as a fixed engine and the environment variables as the controls that steer it.

## **IV. Mastering Build Configuration: A Guide to Harbour Environment Variables**

The flexibility of the Harbour build system is unlocked by understanding and utilizing its rich set of environment variables. These variables are the primary interface for customizing the build process.

### **Installation Path Directives**

While HB\_INSTALL\_PREFIX is the master variable for setting the installation root, more granular control is available.

* **HB\_INSTALL\_PREFIX**: As established, this variable sets the top-level directory for the installation. It is the most common and essential variable for achieving an out-of-source installation.2 However, it is important to be aware of the final directory structure. Some build configurations may create an additional harbour subdirectory within the specified prefix, which can be an unexpected behavior.11 For example, setting HB\_INSTALL\_PREFIX=D:\\Harbour might result in binaries being placed in D:\\Harbour\\harbour\\bin. This detail is critical when later configuring the system PATH.  
* **Granular Path Control (HB\_INSTALL\_BIN, HB\_INSTALL\_LIB, etc.)**: For advanced users who require full control over the final layout, the build system provides variables to override the default subdirectories. Variables such as HB\_INSTALL\_BIN (for executables), HB\_INSTALL\_LIB (for libraries), and HB\_INSTALL\_INC (for header files) can be set to explicit paths, allowing one to bypass the default structure entirely.12 This can be used to create a "flat" installation directory or to conform to specific packaging standards.

### **Compiler and Platform Selection**

These variables are fundamental for defining the build environment and target.

* **HB\_COMPILER**: This variable explicitly tells the build system which C compiler toolchain to use. While the system can often auto-detect a compiler, setting this variable eliminates ambiguity, especially when multiple compilers are present. Valid values include zig (as in the user's case), mingw, msvc (for Microsoft Visual C++), and clang.2  
* **HB\_PLATFORM**: This variable specifies the target operating system and architecture. For a native build on Windows, it will typically auto-detect to win. However, it is essential for cross-compilation. For instance, when building for Android, it must be set to android.3 Similarly, when cross-compiling for Windows from a Linux host, it would be set to win.9

### **Fine-Tuning Build Artifacts (HB\_BUILD\_\* Flags)**

A large family of variables prefixed with HB\_BUILD\_ allows for fine-grained control over the nature of the compiled artifacts. The most common and useful of these are detailed in the project's documentation.4

* **HB\_BUILD\_VERBOSE=yes**: Enables detailed, verbose output from the build system, showing the exact compiler and linker commands being executed. This is invaluable for debugging the build process itself.  
* **HB\_BUILD\_DEBUG=yes**: Instructs the C compiler to include debugging symbols in the generated binaries. This is essential for debugging the Harbour runtime or any custom C-level code with a debugger like GDB.  
* **HB\_BUILD\_STRIP=\[all|bin|lib|no\]**: Controls the stripping of symbols from the final binaries. Stripping symbols reduces the file size, which is desirable for release builds, but makes debugging impossible. The default is no.  
* **HB\_BUILD\_OPTIM=no**: Disables C compiler optimizations. While optimizations are crucial for performance in release builds, disabling them can sometimes make debugging easier, as the generated machine code will more closely follow the source code logic.  
* **HB\_BUILD\_SHARED=yes**: Governs whether Harbour is built as a set of shared libraries (.dll on Windows) rather than being linked statically into each executable.  
* **HB\_BUILD\_CONTRIBS and HB\_BUILD\_3RDEXT**: These variables manage the compilation of optional "contrib" libraries and the auto-detection of third-party dependencies, allowing for a customized build with only the required components.

### **Table 1: Key Harbour Build & Installation Environment Variables**

The following table consolidates the most critical environment variables into a quick-reference guide for configuring the Harbour build process on Windows.

| Variable | Purpose | Common Values | Example Usage |
| :---- | :---- | :---- | :---- |
| HB\_INSTALL\_PREFIX | Sets the root directory for the install target. | Any valid absolute path | set HB\_INSTALL\_PREFIX=C:\\hb32-zig |
| HB\_COMPILER | Specifies the C compiler toolchain to use. | zig, mingw, msvc, clang | set HB\_COMPILER=zig |
| HB\_PLATFORM | Specifies the target platform (often auto-detected). | win, linux, dos, android | set HB\_PLATFORM=win |
| HB\_BUILD\_DEBUG | Includes C-level debug information in binaries. | yes, no (default) | set HB\_BUILD\_DEBUG=yes |
| HB\_BUILD\_STRIP | Strips symbols from binaries to reduce size. | all, bin, lib, no (default) | set HB\_BUILD\_STRIP=all |
| HB\_BUILD\_VERBOSE | Enables detailed output during the build process. | yes, no (default) | set HB\_BUILD\_VERBOSE=yes |
| HB\_BUILD\_NAME | Creates a named sub-build to isolate configurations. | Any string (e.g., "debug") | set HB\_BUILD\_NAME=debug\_build |

## **V. Prescriptive Walkthrough: Building Harbour with Zig on Windows x64**

This section provides a complete, sequential walkthrough that synthesizes the preceding concepts into a practical guide for building Harbour from source with the Zig compiler and installing it to a clean, separate directory.

### **1\. Environment Preparation**

Before beginning, ensure the necessary tools are installed and accessible.

* **Git for Windows**: Verify that git is installed and available in the system PATH.  
* **Zig Compiler**: Verify that Zig is installed (e.g., in C:\\zig) and that the path is known.  
* **Command Prompt**: Open a new, clean Command Prompt instance (cmd.exe) to ensure there are no conflicting environment variables from previous sessions.

### **2\. Cloning the Source Code**

Create a directory for the compilation process and clone the official Harbour core repository into it.

Code snippet

mkdir D:\\HarbourCompile  
cd D:\\HarbourCompile  
git clone https://github.com/harbour/core.git  
cd core

### **3\. Cleaning Previous Builds (Best Practice)**

To ensure a completely fresh build, especially after a previous attempt that resulted in an in-source build, it is highly recommended to run the clean target first. This will remove all previously generated files.

Code snippet

win-make clean

### **4\. Configuring the Build Environment**

In the Command Prompt window, execute the following set commands to configure the environment for the desired build. These settings will only apply to the current console session.

Code snippet

rem Prioritize the Zig compiler in the PATH for this session  
set PATH=C:\\zig;%PATH%

rem Explicitly select the Zig compiler  
set HB\_COMPILER=zig

rem Define the target installation directory  
set HB\_INSTALL\_PREFIX=D:\\harbour-zig-build

### **5\. Executing the Build and Install**

With the environment configured, execute the build using the install target. The \-j8 flag is recommended for multi-core systems to significantly reduce compilation time.

Code snippet

win-make \-j8 install

The build process will now commence. It will compile all Harbour components and, upon successful compilation, will create the D:\\harbour-zig-build directory and populate it with the final binaries, libraries, and headers. The original source directory at D:\\HarbourCompile\\core will remain free of build artifacts.

### **6\. Anatomy of the Destination Directory**

After the process completes, inspect the contents of the installation directory (D:\\harbour-zig-build). The structure should resemble the following:

* bin\\: Contains the Harbour compiler (harbour.exe), the build utility (hbmk2.exe), and other command-line tools.  
* lib\\: Contains the static (.lib) and potentially import (.lib for .dll) libraries.  
* include\\: Contains the C header files (.h) needed for developing C-level extensions for Harbour.  
* doc\\: Contains documentation files.

Be mindful that, depending on the exact version and build script logic, the installation might be nested inside an additional harbour subdirectory (e.g., D:\\harbour-zig-build\\harbour\\bin).11 This is a crucial detail for the final configuration step.

## **VI. Post-Installation: Verification and Environment Configuration**

Building the Harbour compiler is the first major step. The second is configuring the system to use the newly built toolchain to compile Harbour applications. This involves a critical conceptual distinction and a final environment setup.

### **The Dichotomy of Build Systems: win-make vs. hbmk2**

A common point of confusion for new Harbour developers is the distinction between the two primary build tools.

* **win-make**: This tool is used for the "meta-build"—the process of building the Harbour compiler *itself* from its C source code. It is typically used only once after cloning or updating the Harbour source repository.  
* **hbmk2**: This is the Harbour Make tool, and it is the primary utility used in day-to-day development to compile Harbour application code (.prg files) into final executables.7

The workflow is sequential: first, use win-make to build and install the Harbour toolchain. Then, add the resulting toolchain to the system PATH and use hbmk2 from that point forward to build actual applications.

### **Configuring the PATH for Usage**

To make the newly compiled Harbour tools available system-wide, the bin directory of the installation must be added to the Windows PATH environment variable.

1. Identify the correct bin directory path (e.g., D:\\harbour-zig-build\\bin).  
2. Add this path to the User or System PATH variable through the "Edit the system environment variables" control panel.  
3. Alternatively, for a permanent user PATH modification from the command line, use the setx command. Note that this will only affect new Command Prompt windows.  
   Code snippet  
   setx PATH "%PATH%;D:\\harbour-zig-build\\bin"

### **Verification: Compiling a "Hello, World\!" Application**

The final step is to verify that the entire toolchain is functional.

1. Open a **new** Command Prompt window to ensure the updated PATH is loaded.  
2. Create a simple test application file named hello.prg with the following content 7:  
   Code snippet  
   FUNCTION Main()  
     ? "Hello, world\!"  
      RETURN NIL

3. Compile the application using hbmk2:  
   Code snippet  
   hbmk2 hello.prg

4. If the compilation is successful, hbmk2 will create hello.exe in the same directory.  
5. Run the executable:  
   Code snippet  
   hello.exe

The output Hello, world\! on the console confirms that the Harbour compiler was built successfully with Zig, installed correctly, and is now fully operational for application development.

## **VII. Advanced Topics and Troubleshooting**

This section covers advanced techniques for managing complex build environments and provides solutions for common issues that may arise during the build process.

### **Managing Multiple Parallel Builds**

For developers who need to maintain multiple distinct Harbour builds—for example, a 64-bit release build with Zig, a 32-bit debug build with MinGW, and a build for a different platform—the HB\_BUILD\_NAME variable is an invaluable tool.10

When this variable is set, the build system creates all its temporary and final output within subdirectories named after the value provided. This allows multiple configurations to coexist within the same source tree without interfering with one another.

**Example:**

Code snippet

rem Configure and build a 64-bit release version  
set HB\_COMPILER=zig  
set HB\_BUILD\_NAME=zig\_release  
win-make \-j8 install

rem Clean and configure for a 32-bit debug version  
win-make clean  
set HB\_COMPILER=mingw32  
set HB\_BUILD\_DEBUG=yes  
set HB\_BUILD\_NAME=mingw\_debug  
win-make \-j8 install

This would result in two separate, isolated build outputs, allowing the developer to switch between them as needed.

### **Common Pitfalls and Error Resolution**

* **\! HB\_COMPILER not set, could not autodetect**: This error, reported by the build system, indicates that it could not find a suitable C compiler in the PATH and that the HB\_COMPILER variable was not set to resolve the ambiguity.7 The solution is to either fix the PATH to include a supported C compiler's bin directory or to explicitly set HB\_COMPILER to the desired toolchain (e.g., set HB\_COMPILER=mingw).  
* **Path Formatting Issues**: The HB\_INSTALL\_PREFIX variable is sensitive to formatting. On Windows, it should always be an absolute path. Avoid using quotes around the path in a standard cmd.exe session and do not include a trailing backslash, as these can confuse the makefile scripts.4  
* **Toolchain Conflicts**: Having the bin directories of multiple C compilers (e.g., MinGW, MSVC, Clang) in the system PATH simultaneously can lead to unpredictable behavior. The build system's auto-detection may pick the wrong one, or worse, it may mix tools from different toolchains, leading to cryptic linker errors. It is a best practice to ensure only one C compiler toolchain is active in the PATH of the build shell.4  
* **'win-make' is not recognized...**: This error indicates that the win-make.exe utility is not in the current directory. The Harbour build commands must be run from the root of the cloned Harbour source repository, where win-make.exe is located.

#### **Works cited**

1. Harbor Installation and Configuration, accessed October 25, 2025, [https://goharbor.io/docs/1.10/install-config/](https://goharbor.io/docs/1.10/install-config/)  
2. marcosgambeta/harbourpp-v1: Harbour++ v1 \- GitHub, accessed October 25, 2025, [https://github.com/marcosgambeta/harbourpp-v1](https://github.com/marcosgambeta/harbourpp-v1)  
3. How to build HARBOUR to compile PRG for Android in console mode (Windows version), accessed October 25, 2025, [http://www.elektrosoft.it/tutorials/harbour-android-windows-console/how-to-create-application-for-android-in-harbour-windows.asp](http://www.elektrosoft.it/tutorials/harbour-android-windows-console/how-to-create-application-for-android-in-harbour-windows.asp)  
4. vszakats/hb: Harbour fork (from https://github.com/harbour/core) \+ updates & fixes \= 3.4 \- GitHub, accessed October 25, 2025, [https://github.com/vszakats/hb](https://github.com/vszakats/hb)  
5. How to Compile Harbour for MS-DOS Compatible Apps ? \- Google Groups, accessed October 25, 2025, [https://groups.google.com/g/harbour-users/c/bYCS6x7Jl5Y](https://groups.google.com/g/harbour-users/c/bYCS6x7Jl5Y)  
6. How can I install and use "make" in Windows? \- Stack Overflow, accessed October 25, 2025, [https://stackoverflow.com/questions/32127524/how-can-i-install-and-use-make-in-windows](https://stackoverflow.com/questions/32127524/how-can-i-install-and-use-make-in-windows)  
7. Harbour nightly builds \- Google Groups, accessed October 25, 2025, [https://groups.google.com/g/harbour-users/c/alkvJEL7w-8](https://groups.google.com/g/harbour-users/c/alkvJEL7w-8)  
8. How to build Harbour 32 & 64 bits \- FiveTech Software tech support forums, accessed October 25, 2025, [https://fivetechsoft.com/forums/viewtopic.php?t=21695](https://fivetechsoft.com/forums/viewtopic.php?t=21695)  
9. Cross compiler Linux \-\> Windows 32/64Bit \- Google Groups, accessed October 25, 2025, [https://groups.google.com/g/harbour-users/c/crpMyjyAsns](https://groups.google.com/g/harbour-users/c/crpMyjyAsns)  
10. antimix/harbour-core: Harbour mainline \+ Additions and Fixes \= Version 3.4 \- GitHub, accessed October 25, 2025, [https://github.com/antimix/harbour-core](https://github.com/antimix/harbour-core)  
11. Building Harbour \- Learnings on Solaris™, accessed October 25, 2025, [https://learnings-on-solaris.blogspot.com/2017/07/building-harbour.html](https://learnings-on-solaris.blogspot.com/2017/07/building-harbour.html)  
12. harbour with mingw64 \- Google Groups, accessed October 25, 2025, [https://groups.google.com/g/harbour-users/c/13J2TkGdDtg/m/gm4HxBa48sUJ](https://groups.google.com/g/harbour-users/c/13J2TkGdDtg/m/gm4HxBa48sUJ)  
13. Harbour \- step by step, accessed October 25, 2025, [https://www.kresin.ru/en/hrbsteps.html](https://www.kresin.ru/en/hrbsteps.html)  
14. Harbour Make (hbmk2) \- Viva Clipper \- WordPress.com, accessed October 25, 2025, [https://vivaclipper.wordpress.com/2013/11/02/harbour-make-hbmk2/](https://vivaclipper.wordpress.com/2013/11/02/harbour-make-hbmk2/)  
15. Harbor \- installation guide on windows systems \- BlogFaq400, accessed October 25, 2025, [https://blog.faq400.com/en/03-open-source-en/harbor-windows-installation-guide-en/](https://blog.faq400.com/en/03-open-source-en/harbor-windows-installation-guide-en/)