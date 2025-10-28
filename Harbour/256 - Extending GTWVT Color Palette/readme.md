
# **Architectural Blueprint for Extending Harbour's GTWVT Color Palette to 256 Entries**

This document presents a comprehensive architectural strategy for the re-engineering of the Harbour GTWVT terminal driver to support a 256-color palette. The primary objective is to extend the functionality of the HB\_GTI\_PALETTE interface within the GTWVT driver, thereby enabling Harbour applications to move beyond the legacy 16-color limitation. The analysis and proposed solution are predicated on a foundational understanding of GTWVT's core architecture: it is not a traditional console application that interacts with the Windows Console Host (conhost.exe), but rather a self-contained Win32 GUI application that emulates a terminal by rendering text directly onto its window surface using the Windows Graphics Device Interface (GDI). This critical distinction dictates the entire engineering approach, focusing modifications on the GDI-based drawing routines and data structures within the GTWVT C source code. This blueprint dismisses alternative, console-centric technologies as architecturally incompatible and provides a phased, low-risk implementation plan designed to ensure full backward compatibility with existing applications. The structured methodology outlined herein serves as a definitive engineering guide for modifying the Harbour source code to achieve the desired 256-color capability.

## **I. Deconstructing the GTWVT Rendering Engine: A GDI-Based Architecture**

A successful modification of the GTWVT driver hinges on a precise understanding of its underlying rendering technology. Unlike other terminal drivers in the Harbour ecosystem that interface with a system's native console subsystem, GTWVT operates as a standalone graphical application. This section deconstructs this architecture to establish the necessary context for the proposed changes and to definitively demonstrate why the solution lies within the Windows GDI framework, not the Windows Console API or its modern extensions.

### **The Harbour Graphic Terminal (GT) Abstraction Layer**

Harbour achieves its cross-platform terminal I/O capabilities through a powerful abstraction layer known as the Graphic Terminal (GT) system. This system provides a device-independent API for Harbour applications, allowing high-level commands such as SetColor(), @...SAY, and Inkey() to function consistently regardless of the underlying operating system or the specific terminal driver in use. When a Harbour program is compiled, these commands are translated into calls to a standardized set of C-level function pointers within the GT framework.

The actual implementation of these functions is delegated to the selected GT driver, which is linked into the final executable. For example, a call to SetColor() in a Harbour program ultimately invokes a C function like pGt-\>pfnSetColor( fg, bg ) within the GT system. The GT driver (e.g., GTWVT, GTWIN, or GTK) provides the concrete implementation of this function, translating the abstract request into platform-specific API calls. This architectural separation is what allows the same Harbour application code to run in a Windows console, a GTWVT GUI window, or a Linux terminal without modification.

### **GTWVT vs. GTWIN: A Tale of Two Architectures**

To fully appreciate the unique nature of GTWVT, it is instructive to compare it with its sibling driver, GTWIN. While both target the Windows platform, their fundamental architectures are diametrically opposed.

* **GTWIN: The Console API Driver:** GTWIN is a true console driver. It operates as a client of the Windows Console subsystem, managed by the Console Host process (conhost.exe). All text output is performed by calling functions from the classic Windows Console API, such as WriteConsoleOutput or SetConsoleTextAttribute.2 Its color capabilities, therefore, are intrinsically bound to the features and limitations of the console host it is attached to. By default, this means it is limited to the standard 16-color palette defined by the console's properties.3  
* **GTWVT: The GDI-Based GUI Driver:** In stark contrast, GTWVT does not interact with the Windows Console Host for its output. As its documentation implies, it "creates its own GUI window instead of using MS-console window". GTWVT is a pure Win32 application that registers a window class (WNDCLASS), creates a top-level window, and manages its own message loop (GetMessage, DispatchMessage). All visual output—every character, every colored background cell—is explicitly drawn onto this window's client area using the Windows GDI. Evidence of this can be seen in proposed patches to the GTWVT source code, which directly call GDI functions like CreateFontIndirect and TextOutW to render text.4 This means that GTWVT is, in effect, a custom-built terminal emulator, where the "screen buffer" is a region of memory managed by the driver, and the "display" is a GDI device context (HDC).

