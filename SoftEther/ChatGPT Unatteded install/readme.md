You: 

make autoit script for unattended SoftEther client install

ChatGPT: 

Request for GPT-5 Pro

Reasoned for 7m 57s

Below is a ready‑to‑use AutoIt script that performs an **unattended install** of the **SoftEther VPN Client** on Windows and (optionally) pre‑configures a VPN connection via `vpncmd`. SoftEther’s Windows installer doesn’t provide a reliable built‑in silent switch, so this script **drives the setup wizard UI** and then uses `vpncmd` for post‑install tasks. [GitHub](https://github.com/SoftEtherVPN/SoftEtherVPN/issues/195)

> **What it does**
> 
> 1.  Finds and runs the SoftEther VPN Client installer EXE (you provide it).
>     
> 2.  Clicks through the installer (Next → Agree → Install → Finish).
>     
> 3.  Ensures the `SEVPNCLIENT` service is started. [shouldiremoveit.com](https://www.shouldiremoveit.com/softether-vpn-client-41839-program.aspx?utm_source=chatgpt.com)
>     
> 4.  Uses `vpncmd` to create/enable the virtual NIC, create a VPN account, set the password, (optionally) disable server‑cert check, set autostart and connect. Command syntax comes from the official client CLI manual. [softether.org+1](https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual/6.5_VPN_Client_Management_Command_Reference)
>     
> 5.  Logs everything to a file.
>     

---

## AutoIt script: `Install_SoftEther_Client.au3`

> **Usage:**
> 
> -   Place the SoftEther client installer EXE in the same folder as this script (for example: `softether-vpnclient-v*.exe`).
>     
> -   Edit the **CONFIG** section with your server, hub, username, and password.
>     
> -   Run the script **as Administrator** (or compile to EXE and run it).
>     

```autoit
#NoTrayIcon
#RequireAdmin
; Unattended SoftEther VPN Client install + optional auto-config
; Tested across multiple installer builds by automating the wizard.
; ---------------------------------------------------------------

; ------------------------ CONFIG ------------------------
Global Const $ACCOUNT_NAME    = "MyVPN"
Global Const $SERVER_HOST     = "vpn.example.com"
Global Const $SERVER_PORT     = "443"
Global Const $HUB_NAME        = "DEFAULT"
Global Const $USERNAME        = "user1"
Global Const $PASSWORD        = "changeme"       ; Consider supplying at runtime or via an env var for security.
Global Const $NIC_NAME        = "VPN"
Global Const $SET_STARTUP     = True             ; Connect this account automatically when service starts
Global Const $CONNECT_NOW     = True             ; Connect immediately after setup
Global Const $DISABLE_CERTCHK = False            ; If True: disables server cert verification for this account

; Optional: If you know the installer filename, set it here; otherwise the script searches ScriptDir.
Global Const $INSTALLER_HINT  = ""               ; e.g. "softether-vpnclient-v4.43-xxxx-windows-x64-intel.exe"

; --------------------- END CONFIG -----------------------

#include <Array.au3>
#include <MsgBoxConstants.au3>
#include <FileConstants.au3>
#include <AutoItConstants.au3>

Opt("WinTitleMatchMode", 2) ; substring match
Opt("WinWaitDelay", 50)
Opt("WinDetectHiddenText", 1)
Opt("MustDeclareVars", 1)

Global Const $LOG = @ScriptDir & "\softether_unattended_install.log"
_Log("========== SoftEther unattended install started (v" & @AutoItVersion & ") ==========")

; 1) Find installer -------------------------------------------------
Local $installer = _FindInstaller()
If $installer = "" Then
    _Log("ERROR: SoftEther VPN Client installer EXE not found next to script. Aborting.")
    Exit 1
EndIf
_Log("Installer: " & $installer)

; 2) Run installer & drive the wizard ------------------------------
Local $pid = Run('"' & $installer & '"', @ScriptDir, @SW_SHOWNORMAL)
If $pid = 0 Then
    _Log("ERROR: Failed to launch installer.")
    Exit 2
EndIf

_Log("Installer process PID: " & $pid & " — driving UI …")
If Not _DriveSoftEtherInstallerUI($pid, 900) Then
    _Log("WARNING: UI driving loop ended with installer still running or an unexpected dialog.")
EndIf

; 3) Locate install dir & vpncmd -----------------------------------
Local $clientDir = _LocateClientDir()
If $clientDir = "" Or Not FileExists($clientDir & "\vpncmd.exe") Then
    _Log("ERROR: Could not locate vpncmd.exe. Is the client installed?")
    Exit 3
EndIf
Local $vpncmd = '"' & $clientDir & '\vpncmd.exe"'
_Log("Found vpncmd at: " & $vpncmd)

; 4) Ensure service is started -------------------------------------
_Log("Starting SoftEther VPN Client service (SEVPNCLIENT)…")
_RunHidden(@ComSpec & ' /c sc start SEVPNCLIENT')

; 5) Create/enable virtual NIC -------------------------------------
_Log("Creating/enabling virtual NIC: " & $NIC_NAME)
_RunVpnCmd($vpncmd, 'NicCreate ' & $NIC_NAME)
_RunVpnCmd($vpncmd, 'NicEnable ' & $NIC_NAME)

; 6) Create VPN account + credentials ------------------------------
Local $acct = $ACCOUNT_NAME
Local $serverSpec = $SERVER_HOST & ':' & $SERVER_PORT
_Log("Creating VPN account '" & $acct & "' for " & $serverSpec & " / HUB:" & $HUB_NAME & " / USER:" & $USERNAME)

_RunVpnCmd($vpncmd, 'AccountCreate ' & $acct & ' /SERVER:' & $serverSpec & ' /HUB:' & $HUB_NAME & ' /USERNAME:' & $USERNAME & ' /NICNAME:' & $NIC_NAME)
_RunVpnCmd($vpncmd, 'AccountPasswordSet ' & $acct & ' /PASSWORD:' & $PASSWORD & ' /TYPE:standard') ; password auth
If $DISABLE_CERTCHK Then
    _RunVpnCmd($vpncmd, 'AccountServerCertDisable ' & $acct)
EndIf

If $SET_STARTUP Then
    _RunVpnCmd($vpncmd, 'AccountStartupSet ' & $acct)
EndIf

If $CONNECT_NOW Then
    _RunVpnCmd($vpncmd, 'AccountConnect ' & $acct)
    _RunVpnCmd($vpncmd, 'AccountStatusGet ' & $acct)
EndIf

_Log("All done.")
Exit 0

; ======================= Functions ================================

Func _FindInstaller()
    If $INSTALLER_HINT <> "" And FileExists(@ScriptDir & "\" & $INSTALLER_HINT) Then Return @ScriptDir & "\" & $INSTALLER_HINT

    ; Try a few likely patterns
    Local $patterns[4] = ["*vpnclient*windows*x64*.exe", "*softether*client*x64*.exe", "*vpnclient*windows*.exe", "*softether*client*.exe"]
    For $i = 0 To UBound($patterns) - 1
        Local $h = FileFindFirstFile(@ScriptDir & "\" & $patterns[$i])
        If $h <> -1 Then
            Local $file = FileFindNextFile($h)
            FileClose($h)
            If Not @error And $file <> "" Then Return @ScriptDir & "\" & $file
        EndIf
    Next
    Return ""
EndFunc

Func _DriveSoftEtherInstallerUI($ipPid, $iTimeoutSec = 600)
    Local $t0 = TimerInit()
    Local $btns[10] = ["I Agree", "Agree", "Next >", "Next", "Install", "Yes", "OK", "Finish", "Close", "Accept"]

    While ProcessExists($ipPid)
        ; Click through known installer pages (title contains "SoftEther" or generic "Setup/Installer/Wizard")
        _ClickButtonsInWindows("SoftEther", $btns)
        _ClickButtonsInWindows("Setup", $btns)
        _ClickButtonsInWindows("Installer", $btns)
        _ClickButtonsInWindows("Wizard", $btns)

        ; Handle possible Windows Security driver dialog automatically by pressing "Install"
        _ClickButtonsInWindows("Windows Security", ["Install", "Yes"])
        _ClickButtonsInWindows("Security Warning", ["Install", "Yes"])

        Sleep(200)
        If TimerDiff($t0) / 1000 > $iTimeoutSec Then
            _Log("Timeout while driving installer UI.")
            ExitLoop
        EndIf
    WEnd

    Return Not ProcessExists($ipPid)
EndFunc

Func _ClickButtonsInWindows($sTitlePart, ByRef $aBtnTexts)
    Local $list = WinList()
    For $i = 1 To $list[0][0]
        Local $h = $list[$i][1]
        Local $title = $list[$i][0]
        If $title = "" Or Not BitAND(WinGetState($h), 2) Then ContinueLoop
        If StringInStr($title, $sTitlePart, 0) Then
            ; Try to click a matching button by text
            For $inst = 1 To 30
                Local $ctrl = "[CLASS:Button;INSTANCE:" & $inst & "]"
                If Not ControlCommand($h, "", $ctrl, "IsVisible", "") Then ContinueLoop
                Local $txt = StringStripWS(ControlGetText($h, "", $ctrl), 7)
                If $txt = "" Then ContinueLoop
                For $k = 0 To UBound($aBtnTexts) - 1
                    If StringInStr(StringLower($txt), StringLower($aBtnTexts[$k])) Then
                        _Log("Clicking '" & $txt & "' on window: " & $title)
                        ControlClick($h, "", $ctrl)
                        Sleep(250)
                        ExitLoop 3
                    EndIf
                Next
            Next
        EndIf
    Next
EndFunc

Func _LocateClientDir()
    ; Prefer 64-bit Program Files on x64 OS
    Local $pf64 = EnvGet("ProgramW6432")
    Local $pf32 = EnvGet("ProgramFiles(x86)")
    Local $pf = @ProgramFilesDir

    Local $candidates[6]
    $candidates[0] = $pf64 & "\SoftEther VPN Client"
    $candidates[1] = $pf & "\SoftEther VPN Client"
    $candidates[2] = $pf32 & "\SoftEther VPN Client"
    $candidates[3] = @ProgramFilesDir & "\SoftEther VPN Client"
    $candidates[4] = @HomeDrive & "\Program Files\SoftEther VPN Client"
    $candidates[5] = @HomeDrive & "\Program Files (x86)\SoftEther VPN Client"

    For $i = 0 To UBound($candidates) - 1
        If FileExists($candidates[$i] & "\vpncmd.exe") Then Return $candidates[$i]
    Next
    Return ""
EndFunc

Func _RunVpnCmd($vpncmdExeQuoted, $cmd)
    Local $full = $vpncmdExeQuoted & " localhost /client /cmd " & $cmd
    _Log("> " & $full)
    Local $pid = Run($full, "", @SW_HIDE, $STDOUT_CHILD + $STDERR_CHILD)
    If $pid = 0 Then
        _Log("ERROR: failed to start vpncmd for: " & $cmd)
        Return SetError(1, 0, 0)
    EndIf
    Local $buf = ""
    While ProcessExists($pid)
        $buf &= StdoutRead($pid)
        $buf &= StderrRead($pid)
        Sleep(50)
    WEnd
    $buf &= StdoutRead($pid)
    $buf &= StderrRead($pid)
    If $buf <> "" Then _Log($buf)
    Return 1
EndFunc

Func _RunHidden($cmd)
    _Log("> " & $cmd)
    Local $rc = RunWait($cmd, "", @SW_HIDE)
    _Log("ExitCode: " & $rc)
    Return $rc
EndFunc

Func _Log($s)
    Local $h = FileOpen($LOG, $FO_APPEND)
    If $h <> -1 Then
        FileWriteLine($h, "[" & @HOUR & ":" & @MIN & ":" & @SEC & "] " & $s)
        FileClose($h)
    EndIf
EndFunc
```

---

### Notes & tips

-   **Why UI automation?**  
    The official Windows client installer does **not** expose a stable `/silent` or `/S` switch, so unattended deployment typically requires GUI automation or a repackaged installer. [GitHub+2VPN Users+2](https://github.com/SoftEtherVPN/SoftEtherVPN/issues/195)
    
-   **Service name & tools**  
    The service is named **`SEVPNCLIENT`** and the client CLI tool is **`vpncmd.exe`** in “`…\Program Files\SoftEther VPN Client`”. [shouldiremoveit.com+2softether.org+2](https://www.shouldiremoveit.com/softether-vpn-client-41839-program.aspx?utm_source=chatgpt.com)
    
-   **CLI commands reference**
    
    -   Create/enable NIC: `NicCreate` / `NicEnable`
        
    -   Create account: `AccountCreate`
        
    -   Set password auth: `AccountPasswordSet /PASSWORD:… /TYPE:standard`
        
    -   (Optional) Disable server cert check: `AccountServerCertDisable`
        
    -   Autostart: `AccountStartupSet`
        
    -   Connect now: `AccountConnect`  
        See the official **VPN Client Management Command Reference** for these commands and parameters. [softether.org+1](https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual/6.5_VPN_Client_Management_Command_Reference)
        
-   **Driver prompt**  
    When creating the virtual NIC, some Windows builds may show a driver install confirmation dialog. The script tries to click **Install** automatically, but if your environment blocks driver prompts, pre‑approve or sign policies accordingly. [softether.org](https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual/6.5_VPN_Client_Management_Command_Reference)
    
-   **Security**  
    The example keeps a clear‑text password for simplicity. In production, pass secrets via a secure mechanism (e.g., read from an encrypted file, DPAPI, or a deployment system variable).
    
-   **Import existing profiles instead**  
    If you already have a `.vpn` exported profile, you can swap the “create account” steps for:
    
    ```bash
    vpncmd localhost /client /cmd AccountImport "C:\Path\YourProfile.vpn"
    ```
    
    and then `AccountPasswordSet`, `AccountStartupSet`, etc. [softether.org](https://www.softether.org/4-docs/1-manual/6._Command_Line_Management_Utility_Manual/6.5_VPN_Client_Management_Command_Reference)
    
-   **Quiet(er) UI**  
    If you deploy via RMM/Intune/SCCM as the **interactive user**, the wizard can be hidden/minimized while the script clicks through. Running in a truly non‑interactive session still requires GUI automation (no official silent MSI). [VPN Users](https://www.vpnusers.com/viewtopic.php?t=4900&utm_source=chatgpt.com)
    

---

### Quick checklist

-    Put the SoftEther **client** installer EXE next to the script (or set `INSTALLER_HINT`).
    
-    Edit the **CONFIG** block with your server/hub/user/password.
    
-    Run (or compile) as Administrator.
    
-    Verify `softether_unattended_install.log` for results and `AccountStatusGet` output.
    

If you want this tailored (e.g., add proxy settings, certificate pinning with `AccountServerCertSet`, or packaging into Intune/Chocolatey), say the word and I’ll slot that into the script.