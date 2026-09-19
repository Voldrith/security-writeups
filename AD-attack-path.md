# Active Directory Attack Path — Kerberoasting to SYSTEM

**Environment:** Practical lab (simulated AD domain)
**Focus:** Enumeration, credential attacks, ACL abuse, privilege escalation, lateral movement
**Tools used:** Nmap, Impacket, BloodHound, Rubeus, PowerView, Chisel

---

## Summary

This writeup documents a full Active Directory attack path in a simulated lab environment — from initial enumeration through to SYSTEM-level compromise on a domain-joined Windows host, with pivoting across segmented networks.

The chain covered:
1. Network and service enumeration
2. AD user and group enumeration
3. Kerberoasting to recover a service account hash
4. Credential abuse and ACL/permission analysis
5. Lateral movement to a second host
6. Privilege escalation to SYSTEM via SeImpersonatePrivilege
7. Pivoting through segmented networks using tunnelling

---

## 1. Reconnaissance & Enumeration

Initial Nmap scan across the target range identified exposed SMB (445), Kerberos (88), LDAP (389), and RDP (3389) services on the domain controller and a domain-joined host.

    nmap -sC -sV -p- <target>

SMB enumeration revealed the domain name and a list of domain users via null session / guest access.

LDAP enumeration provided:
- Domain user accounts
- Group memberships
- Service Principal Names (SPNs) tied to service accounts

---

## 2. Kerberoasting

With a valid domain user context, I queried for accounts with registered SPNs — these are Kerberoastable.

    impacket-GetUserSPNs <domain>/<user>:<pass> -dc-ip <dc-ip> -request

A service account returned a Kerberos TGS hash. I cracked it offline with Hashcat:

    hashcat -m 13100 hash.txt rockyou.txt

Recovered plaintext credentials for the service account.

---

## 3. ACL & Permission Analysis

Using the recovered service account, I enumerated ACLs across the domain to identify misconfigurations.

Findings:
- The service account had **GenericWrite** over a second user
- That second user had **local admin** on a domain-joined host

This provided a clear escalation path.

---

## 4. Lateral Movement

With the second user's credentials, I authenticated to the domain-joined host via SMB/WinRM and established a foothold.

    impacket-psexec <domain>/<user>:<pass>@<host>

Confirmed `whoami` returned the domain user context — not yet SYSTEM.

---

## 5. Privilege Escalation to SYSTEM

Checked token privileges on the foothold:

    whoami /priv

Found **SeImpersonatePrivilege** enabled. This is exploitable via a potato-family attack (JuicyPotato, PrintSpoofer, or RoguePotato depending on OS version).

Used a potato exploit to coerce a SYSTEM-level token and impersonate it, resulting in:

    nt authority\system

Confirmed SYSTEM access on the domain-joined host.

---

## 6. Pivoting Across Segmented Hosts

The compromised host could reach an internal subnet not directly accessible from the attacker machine. Set up a tunnel using Chisel:

- Chisel server on attacker machine
- Chisel client on compromised host
- Proxychains routed tooling through the tunnel

Used Impacket through the proxy to continue enumeration into the segmented network without direct line of sight.

---

## Impact

This chain demonstrates what a single compromised domain user account can lead to in a misconfigured AD environment:

- Full credential compromise of a service account (Kerberoasting)
- Escalation to domain-joined host local admin via ACL abuse
- SYSTEM-level access on a production-like host
- Reach into segmented internal networks

In a real environment, this would represent a **critical-severity** finding — full domain compromise from an unprivileged starting position.

---

## Remediation Recommendations

**Kerberoasting:**
- Use strong, unique passwords (25+ chars) for service accounts — or use Group Managed Service Accounts (gMSA), which rotate passwords automatically
- Avoid SPNs on accounts with privileged access
- Monitor for TGS requests against service accounts from non-service sources

**ACL abuse:**
- Audit ACLs regularly — especially GenericWrite, GenericAll, WriteDACL, WriteOwner on users and groups
- Apply least-privilege — service accounts should not have write permissions on user objects
- Use tools like BloodHound to visualize attack paths proactively

**SeImpersonatePrivilege:**
- Remove the privilege from service accounts that don't need it
- Keep Windows patched — modern potato exploits depend on unpatched or misconfigured service configurations
- Restrict which accounts can run services

**Segmentation:**
- Enforce network segmentation with host-based firewalls, not just VLANs
- Monitor for internal pivoting — outbound traffic from workstations to internal subnets is a red flag
- Deploy EDR with behavioural detection for tunnelling tools (Chisel, Ligolo, etc.)

---

## Lessons Learned

- **Kerberoasting is still alive.** It only requires one valid domain user — no admin needed.
- **ACLs are the quiet attack path.** Most AD compromises move laterally through permission misconfigurations, not exploits.
- **SeImpersonatePrivilege is a gift.** Any service account with it is effectively SYSTEM if you can land a foothold.
- **Segmentation isn't real unless it's enforced.** VLANs alone don't stop pivoting — you need host-level controls.

---

## Tools Referenced

Nmap · Impacket · Hashcat · BloodHound · PowerView · Rubeus · Chisel · Proxychains
