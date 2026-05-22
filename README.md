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

### Nmap Scan

```bash
nmap -sS -sV -p- <TARGET_IP> -oN full_scan.txt
