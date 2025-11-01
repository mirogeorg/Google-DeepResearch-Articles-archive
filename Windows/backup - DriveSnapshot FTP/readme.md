
# **Secure Configuration Guide for Remote DriveSnapshot Backups**

## **Section 1: Executive Recommendation: Server and Protocol Selection**

This report details the recommended server software, network configuration, and client-side commands for transferring DriveSnapshot disk images from a remote Windows computer to a local server.

### **1.1 Direct Answer and Primary Recommendation**

The direct answer to the query "What FTP server should I use?" is that a standard File Transfer Protocol (FTP) server should **not** be used for this task. The use of standard FTP presents critical, unacceptable security risks for transferring a full disk image.

* **Primary Protocol Recommendation:** **SFTP (SSH File Transfer Protocol)**. This protocol is explicitly supported by DriveSnapshot 1 and provides robust, modern encryption for all communication.  
* **Primary Server Recommendation:** **Microsoft Windows OpenSSH Server**. This is a free, Microsoft-supported, and native component available in all modern Windows client and server operating systems, including Windows 11, Windows 10, and Windows Server 2019+.2

### **1.2 Critical Security Briefing: Why SFTP is Non-Negotiable**

A full disk image contains the entirety of a system's data, including operating system files, user credentials, cached passwords, and sensitive documents. Transferring this file securely is paramount.

* **FTP (File Transfer Protocol):** This is an insecure, legacy protocol.6 All data, including the username, password, and the entire disk image file, is transferred in **clear text**. Anyone monitoring the network traffic can intercept these credentials and the complete backup file.7 Microsoft documentation regarding FTP, even in other contexts, highlights its insecurity and recommends using a VPN if FTP is the only option.9  
* **FTPS (FTP over TLS/SSL):** This is a more secure option than plain FTP, as it wraps the transfer in TLS encryption, similar to HTTPS.10 However, it inherits all the protocol-level complexities of FTP, most notably the requirement for a "passive mode" port range, which complicates firewall configuration.  
* **SFTP (SSH File Transfer Protocol):** This is a modern, entirely separate protocol that is not related to FTP.11 It runs over the **SSH (Secure Shell)** protocol.6 Its primary advantages are:  
  1. **Strong Encryption:** All authentication and data transfer are encrypted by default.  
  2. **Simplicity:** It uses a **single port** (default 22\) for all communication, including authentication, commands, and data transfer.6 This dramatically simplifies network and firewall configuration.

DriveSnapshot explicitly supports SFTP, providing specific command-line options for it, such as identifying the connection *Type* as SFTP and managing SSH-specific features like host keys.1 This indicates that SFTP is a first-class, fully supported protocol by the software.

### **1.3 Server Software Comparative Analysis**

The recommendation of the Windows OpenSSH Server is based on a direct comparison with the most common alternative, FileZilla Server.

* **Recommended: Windows OpenSSH Server (Provides SFTP)**  
  * **Cost:** Free.  
  * **Source:** Native Windows component supported directly by Microsoft.2  
  * **Protocol:** Provides SFTP (and SSH) services.  
  * **Key Advantage:** Uses the single-port SFTP protocol, requiring only TCP port 22 to be opened on the firewall and router.4 This is simple and minimizes the network's attack surface.  
* **Alternative: FileZilla Server (Provides FTPS)**  
  * **Cost:** The standard FileZilla Server is free.15  
  * **Protocol:** This is the critical distinction. The **free FileZilla Server supports FTP and FTPS, but it does *not* support SFTP**.15 SFTP support is only available in the paid "FileZilla Pro Enterprise Server".16  
  * **Key Disadvantage:** To use the free version securely, one must use FTPS. This requires configuring **Passive Mode**, which involves opening the standard control port (21) *plus* a large range of passive data ports (e.g., 50100-51100) on the firewall and router.17 This is significantly more complex to configure and maintain.

**Conclusion:** The Windows OpenSSH Server is the superior choice. It is free, native to the OS, and provides the secure, single-port SFTP protocol that DriveSnapshot explicitly supports. The free FileZilla Server is a poor choice for this task as it forces the use of the more complex, multi-port FTPS protocol.

### **1.4 Key Insight: The Protocol vs. Server Trap**

A common point of failure is confusing the similar-sounding FTPS and SFTP protocols. This confusion can lead to downloading FileZilla Server (an "FTP Server") and attempting to connect to it using DriveSnapshot's SFTP protocol type, which will fail.

* **FTPS** \= FTP \+ TLS Encryption. It is still the FTP protocol.11  
* **SFTP** \= SSH File Transfer Protocol. It is a completely different protocol based on SSH.11

The following matrix summarizes the technical and security considerations that form the basis of the primary recommendation.

**Table 1: Protocol & Server Recommendation Matrix**