This architectural divergence is the single most important factor in determining the correct strategy for color palette extension. Because GTWVT controls its own rendering pipeline via GDI, it is not subject to the limitations of the Windows Console. Its color capabilities are limited only by what can be expressed through GDI functions, which have supported 24-bit true color for decades. The current 16-color limit is not an external constraint imposed by Windows, but an internal limitation of the GTWVT driver's own implementation.

### **The Inapplicability of Virtual Terminal (VT) Sequences**

Any modern investigation into enhancing color support in a Windows command-line environment will inevitably lead to the topic of Virtual Terminal (VT) sequences, also known as ANSI escape codes. Since the Windows 10 Anniversary Update, the Windows Console Host has supported a rich dialect of these sequences, enabling features like 256-color and true-color output in console applications.5 It is crucial, however, to recognize that this entire technology stack is architecturally incompatible with GTWVT and represents a significant potential "red herring" for development.

VT sequences are special command strings embedded within the standard output stream. For example, to set the foreground color to index 214 from the 256-color palette, an application would write the sequence \\e\[38;5;214m to its output.7 A terminal program (like Windows Terminal, conhost.exe, or xterm on Linux) intercepts and interprets this sequence, changing its internal state to render subsequent text in the specified color. To enable this behavior in a standard Windows console application, a program must first get the handle to the console output buffer and call the SetConsoleMode API with the ENABLE\_VIRTUAL\_TERMINAL\_PROCESSING flag.9

This mechanism is entirely irrelevant to GTWVT for one simple reason: GTWVT is not a client of the Windows Console Host. It is the terminal. There is no intermediary process to interpret VT sequences. Calling SetConsoleMode on a handle related to GTWVT would have no effect on its rendering, as it does not use the console output buffer. If a Harbour application running under GTWVT were to output a VT sequence, the driver would simply interpret it as a string of literal characters and dutifully render ESC, ; and COLORREF bgColor \= s\_aStdPalette;.  
4\. Windows GDI Layer: The retrieved COLORREF values are then passed to the Windows GDI. The driver calls SetTextColor(hDC, fgColor) and SetBkColor(hDC, bgColor), where hDC is the device context handle for the GTWVT window. From this point forward, any text rendered to that device context using functions like TextOut or ExtTextOut will appear in the selected colors until they are changed again.  
This flow confirms that the critical points of intervention are the internal COLORREF palette array and the logic that translates color requests into indices for that array.

| Feature | Classic Console API (for GTWIN) | Virtual Terminal Sequences (Not for GTWVT) |
| :---- | :---- | :---- |
| **Enabling Mechanism** | Enabled by default for console applications. | SetConsoleMode(hOut, ENABLE\_VIRTUAL\_TERMINAL\_PROCESSING).10 |
| **Color Setting Command** | SetConsoleTextAttribute(hOut, wAttributes) | printf("\\e\[38;5;\<n\>m") for foreground.7 |
| **Color Space** | 16 fixed colors, user-configurable. | 256 colors (xterm palette) or 24-bit True Color.11 |
| **Architectural Locus** | Interpretation and rendering handled by conhost.exe or Windows Terminal. | Interpretation and rendering handled by conhost.exe or Windows Terminal. |
| **Applicability to GTWVT** | **None.** GTWVT does not use the Console API for output. | **None.** GTWVT is its own renderer and does not interpret VT sequences. |

## **II. Analysis of the Existing 16-Color Palette Implementation**

Before extending the color system, it is essential to locate and understand the existing 16-color implementation within the Harbour source code. This involves navigating the source repository, identifying the key data structures, and analyzing the color parsing logic.

### **Locating the GTWVT Source Code**

The official source code for Harbour is hosted on GitHub in the harbour/core repository.12 Harbour's architecture organizes third-party and optional components into a contrib directory. As GTWVT is a platform-specific terminal driver, its source code is located within this directory. The specific path to the relevant files is contrib/gtwvt/. Within this directory, developers will find the C source files (e.g., gtwvt.c, wvtcore.c) and header files (wvt.h) that constitute the driver. These files contain the Win32 API calls, GDI rendering logic, and the color management system that must be modified.

### **Identifying the Core Palette Structure**

The 16 standard console colors are almost certainly defined within the GTWVT source code as a static C array of COLORREF types. A COLORREF is a 32-bit unsigned integer (DWORD) used by the Windows GDI to represent an RGB color. The format is 0x00bbggrr, where bb, gg, and rr are the 8-bit values for blue, green, and red, respectively. The WinAPI provides the RGB() macro to construct a COLORREF from its constituent components.

A search within the contrib/gtwvt/ source files for a definition similar to the following will locate the core palette structure:

C

/\* Hypothetical example of the existing 16-color palette definition \*/  
static COLORREF s\_aStdPalette \=  
{  
    RGB(  0,   0,   0),  // 0: Black  
    RGB(  0,   0, 128),  // 1: Blue  
    RGB(  0, 128,   0),  // 2: Green  
    RGB(  0, 128, 128),  // 3: Cyan  
    RGB(128,   0,   0),  // 4: Red  
    RGB(128,   0, 128),  // 5: Magenta  
    RGB(128, 128,   0),  // 6: Brown/Yellow  
    RGB(192, 192, 192),  // 7: Light Gray (Standard White)  
    RGB(128, 128, 128),  // 8: Dark Gray (Intense Black)  
    RGB(  0,   0, 255),  // 9: Light Blue  
    RGB(  0, 255,   0),  // 10: Light Green  
    RGB(  0, 255, 255),  // 11: Light Cyan  
    RGB(255,   0,   0),  // 12: Light Red  
    RGB(255,   0, 255),  // 13: Light Magenta  
    RGB(255, 255,   0),  // 14: Yellow  
    RGB(255, 255, 255)   // 15: White (Intense White)  
};

This array, s\_aStdPalette, is the central data structure that maps the 16 abstract Harbour color indices to concrete GDI color values. It is this static, hard-coded array that imposes the 16-color limitation.

### **Color Indexing and Mapping Logic**

Harbour's SetColor() function accepts a descriptive string that is more complex than simple indices. It supports standard color letters (N, B, G, BG, R, RB, W, GR), an intensity modifier (+), a blinking modifier (\*), and a background color specification following a slash (/).

Within the GTWVT C source, there must be a parsing function that translates this string into two integer indices: one for the foreground (0-15) and one for the background (0-7). This logic would, for example, parse "W+/B" into a foreground index of 15 (W=7 \+ \+=8) and a background index of 1 (B). These integer indices are then used to access the s\_aStdPalette array. The blinking attribute (\*) is typically handled by setting a flag that causes the text to be redrawn periodically with different colors, a behavior that is separate from the static palette definition itself. Understanding this existing parsing mechanism is crucial for designing an extension that can coexist with it.

## **III. A Multi-Phase Strategy for 256-Color Integration**

The following multi-phase strategy provides a structured and robust plan for integrating 256-color support into the GTWVT driver. This approach prioritizes backward compatibility, isolates changes into logical steps, and leverages the existing GDI-based architecture for a clean and efficient implementation.

### **Phase 1: Expanding the Core Palette Data Structure**

The first and most fundamental step is to replace the existing 16-element color palette with a new 256-element structure.

1. **Declaration:** Locate the static 16-element COLORREF array (e.g., s\_aStdPalette) and replace it with a 256-element version. It is recommended to use a new name to signify its extended nature, for example, static COLORREF s\_aExtPalette.  
2. **Initialization:** The new array must be initialized with the standard RGB values corresponding to the xterm 256-color model. This model is a de facto standard for 256-color terminals and is composed of three distinct sections 7:  
   * **Indices 0-15:** These must be populated with the original 16 ANSI colors to ensure perfect backward compatibility. The values can be copied directly from the old palette definition.  
   * **Indices 16-231:** This block contains a 6x6x6 color cube. The index can be calculated from a 0-5 red, green, and blue component using the formula: index \= 16 \+ (36 \* r) \+ (6 \* g) \+ b. The corresponding RGB values are typically spaced evenly (e.g., 0, 95, 135, 175, 215, 255 for each component).  
   * **Indices 232-255:** This block contains a 24-step grayscale ramp, starting from a dark gray (near black) and ending in a light gray (near white).

The following table provides the complete, ready-to-use C implementation for this 256-color palette.

| Index | xterm Description | Hex RGB | C Implementation (RGB macro) |
| :---- | :---- | :---- | :---- |
| 0 | Black | \#000000 | RGB(0, 0, 0\) |
| 1 | Red | \#800000 | RGB(128, 0, 0\) |
| 2 | Green | \#008000 | RGB(0, 128, 0\) |
| 3 | Yellow | \#808000 | RGB(128, 128, 0\) |
| 4 | Blue | \#000080 | RGB(0, 0, 128\) |
| 5 | Magenta | \#800080 | RGB(128, 0, 128\) |
| 6 | Cyan | \#008080 | RGB(0, 128, 128\) |
| 7 | White | \#c0c0c0 | RGB(192, 192, 192\) |
| 8 | Bright Black (Gray) | \#808080 | RGB(128, 128, 128\) |
| 9 | Bright Red | \#ff0000 | RGB(255, 0, 0\) |
| 10 | Bright Green | \#00ff00 | RGB(0, 255, 0\) |
| 11 | Bright Yellow | \#ffff00 | RGB(255, 255, 0\) |
| 12 | Bright Blue | \#0000ff | RGB(0, 0, 255\) |
| 13 | Bright Magenta | \#ff00ff | RGB(255, 0, 255\) |
| 14 | Bright Cyan | \#00ffff | RGB(0, 255, 255\) |
| 15 | Bright White | \#ffffff | RGB(255, 255, 255\) |
| 16-231 | 6x6x6 Color Cube | *(Varies)* | *(Calculated as 16 \+ 36\*r \+ 6\*g \+ b)* |
| ... | *e.g., Index 21* | \#0000ff | RGB(0, 0, 255\) |
| ... | *e.g., Index 46* | \#00ff00 | RGB(0, 255, 0\) |
| ... | *e.g., Index 196* | \#ff0000 | RGB(255, 0, 0\) |
| ... | *e.g., Index 226* | \#ffff00 | RGB(255, 255, 0\) |
| 232 | Grayscale 1 | \#080808 | RGB(8, 8, 8\) |
| 233 | Grayscale 2 | \#121212 | RGB(18, 18, 18\) |
| ... | *(Steps of approx. 10\)* | *(Varies)* | *(Calculated)* |
| 255 | Grayscale 24 | \#eeeeee | RGB(238, 238, 238\) |
| *(Note: A complete 256-entry table would be provided in an actual implementation document. The above is an illustrative abbreviation.)* |  |  |  |

### **Phase 2: Re-engineering the Color Selection Logic**

With the expanded palette in place, the logic that interprets the SetColor() string must be enhanced to access it. The paramount design principle for this phase is **non-breaking backward compatibility**.

A new, dual-mode parser must be implemented. This function will replace the existing color string parser. Its logic should proceed as follows:

1. **Initial Check:** The function receives the color string from Harbour (e.g., "W+/B" or "21/221").  
2. **Legacy Format Detection:** The parser first inspects the string to determine if it uses the legacy letter-based format. A simple heuristic is to check if the string (or its foreground/background components) contains any non-numeric characters (aside from \+ and \*).  
3. **Legacy Path:** If the legacy format is detected, the parser invokes the *original* color parsing logic. This logic will produce indices in the 0-15 range, which will correctly map to the first 16 entries of the new s\_aExtPalette array, ensuring that existing application code continues to function identically.  
4. **Numeric Format Detection:** If the string does not appear to be in the legacy format, the parser then attempts to interpret it as a pair of numeric indices.  
5. **Numeric Path:** The parser should use a function like sscanf or strtol to extract the integer values for the foreground and background colors. For example, sscanf(colorString, "%d/%d", \&fg\_index, \&bg\_index) could parse "21/221". The logic must also handle cases where only a foreground color is provided (e.g., "21").  
6. **Index Validation:** The parsed numeric indices must be validated to ensure they fall within the valid range of 0-255. Out-of-range values should be clamped or handled gracefully (e.g., by defaulting to a standard color) to prevent array out-of-bounds access.

This layered approach guarantees that the enhancement is purely additive. The new numeric format provides access to the full 256-color palette, while the legacy format continues to work as before, making the transition seamless for existing codebases.

### **Phase 3: Modifying the HB\_GTI\_PALETTE Interface**

The Harbour function hb\_gtInfo() provides a generic mechanism for applications to query and configure driver-specific settings. The selector HB\_GTI\_PALETTE is used to interact with the driver's color palette. The underlying C function in GTWVT that handles this selector must be updated.

This function will typically contain a switch statement that branches based on the selector passed to hb\_gtInfo(). The code block for case HB\_GTI\_PALETTE: needs to be modified for both "get" and "set" operations.

* **Setting the Palette:** The existing code likely expects a Harbour array containing 16 sub-arrays, each with three numbers (R, G, B). It iterates 16 times, updating the s\_aStdPalette. This logic must be changed to accommodate an array of up to 256 elements. The loop should now run up to 256 times (or the actual size of the passed array, whichever is smaller), updating the corresponding entries in the new s\_aExtPalette. This allows a Harbour application to dynamically redefine the entire 256-color palette at runtime.  
* **Getting the Palette:** The existing code constructs and returns a Harbour array of 16 RGB sub-arrays. This must be changed to iterate through all 256 entries of s\_aExtPalette and return a full 256-element Harbour array.

These changes ensure that the external interface for palette manipulation is consistent with the new internal 256-color capability.

### **Phase 4: Recompilation and Deployment**

Once the C source code modifications in contrib/gtwvt/ are complete, the Harbour project must be recompiled.

1. **Build Harbour:** Navigate to the root directory of the harbour/core source tree. Execute the platform-specific build command (e.g., make on Linux, or the appropriate build script for your Windows C compiler like MinGW or MSVC). The Harbour build system is designed to automatically detect the modified source files and recompile the gtwvt library (gtwvt.lib or gtwvt.a) along with the rest of the core components.  
2. **Relink Application:** After the Harbour libraries have been successfully rebuilt, the user's application must be relinked to incorporate the updated driver. This is typically done by running the Harbour make tool, hbmk2, with the application's build file (.hbp). No changes should be necessary in the .hbp file itself, as it already specifies the use of the GTWVT driver. The linker will automatically pick up the newly compiled gtwvt.lib and link its new functionality into the final executable. The application will then be ready to use the extended 256-color palette.

## **IV. Advanced Considerations and Best Practices**

Implementing the core functionality is only part of a robust engineering solution. This section addresses potential edge cases, performance implications, and alternative approaches to ensure the final implementation is stable, efficient, and well-designed.

### **Ensuring Robust Backward Compatibility**

The dual-mode parser described in Phase 2 is the cornerstone of backward compatibility, but its implementation requires careful consideration of edge cases to prevent ambiguous or incorrect behavior.

* **Ambiguous Inputs:** The parser must clearly define its behavior for inputs that could be interpreted in multiple ways. For instance, single-digit color strings like "7" could theoretically be a legacy color (White) or a numeric index. The rule should be that if an input can be parsed entirely as a valid number, the numeric path takes precedence. The legacy path should only be triggered by the presence of non-numeric characters like W, B, \+, etc.  
* **Error Handling:** The numeric parser must be resilient to invalid input. A string like "300/10" contains an out-of-range foreground index. The parser should not cause a crash; instead, it should clamp the value to the maximum valid index (255) or ignore the invalid component of the color pair. Similarly, malformed strings like "10/ABC" should be handled gracefully, perhaps by applying only the valid numeric part and ignoring the rest.  
* **Partial Numeric Specification:** The behavior for SetColor("21") should be explicitly defined. The most intuitive behavior, consistent with legacy SetColor(), is to set the foreground color to index 21 and leave the background color unchanged.

By preemptively defining the logic for these scenarios, the extended driver will behave predictably and reliably for both old and new application code.

### **Performance and Resource Analysis**

The proposed architectural changes will have a negligible impact on application performance and resource consumption.

* **Memory Footprint:** The primary change is the expansion of the static color palette array. The memory increase is from 16 \* sizeof(COLORREF) (64 bytes) to 256 \* sizeof(COLORREF) (1024 bytes). This increase of less than 1 KB is trivial in the context of any modern operating system and will have no measurable impact on the application's memory usage.  
* **CPU Performance:** The color lookup operation remains an O(1) direct array access, which is computationally insignificant. The new dual-mode color string parser adds, at most, a few string inspection operations and a conditional branch before falling into either the legacy or numeric parsing path. The overhead of this additional logic is minuscule and will not create any performance bottleneck or noticeable slowdown in rendering operations.

The overall performance profile of the GTWVT driver will remain unchanged, ensuring that the color extension comes at no cost to application speed or responsiveness.

### **Alternative GDI Techniques (And Why to Avoid Them)**

The Windows GDI includes a more complex mechanism for color management known as the Palette Manager, exposed through APIs like CreatePalette, SelectPalette, and RealizePalette. This technology was designed for legacy hardware that operated in 8-bit (256-color) *palettized* display modes. In such modes, the video hardware had a small, modifiable hardware color lookup table (CLUT). The Palette Manager allowed applications to define their own logical palettes and have Windows map them to the hardware palette to achieve the best possible color representation.

While this might seem relevant to a 256-color problem, using the GDI Palette Manager in this context would be a significant and unnecessary engineering error for several reasons:

1. **Obsoletion:** Palettized display modes are obsolete. Virtually all modern displays and graphics drivers operate in 24-bit or 32-bit true-color modes. In these modes, the GDI Palette Manager provides no functional benefit; it is effectively a compatibility layer that adds complexity without improving capability.  
2. **Complexity:** A proper implementation using the Palette Manager is far more complex than the proposed direct COLORREF approach. It requires creating and managing palette handles, selecting them into the device context, and responding to system window messages like WM\_QUERYNEWPALETTE and WM\_PALETTECHANGED to correctly handle palette realization for foreground and background windows. This introduces significant architectural overhead for no tangible gain.  
3. **No Added Value:** On a true-color display, calling SetTextColor with a direct COLORREF value of RGB(255, 95, 0\) achieves the exact same visual result as creating a logical palette containing that color, realizing the palette, and then setting the color using a palette index.

The proposed solution of using a simple array of COLORREF values and passing them directly to GDI functions like SetTextColor and SetBkColor is the modern, correct, and vastly simpler approach. It leverages the true-color capabilities of the underlying system directly, achieving the desired outcome with minimal code and maximum clarity.

## **Conclusion**

The extension of Harbour's GTWVT driver to support a 256-color palette is a highly achievable engineering task that requires a targeted modification of its internal GDI-based rendering engine. The architectural analysis presented in this document confirms that GTWVT operates as a self-contained GUI application, rendering its own terminal display, and is therefore independent of the limitations and technologies of the standard Windows Console.

The recommended architectural path is clear and low-risk. It fundamentally involves replacing the driver's internal 16-entry COLORREF array with a 256-entry structure initialized with the standard xterm color palette. This change must be accompanied by a carefully designed enhancement to the color string parser, implementing a dual-mode logic that introduces support for numeric color indices while preserving perfect backward compatibility for the legacy letter-based format. Finally, the C-level implementation of the hb\_gtInfo(HB\_GTI\_PALETTE) interface must be updated to allow Harbour applications to programmatically get and set the new, larger palette.

This report strongly advises against any investigation into console-centric solutions, such as Virtual Terminal (VT) sequences. Such an approach is architecturally incompatible with GTWVT's GDI-based design and would represent a misdirection of development effort. By adhering to the GDI-focused, multi-phase strategy detailed herein, developers can successfully implement a robust, performant, and seamlessly integrated 256-color capability for GTWVT, significantly enhancing the visual potential of Harbour applications on the Windows platform.

#### **Works cited**

1. Low-Level Console Output Functions \- Microsoft Learn, accessed October 23, 2025, [https://learn.microsoft.com/en-us/windows/console/low-level-console-output-functions](https://learn.microsoft.com/en-us/windows/console/low-level-console-output-functions)  
2. Is there a way to get more colors in Windows console (c++)? \- Stack Overflow, accessed October 23, 2025, [https://stackoverflow.com/questions/71488464/is-there-a-way-to-get-more-colors-in-windows-console-c](https://stackoverflow.com/questions/71488464/is-there-a-way-to-get-more-colors-in-windows-console-c)  
3. hb\_gt\_wvt\_gfx\_Text() · Issue \#291 · harbour/core \- GitHub, accessed October 23, 2025, [https://github.com/harbour/core/issues/291](https://github.com/harbour/core/issues/291)  
4. c++ \- ENABLE\_VIRTUAL\_TERMINAL\_PROCESSING and DISABLE\_NEWLINE\_AUTO\_RETURN failing \- Stack Overflow, accessed October 23, 2025, [https://stackoverflow.com/questions/46030331/enable-virtual-terminal-processing-and-disable-newline-auto-return-failing](https://stackoverflow.com/questions/46030331/enable-virtual-terminal-processing-and-disable-newline-auto-return-failing)  
5. D51615 Set Windows console mode to enable support for ansi escape codes \- LLVM Phabricator archive, accessed October 23, 2025, [https://reviews.llvm.org/D51615](https://reviews.llvm.org/D51615)  
6. Xterm 256 Colors \- TinTin++ MUD client, accessed October 23, 2025, [https://tintin.mudhalla.net/info/256color/](https://tintin.mudhalla.net/info/256color/)  
7. ANSI X3.64 and Xterm-256 Support \- ConEmu, accessed October 23, 2025, [https://conemu.github.io/en/AnsiEscapeCodes.html](https://conemu.github.io/en/AnsiEscapeCodes.html)  
8. Enable/disable/check color support for Windows (ENABLE\_VIRTUAL\_TERMINAL\_PROCESSING flag) \- GitHub Gist, accessed October 23, 2025, [https://gist.github.com/mlocati/21a9233ac83f7d3d7837535bc109b3b7](https://gist.github.com/mlocati/21a9233ac83f7d3d7837535bc109b3b7)  
9. SetConsoleMode function \- Windows Console \- Microsoft Learn, accessed October 23, 2025, [https://learn.microsoft.com/en-us/windows/console/setconsolemode](https://learn.microsoft.com/en-us/windows/console/setconsolemode)  
10. ANSI escape code \- Wikipedia, accessed October 23, 2025, [https://en.wikipedia.org/wiki/ANSI\_escape\_code](https://en.wikipedia.org/wiki/ANSI_escape_code)  
11. harbour/core: Portable, xBase compatible programming ... \- GitHub, accessed October 23, 2025, [https://github.com/harbour/core](https://github.com/harbour/core)  
12. Harbour, accessed October 23, 2025, [https://harbour.github.io/](https://harbour.github.io/)  
13. ANSI Escape Codes \- GitHub Gist, accessed October 23, 2025, [https://gist.github.com/fnky/458719343aabd01cfb17a3a4f7296797](https://gist.github.com/fnky/458719343aabd01cfb17a3a4f7296797)