| Feature | Plain FTP (FileZilla / IIS) | FTPS (FTP over TLS) (FileZilla Server) | SFTP (SSH FTP) (Windows OpenSSH) |
| :---- | :---- | :---- | :---- |
| **Security** | **None (Insecure)** \[6, 7\] | **High (TLS Encryption)** 10 | **High (SSH Encryption)** \[6, 11\] |
| **Protocol Base** | FTP | FTP 11 | **SSH** 11 |
| **Default Port(s)** | 21 \[13\] | 21 (Control) \+ Passive Range (e.g., 5000+ ports) \[17, 18\] | **22 (Single Port)** \[13, 14\] |
| **Firewall/Router Complexity** | **Very High** (Requires passive port range) | **Very High** (Requires passive port range) 17 | **Low** (Single port forward) 14 |
| **DriveSnapshot Support** | Type=FTP 1 | Ambiguous (Not explicitly named) | **Explicit (Type=SFTP)** 1 |
| **Recommended Free Server** | None (Insecure) | FileZilla Server 15 | **Windows OpenSSH Server** \[2\] |
| **Expert Recommendation** | **STRONGLY REJECT** | **Secondary (Complex) Option** | **PRIMARY RECOMMENDATION** |

---

## **Section 2: Implementation Guide: Windows OpenSSH Server (Primary Recommendation)**

This guide provides the complete, step-by-step instructions for configuring the Windows OpenSSH Server on the local Windows machine that will receive the backups.

### **2.1 Installation and Service Activation**

First, the "OpenSSH Server" feature must be installed and activated.

* **Method 1: Using the GUI (Windows 11 / Windows 10\)**  
  1. Go to **Settings** \> **System** \> **Optional features** (on Windows 11\) or **Settings** \> **Apps** \> **Apps & features** \> **Optional features** (on Windows 10).2  
  2. Click "View features" or "Add a feature".  
  3. Locate "OpenSSH Server" in the list, select it, and click "Next" or "Install".2  
* **Method 2: Using PowerShell (Administrator)**  
  1. Right-click the Start menu and select **PowerShell (Admin)** or **Windows Terminal (Admin)**.  
  2. Run the following command to install the server 3:  
     Add-WindowsCapability \-Online \-Name OpenSSH.Server\~\~\~\~0.0.1.0  
* **Service Configuration:**  
  1. Press Win \+ R, type services.msc, and press Enter.3  
  2. In the services list, locate and double-click **OpenSSH SSH Server**.3  
  3. Change the **Startup type** from "Manual" to **Automatic**.3  
  4. Click the **Start** button to run the service.3

### **2.2 Creating the Secure Backup Environment**

To securely receive backups, a dedicated, non-administrator user and a "jailed" (chrooted) directory must be created. This prevents the backup user from accessing any other files on the server.

* **Step 1: Create the Backup User**  
  1. Press Win \+ R, type lusrmgr.msc, and press Enter.  
  2. Right-click on the "Users" folder and select "New User...".  
  3. **Username:** snapshot\_uploader  
  4. **Password:** Enter a very strong, complex password.  
  5. Uncheck "User must change password at next logon".  
  6. Check "Password never expires".  
  7. Click "Create".  
* **Step 2: Create the Directory Structure**  
  1. Create a root folder for the SFTP "jail" (e.g., D:\\SFTP\_Root).  
  2. Inside that folder, create a subdirectory for the actual file uploads (e.g., D:\\SFTP\_Root\\Uploads). This two-level structure is essential for the security configuration in the next section.  
* **Step 3: Configure the sshd\_config File**  
  1. Open File Explorer and navigate to the OpenSSH configuration directory: %programdata%\\ssh (this typically resolves to C:\\ProgramData\\ssh).5  
  2. Open the sshd\_config file using Notepad (or another text editor) *as an Administrator*.  
  3. Scroll to the *very end* of the file and add the following lines. These settings will apply *only* to the snapshot\_uploader user:  
     Ini, TOML  
     \# \--- DriveSnapshot SFTP Configuration \---  
     \# Ensure password authentication is enabled  
     PasswordAuthentication yes

     \# Match our specific backup user  
     Match User snapshot\_uploader  
         \# Force this user into an SFTP-only session, disabling shell access  
         ForceCommand internal-sftp \[5, 20, 21, 22\]

         \# Specify the "jail" root directory created in Step 2  
         ChrootDirectory D:\\SFTP\_Root \[5, 20, 22\]

         \# Disable all forms of tunneling and port forwarding for security  
         PermitTunnel no \[20\]  
         AllowAgentForwarding no \[20\]  
         AllowTcpForwarding no \[20\]  
         X11Forwarding no \[20, 21\]

* **Step 4: Restart the OpenSSH Service**  
  1. Return to the services.msc window.  
  2. Right-click **OpenSSH SSH Server** and select **Restart** for the new configuration to take effect.23

### **2.3 Critical Insight: The ChrootDirectory NTFS Permission "Trap"**

This is the most common and complex failure point when configuring OpenSSH on Windows. If the ChrootDirectory permissions are set incorrectly, the server will reject the snapshot\_uploader's connection.

The connection will fail because, for security reasons derived from UNIX principles, the **jail root directory (D:\\SFTP\_Root) must not be writable by the user being jailed**.24 All file system access control is handled by standard Windows NTFS permissions, not the sshd\_config file.21

The solution is to make the jail root (D:\\SFTP\_Root) read-only for the user, but grant write permissions to the subdirectory (D:\\SFTP\_Root\\Uploads).

* **Precise NTFS Permission Guide (Step-by-Step):**  
  1. **Configure Parent ("Jail") Directory (D:\\SFTP\_Root)**  
     * Right-click the D:\\SFTP\_Root folder \> **Properties**.  
     * Go to the **Security** tab and click **Advanced**.  
     * Click **Disable inheritance** and select "Convert inherited permissions into explicit permissions on this object."  
     * Ensure NT AUTHORITY\\SYSTEM and BUILTIN\\Administrators have Full Control.5  
     * Click **Add**, then "Select a principal," and type snapshot\_uploader to add the user.  
     * Set the snapshot\_uploader user's permissions to:  
       * **Allow:** Read & execute  
       * **Allow:** List folder contents  
       * **Allow:** Read  
       * **Crucially, ensure Write and Modify are *NOT* checked (Deny).**  
     * Click "OK" to apply.  
  2. **Configure Child ("Upload") Directory (D:\\SFTP\_Root\\Uploads)**  
     * Right-click the D:\\SFTP\_Root\\Uploads folder \> **Properties**.  
     * Go to the **Security** tab and click **Edit...**.  
     * Select the snapshot\_uploader user in the list.  
     * Check the **Allow** box for **Modify** and **Write**. This will grant the user write-access *only* inside this subdirectory.  
     * Click "OK" to apply.

When the snapshot\_uploader user logs in, they will be "jailed" in D:\\SFTP\_Root (which they cannot write to) and will see the Uploads folder. They can then navigate into Uploads and successfully write the backup file.

---

## **Section 3: Network and Accessibility Configuration**

With the server configured, it must be made accessible to the remote computer over the internet. This involves handling the server's public IP address and configuring the network router.

### **3.1 Managing Your "Real IP": The Dynamic DNS (DDNS) Requirement**

The "real IP" (public IP address) assigned by an Internet Service Provider (ISP) is typically *dynamic*, meaning it changes periodically.25 When this IP changes, the remote DriveSnapshot script will fail.

The solution is a **Dynamic DNS (DDNS)** service, which provides a fixed hostname (e.g., my-backup-server.noip.com) that automatically updates to point to the server's current public IP.26

* **Recommended Free Providers:**  
  * No-IP 28  
  * Dynu 27  
  * Afraid.org (FreeDNS) 31  
* **Setup Process:**  
  1. Create an account with a DDNS provider (e.g., No-IP).  
  2. Create a new, free hostname. The provider will automatically detect and associate it with the current public IP address.28  
  3. Configure an "update client" to keep the IP address current. There are two methods:  
     * **Method 1 (Software Client):** Install the provider's Dynamic Update Client (DUC) software on the server. This application runs in the background and notifies the DDNS provider of any IP changes.28  
     * **Method 2 (Router-based \- Preferred):** This is the most robust "set it and forget it" solution. Log in to the administration panel of the network router. Most modern routers have a built-in "DDNS" or "Dynamic DNS" settings page.27 Enter the DDNS provider, hostname, username, and password here. The router, which is always on, will manage IP updates without requiring any software on the server.

### **3.2 Router Port Forwarding Configuration**

The DDNS hostname points to the public IP of the router. The router must be instructed to *forward* incoming SFTP traffic to the server's *private* IP address (e.g., 192.168.1.x).14

* **Step-by-Step Guide:**  
  1. **Find Server's Private IP:** On the server, open a Command Prompt (cmd) and type ipconfig. Note the "IPv4 Address" (e.g., 192.168.1.50). It is strongly recommended to set a static IP or DHCP reservation for this server within the router's settings to ensure this private IP does not change.  
  2. **Access Router:** In a browser, navigate to the router's gateway IP address (commonly 192.168.1.1 or 192.168.0.1).14  
  3. **Find Port Forwarding:** Log in and locate the "Port Forwarding," "Virtual Server," or "NAT" settings page.14  
  4. **Create the SFTP Rule:**  
     * **Service Name:** SFTP (OpenSSH)  
     * **External Port:** 22  
     * **Internal Port:** 22  
     * **Protocol:** TCP  
     * **Device/Internal IP:** 192.168.1.50 (the server's private IPv4 address)  
  5. Save and apply the rule. The router will now forward all incoming TCP Port 22 traffic to the OpenSSH server.

### **3.3 Windows Defender Firewall Configuration**

Finally, the server's own firewall must allow the incoming connection. The OpenSSH Server installation *should* create this rule automatically 33, but it is critical to verify.

* **Verification and Manual Setup:**  
  1. On the server, open **Windows Defender Firewall with Advanced Security**.14  
  2. Click **Inbound Rules**.14  
  3. Look for a rule named "OpenSSH SSH Server (sshd)" or similar. Ensure it is enabled and allows connections on TCP Port 22\.  
  4. **If the rule is missing:**  
     * Click **New Rule...**.14  
     * **Rule Type:** Select **Port**.14  
     * **Protocol and Ports:** Select **TCP** and "Specific local ports:" 22\.14  
     * **Action:** Select **Allow the connection**.14  
     * **Profile:** Check all profiles (Domain, Private, Public) to ensure the connection is allowed from the internet.  
     * **Name:** Allow SFTP Port 22\.14  
     * Click "Finish".

---

## **Section 4: DriveSnapshot Client-Side Configuration (On the Remote PC)**

These commands must be executed on the *remote* computer that is being backed up.

### **4.1 Secure Credential Storage**

To avoid storing the SFTP password in a plain text batch file, DriveSnapshot provides a command to store the credentials securely in the remote machine's registry in an encrypted form.1

* **Command:** Open a Command Prompt *as Administrator* on the remote PC and run this command **one time**:  
  DOS  
  snapshot.exe \--AddFTPAccount:SFTP,snapshot\_uploader,my-backup-server.noip.com,YOUR\_STRONG\_PASSWORD,22

* **Analysis:**  
  * \--AddFTPAccount: The command to store credentials.1  
  * SFTP: The *Type*.1 This is critical; using FTP would fail.  
  * snapshot\_uploader: The username created in Section 2.2.  
  * my-backup-server.noip.com: The DDNS hostname from Section 3.1.  
  * YOUR\_STRONG\_PASSWORD: The strong password set for the user.  
  * 22: The port for SFTP.

Once this is run, DriveSnapshot will automatically retrieve these credentials when it sees a matching username and server, and the password no longer needs to be specified in the backup command.34

### **4.2 First-Time Connection: Handling the SFTP Host Key**

The first time any SFTP client connects to an SSH server, the server presents its "host key" for verification. This normally results in an interactive "Do you trust this host? (y/n)" prompt.1 If a backup script is run unattended, this prompt will cause the script to hang indefinitely, waiting for user input.

To prevent this, DriveSnapshot provides a command to pre-accept and store this host key.1

* **Command:** Run this command **one time** from a Command Prompt on the remote PC (after the server is fully configured):  
  DOS  
  snapshot.exe \--AcceptHostKey sftp://snapshot\_uploader@my-backup-server.noip.com

This will initiate a connection, accept the server's host key, and store it. All future connections will be automatic and non-interactive.

### **4.3 The Final DriveSnapshot Backup Command**

With credentials and the host key stored, the final backup command can be placed in a batch file or scheduled task.

* **Command:**  
  DOS  
  snapshot.exe C: sftp://snapshot\_uploader@my-backup-server.noip.com/Uploads/remote-pc-backup.sna \-oD:\\local-hash.txt

* **Analysis:**  
  * snapshot.exe C:: The command to back up the entire C: drive.35  
  * sftp://snapshot\_uploader@my-backup-server.noip.com: The SFTP destination. Notice the **password is not present**. DriveSnapshot will automatically find the securely stored credentials.34  
  * /Uploads/remote-pc-backup.sna: The path *inside the jail*. This Uploads folder corresponds to the D:\\SFTP\_Root\\Uploads directory on the server, which has the correct write permissions (from Section 2.3).  
  * \-oD:\\local-hash.txt: This is a **critical** option. The DriveSnapshot documentation states that when backing up to an FTP server, the hash file (for integrity) is not created on the server.34 The \-o option must be used to specify a path for a *local* hash file (on the remote machine).34

---

## **Section 5: Alternative Guide: FTPS with FileZilla Server (Secondary Option)**

If the primary recommendation of using the Windows OpenSSH Server is not feasible, this alternative guide details the configuration for the free FileZilla Server using FTPS. This method is more complex due to the networking requirements of the FTP protocol.

### **5.1 Installation and Basic Setup**

1. Download and install the free **FileZilla Server** from the official site.15  
2. After installation, the "FileZilla Server Interface" will open. Connect to the local server (default: 127.0.0.1, port 14148\) to begin configuration.

### **5.2 User and Folder Configuration**

1. In the menu, go to Edit \> Users.  
2. Click "Add" to create a new user (e.g., snapshot\_uploader) and set a strong password.  
3. Select the "Shared folders" page for this user.  
4. Click "Add" under the "Shared folders" panel and select a directory to be this user's home (e.g., D:\\Backups\\RemoteImages).  
5. Set the permissions for this folder to allow the user to write, delete, and list files. Enable **Files** (Write, Delete, Append) and **Directories** (Create, Delete, List).36

### **5.3 Configuring FTPS (FTP over TLS)**

This is essential for security. Do not use plain, unencrypted FTP.

1. In the menu, go to Edit \> Settings.  
2. Select the "FTP over TLS settings" page.  
3. Check "Allow FTP over TLS support (FTPS)".  
4. Click "Generate new certificate..." and follow the prompts to create a self-signed certificate. This will enable encryption.

### **5.4 The Complexity of Passive Mode (Crucial Drawback)**

FTPS, like FTP, requires a separate data channel for directory listings and file transfers.38 For a server behind a router, this requires configuring **passive mode**.

* **Step 1: Configure FileZilla Server for Passive Mode**  
  1. Go to Edit \> Settings \> Passive mode settings.  
  2. Check "Use custom port range" and enter a large range (e.g., 50100-51100).17  
  3. Under "External Server IP or Hostname", enter the DDNS hostname (e.g., my-backup-server.noip.com).32 This is *essential*—it tells the client the correct *public* address to connect to for data transfers.19  
* Step 2: Configure Router Port Forwarding  
  This is far more complex than the SFTP setup. Two rules are required:  
  1. Log in to the router's port forwarding page (see Section 3.2).  
  2. **Rule 1 (Control):** Forward **TCP Port 21** to the server's private IP.19  
  3. **Rule 2 (Data):** Forward the **TCP Port Range 50100-51100** to the server's private IP.17  
* Step 3: Configure Windows Firewall  
  Two firewall rules are also required:  
  1. Open "Windows Defender Firewall with Advanced Security" (see Section 3.3).  
  2. **Rule 1 (Control):** Create an Inbound Rule to allow **TCP Port 21**.  
  3. **Rule 2 (Data):** Create an Inbound Rule to allow the **TCP Port Range 50100-51100**.17

This configuration, with thousands of open ports, is significantly more complex and increases the network's attack surface compared to the single-port SFTP solution.

### **5.5 DriveSnapshot Command for FTPS**

The commands are similar, but the *Type* must be set to FTP.

1. Store Credentials (Run once on remote PC):  
   snapshot.exe \--AddFTPAccount:FTP,snapshot\_uploader,my-backup-server.noip.com,YOUR\_STRONG\_PASSWORD,21  
2. Final Backup Command:  
   snapshot.exe C: ftp://snapshot\_uploader@my-backup-server.noip.com/remote-pc-backup.sna \-oD:\\local-hash.txt

Note the ambiguity: DriveSnapshot's documentation only specifies FTP or SFTP as types.1 It is assumed that using Type=FTP will allow the client to automatically negotiate the FTPS encryption (known as "Explicit FTPS") when the server requires it, as configured. This lack of explicit documentation for FTPS is another reason to prefer the clearly supported SFTP protocol.

---

## **Section 6: Troubleshooting and Diagnostics**

If the remote connection fails, follow this diagnostic checklist.

### **6.1 If SFTP Connection Fails (OpenSSH)**

* Error: "Connection refused" 33  
  This error means the connection reached the public IP but was rejected by the server or router.  
  1. **Check the service:** Is the "OpenSSH SSH Server" service *running* in services.msc? 3  
  2. **Check the firewall:** Is the Inbound Rule for **TCP Port 22** *enabled* in Windows Defender Firewall? 4  
  3. **Check port forwarding:** Is the router forwarding **TCP Port 22** to the *correct* private IP address of the server? 14  
  4. **Check the port (ISP):** Some residential ISPs block inbound port 22\. As a test, change the ListenAddress port in sshd\_config to 2222, update the firewall and port forwarding rules, and try connecting on port 2222\.33  
* Error: "Connection times out"  
  This usually indicates a firewall or port forwarding error. The packets are being sent but are dropped by the router or firewall, receiving no response.44 Re-check router and firewall rules.  
* Error: "Host key verification failed" 1  
  The server's host key may have changed (e.g., after an OS reinstall). Run the \--AcceptHostKey command from Section 4.2 again on the remote machine.  
* Error: "Permission denied" or Connection Closes Immediately After Login  
  This is almost certainly the ChrootDirectory NTFS permission "trap" (see Section 2.3). The permissions on the jail root (D:\\SFTP\_Root) are incorrect. Ensure the snapshot\_uploader user has only Read/Execute permissions on the root and Modify/Write permissions only on the Uploads subdirectory.

### **6.2 If FTP/FTPS Connection Fails (FileZilla)**

* Error: "Connection refused" (on Port 21\)  
  This is the same as the SFTP "Connection refused" error. Check that the "FileZilla Server" service is running, the firewall rule for TCP Port 21 is active, and the router is forwarding TCP Port 21 correctly.  
* Error: "530 Login incorrect" 47  
  This error is positive news: it means the control connection on Port 21 was successful, but authentication failed.49  
  1. Double-check the username and password.  
  2. Check the NTFS permissions on the user's home directory (e.g., D:\\Backups\\RemoteImages). The user must have at least Read/List permissions to log in.47  
* Error: "Failed to retrieve directory listing" or Command Hangs After Login  
  This is the classic passive mode failure. The control connection on Port 21 worked, but the client's attempt to open a new data connection on a high-numbered port (e.g., 50123\) failed.  
  1. **Check router forwarding:** Ensure the *entire* passive port range (50100-51100) is forwarded in the router.18  
  2. **Check firewall:** Ensure the *entire* passive port range is allowed as an Inbound Rule in the Windows Firewall.17  
  3. **Check external IP setting:** In FileZilla's "Passive mode settings," ensure the "External Server IP or Hostname" is correctly set to the DDNS hostname.40 If this is blank, the server will incorrectly tell the remote client to connect to its *private* IP (e.g., 192.168.1.50), which is impossible from the internet.

#### **Works cited**

1. Snapshot \- command line options \- DriveSnapshot.de, accessed November 1, 2025, [http://www.drivesnapshot.de/en/commandline.htm](http://www.drivesnapshot.de/en/commandline.htm)  
2. Installing SFTP/SSH Server on Windows using OpenSSH \- WinSCP, accessed November 1, 2025, [https://winscp.net/eng/docs/guide\_windows\_openssh\_server](https://winscp.net/eng/docs/guide_windows_openssh_server)  
3. Get started with OpenSSH Server for Windows \- Microsoft Learn, accessed November 1, 2025, [https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh\_install\_firstuse](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)  
4. Installing SFTP/SSH Server on Windows using OpenSSH \- Experience League \- Adobe, accessed November 1, 2025, [https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-14765](https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-14765)  
5. OpenSSH Server Configuration for Windows \- Microsoft Learn, accessed November 1, 2025, [https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)  
6. How To Use SFTP to Securely Transfer Files with a Remote Server | DigitalOcean, accessed November 1, 2025, [https://www.digitalocean.com/community/tutorials/how-to-use-sftp-to-securely-transfer-files-with-a-remote-server](https://www.digitalocean.com/community/tutorials/how-to-use-sftp-to-securely-transfer-files-with-a-remote-server)  
7. \[Help\] trying to set up my first filezilla ftp server. failing miserably. : r/HomeServer \- Reddit, accessed November 1, 2025, [https://www.reddit.com/r/HomeServer/comments/2lfnzy/help\_trying\_to\_set\_up\_my\_first\_filezilla\_ftp/](https://www.reddit.com/r/HomeServer/comments/2lfnzy/help_trying_to_set_up_my_first_filezilla_ftp/)  
8. Insecure FTP connection \- error message \- FileZilla Forums, accessed November 1, 2025, [https://forum.filezilla-project.org/viewtopic.php?t=49650](https://forum.filezilla-project.org/viewtopic.php?t=49650)  
9. Deliver a Snapshot Through FTP \- SQL Server \- Microsoft Learn, accessed November 1, 2025, [https://learn.microsoft.com/en-us/sql/relational-databases/replication/publish/deliver-a-snapshot-through-ftp?view=sql-server-ver17](https://learn.microsoft.com/en-us/sql/relational-databases/replication/publish/deliver-a-snapshot-through-ftp?view=sql-server-ver17)  
10. File Sharing Solution using Filezilla® FTP Server \- Microsoft Marketplace, accessed November 1, 2025, [https://marketplace.microsoft.com/en-us/marketplace/apps/cloud-infrastructure-services.file-sharing-ftp?tab=Overview](https://marketplace.microsoft.com/en-us/marketplace/apps/cloud-infrastructure-services.file-sharing-ftp?tab=Overview)  
11. FileZilla Supported Protocols, accessed November 1, 2025, [https://filezillapro.com/docs/v3/features-v3/filezilla-protocols/](https://filezillapro.com/docs/v3/features-v3/filezilla-protocols/)  
12. SFTP commands cheat sheet, accessed November 1, 2025, [https://sftptogo.com/blog/sftp-commands-cheat-sheet/](https://sftptogo.com/blog/sftp-commands-cheat-sheet/)  
13. Reading a file on a remote FTP, FTPS, or SFTP directory \- IBM, accessed November 1, 2025, [https://www.ibm.com/docs/en/app-connect/11.0.0?topic=files-reading-file-remote-ftp-ftps-sftp-directory](https://www.ibm.com/docs/en/app-connect/11.0.0?topic=files-reading-file-remote-ftp-ftps-sftp-directory)  
14. Filezilla FTP Server \- How to enable FTP over TLS on Windows 10 \- YouTube, accessed November 1, 2025, [https://www.youtube.com/watch?v=ncQZJw9W7QQ](https://www.youtube.com/watch?v=ncQZJw9W7QQ)  
15. FileZilla \- The free FTP solution, accessed November 1, 2025, [https://filezilla-project.org/](https://filezilla-project.org/)  
16. Protocol Support in FileZilla Server vs. FileZilla Pro Enterprise Server, accessed November 1, 2025, [https://filezillapro.com/docs/server/basic-usage-instructions-server/protocols-supported-filezilla-server/](https://filezillapro.com/docs/server/basic-usage-instructions-server/protocols-supported-filezilla-server/)  
17. Setup FileZilla Server Passive Ports on Windows Server 2012 — (B)logs of Johan Dörper, accessed November 1, 2025, [https://johandorper.com/log/filezilla-server-passive-ports](https://johandorper.com/log/filezilla-server-passive-ports)  
18. Please help me set up port forwarding \- FileZilla Forums, accessed November 1, 2025, [https://forum.filezilla-project.org/viewtopic.php?t=61552](https://forum.filezilla-project.org/viewtopic.php?t=61552)  
19. How to port forward passive FTP from a router \- Super User, accessed November 1, 2025, [https://superuser.com/questions/1258005/how-to-port-forward-passive-ftp-from-a-router](https://superuser.com/questions/1258005/how-to-port-forward-passive-ftp-from-a-router)  
20. ssh \- How do I restrict users to SFTP in OpenSSH on Windows ..., accessed November 1, 2025, [https://superuser.com/questions/1549527/how-do-i-restrict-users-to-sftp-in-openssh-on-windows-server](https://superuser.com/questions/1549527/how-do-i-restrict-users-to-sftp-in-openssh-on-windows-server)  
21. OpenSSH Windows Server 2019 \- Restricting "Execute" Permissions \- Microsoft Q\&A, accessed November 1, 2025, [https://learn.microsoft.com/en-us/answers/questions/1168301/openssh-windows-server-2019-restricting-execute-pe](https://learn.microsoft.com/en-us/answers/questions/1168301/openssh-windows-server-2019-restricting-execute-pe)  
22. Installing OpenSSH Server on Windows 11 \- Greg Beifuss, accessed November 1, 2025, [https://gbeifuss.github.io/p/installing-openssh-server-on-windows-11/](https://gbeifuss.github.io/p/installing-openssh-server-on-windows-11/)  
23. OpenSSH SFTP: chrooted user with access to other chrooted users' files \- Server Fault, accessed November 1, 2025, [https://serverfault.com/questions/138362/openssh-sftp-chrooted-user-with-access-to-other-chrooted-users-files](https://serverfault.com/questions/138362/openssh-sftp-chrooted-user-with-access-to-other-chrooted-users-files)  
24. What is Dynamic DNS? How Does It Work and How to Setup DDNS? \- ClouDNS, accessed November 1, 2025, [https://www.cloudns.net/blog/what-is-dynamic-dns/](https://www.cloudns.net/blog/what-is-dynamic-dns/)  
25. Dynamic DNS | Cloudflare, accessed November 1, 2025, [https://www.cloudflare.com/learning/dns/glossary/dynamic-dns/](https://www.cloudflare.com/learning/dns/glossary/dynamic-dns/)  
26. Dynamic DNS (DDNS) for Free: Remote Access to Home Server with Dynu \- YouTube, accessed November 1, 2025, [https://www.youtube.com/watch?v=wCJjiHp0d0w](https://www.youtube.com/watch?v=wCJjiHp0d0w)  
27. Free Dynamic DNS: Getting Started Guide | Support | No-IP Knowledge Base, accessed November 1, 2025, [https://www.noip.com/support/knowledgebase/free-dynamic-dns-getting-started-guide-ip-version](https://www.noip.com/support/knowledgebase/free-dynamic-dns-getting-started-guide-ip-version)  
28. No-IP | Smarter DNS Starts Here, accessed November 1, 2025, [https://www.noip.com/](https://www.noip.com/)  
29. Chose a good free DDNS provider \- Ubiquiti Community, accessed November 1, 2025, [https://community.ui.com/questions/46a8a937-b9f6-447b-a7a8-067fe088e89e](https://community.ui.com/questions/46a8a937-b9f6-447b-a7a8-067fe088e89e)  
30. The best free Dynamic DNS service providers \- IONOS, accessed November 1, 2025, [https://www.ionos.com/digitalguide/server/tools/free-dynamic-dns-providers-an-overview/](https://www.ionos.com/digitalguide/server/tools/free-dynamic-dns-providers-an-overview/)  
31. How to set up port forwarding behind a router or a firewall manually? \- Xlight FTP Server, accessed November 1, 2025, [https://www.xlightftpd.com/help/setup\_behind\_firewall.html](https://www.xlightftpd.com/help/setup_behind_firewall.html)  
32. SSH connection refused on windows ssh server \- Stack Overflow, accessed November 1, 2025, [https://stackoverflow.com/questions/75775182/ssh-connection-refused-on-windows-ssh-server](https://stackoverflow.com/questions/75775182/ssh-connection-refused-on-windows-ssh-server)  
33. Using an FTP Server \- DriveSnapshot.de, accessed November 1, 2025, [http://www.drivesnapshot.de/en/ftp.htm](http://www.drivesnapshot.de/en/ftp.htm)  
34. How to use DriveSnapshot \- YouTube, accessed November 1, 2025, [https://www.youtube.com/watch?v=Uyq\_5EPILbw](https://www.youtube.com/watch?v=Uyq_5EPILbw)  
35. FileZilla Server Tutorial \- Setup FTP Server \- YouTube, accessed November 1, 2025, [https://www.youtube.com/watch?v=\_4a4qSaIIrw](https://www.youtube.com/watch?v=_4a4qSaIIrw)  
36. Setting up folder permissions \- FileZilla Forums, accessed November 1, 2025, [https://forum.filezilla-project.org/viewtopic.php?t=7811](https://forum.filezilla-project.org/viewtopic.php?t=7811)  
37. Network Configuration \- FileZilla Wiki, accessed November 1, 2025, [https://wiki.filezilla-project.org/Network\_Configuration](https://wiki.filezilla-project.org/Network_Configuration)  
38. FTP Passive Port Range Standard? \- firewall \- Super User, accessed November 1, 2025, [https://superuser.com/questions/1299562/ftp-passive-port-range-standard](https://superuser.com/questions/1299562/ftp-passive-port-range-standard)  
39. FileZilla Server Passive Mode, accessed November 1, 2025, [https://filezillapro.com/docs/server/advanced-options/filezilla-server-passive-mode/](https://filezillapro.com/docs/server/advanced-options/filezilla-server-passive-mode/)  
40. Do I need to open router ports to use vsftpd passive mode? : r/selfhosted \- Reddit, accessed November 1, 2025, [https://www.reddit.com/r/selfhosted/comments/ouqj62/do\_i\_need\_to\_open\_router\_ports\_to\_use\_vsftpd/](https://www.reddit.com/r/selfhosted/comments/ouqj62/do_i_need_to_open_router_ports_to_use_vsftpd/)  
41. SSH connection refused: what it is, causes, and 6 effective methods to fix it \- Hostinger, accessed November 1, 2025, [https://www.hostinger.com/tutorials/ssh-connection-refused](https://www.hostinger.com/tutorials/ssh-connection-refused)  
42. sftp connection refused issue \- linux \- Stack Overflow, accessed November 1, 2025, [https://stackoverflow.com/questions/25910131/sftp-connection-refused-issue](https://stackoverflow.com/questions/25910131/sftp-connection-refused-issue)  
43. Unable to open port 22 for SFTP file transfer (Remote Desktop environment) \- Super User, accessed November 1, 2025, [https://superuser.com/questions/1649643/unable-to-open-port-22-for-sftp-file-transfer-remote-desktop-environment](https://superuser.com/questions/1649643/unable-to-open-port-22-for-sftp-file-transfer-remote-desktop-environment)  
44. How to Fix the SSH “Connection Refused” Error \- phoenixNAP, accessed November 1, 2025, [https://phoenixnap.com/kb/ssh-connection-refused](https://phoenixnap.com/kb/ssh-connection-refused)  
45. 6.2 Technical Notes \- Archived Linux @ CERN, accessed November 1, 2025, [https://linux-archive.web.cern.ch/scientific6/docs/rhel/6.2\_Technical\_Notes/](https://linux-archive.web.cern.ch/scientific6/docs/rhel/6.2_Technical_Notes/)  
46. FTP “530 User cannot log in” error and solution \- Microsoft Community Hub, accessed November 1, 2025, [https://techcommunity.microsoft.com/blog/iis-support-blog/ftp-%E2%80%9C530-user-cannot-log-in%E2%80%9D-error-and-solution/364570](https://techcommunity.microsoft.com/blog/iis-support-blog/ftp-%E2%80%9C530-user-cannot-log-in%E2%80%9D-error-and-solution/364570)  
47. FTP error "530 Login authentication failed" \- Stack Overflow, accessed November 1, 2025, [https://stackoverflow.com/questions/10695290/ftp-error-530-login-authentication-failed](https://stackoverflow.com/questions/10695290/ftp-error-530-login-authentication-failed)  
48. Unable to connect FTP: 530 Login incorrect \- Unix & Linux Stack Exchange, accessed November 1, 2025, [https://unix.stackexchange.com/questions/124248/unable-to-connect-ftp-530-login-incorrect](https://unix.stackexchange.com/questions/124248/unable-to-connect-ftp-530-login-incorrect)