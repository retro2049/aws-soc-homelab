# DC01 — Active Directory Hardening

DC01 is the defensive showcase of the lab. It's a Windows Server 2022 domain controller (`corp.soclab.local`) that I hardened to a CIS baseline. The point of this box is to show what a *defended* AD looks like. The offensive DC work happens against a separate, deliberately-broken AD (GOAD) and the comparison between the two is the whole story: here's what's broken in the wild, here's what I build to fix it.

Everything below was applied by hand so I have a general understanding of the controls.

## What I hardened, and why

**Protected Users group** — Put the privileged accounts in it. Members can't use NTLM, can't use RC4, and their credentials aren't cached, which shuts down a lot of credential-theft and replay attacks.

**AES-only Kerberos (RC4 disabled)** — Set supported encryption types to AES only. This is what makes Kerberoasting impractical: RC4 service tickets crack fast offline, AES ones don't.

**NTLMv1 disabled** — Forced NTLMv2 only. NTLMv1 is trivially crackable and abused in relay attacks.

**SMBv1 disabled + SMB signing required** — Killed the legacy protocol behind WannaCry/EternalBlue, and signing blocks SMB relay.

**LLMNR and NetBIOS disabled** — These are the name-resolution protocols that Responder poisons to capture hashes. Turning them off removes that whole attack path. (AD doesn't need them — DNS handles resolution.)

**Full audit policy** — Enabled success/failure auditing across logon, Kerberos, account/group management, directory service access + changes + replication, privilege use, and process creation. This is what actually feeds the SIEM — without it, the detections have nothing to detect.

**Command-line logging in Event 4688** — Process creation events include the full command line, not just the binary name. Huge for catching what an attacker actually ran.

**PowerShell logging** — Script-block, module, and transcript logging all on. Catches encoded/obfuscated PowerShell, which is most of what gets used post-exploitation.

**Fine-grained password policies** — A 25-character policy for service accounts and a separate tighter one for Tier-0 admins. Long service-account passwords are the other half of making Kerberoasting harder.

**Windows LAPS** — Randomizes and rotates the local administrator password on each machine. This breaks lateral movement — a stolen local-admin hash from one box is useless on the next.

**Print Spooler disabled** — PrintNightmare mitigation. A DC has no reason to run the spooler.

**LDAP signing required** — Blocks unsigned/anonymous LDAP binds and LDAP relay.

**Account lockout policy** — Lockout after repeated failures, which blunts brute force and slows password spraying.

## Tiered admin model

Built a Tier 0 / 1 / 2 OU structure with a dedicated Tier-0 admin account separate from any day-to-day user. The idea is that high-privilege accounts never log into low-trust machines, so a compromised workstation can't hand over the domain.

## How this maps to attacks

| Control | Attack it blunts |
|---------|------------------|
| AES-only Kerberos + long service passwords | Kerberoasting |
| Protected Users + LAPS | Pass-the-Hash / lateral movement |
| LLMNR/NetBIOS off + SMB signing | Responder poisoning + NTLM relay |
| NTLMv1 off | Hash cracking / relay |
| LDAP signing | LDAP relay, anonymous enumeration |
| Lockout policy | Brute force, password spray |
| Audit + cmdline + PowerShell logging | Detection of everything post-compromise |

## Things I learned the hard way

- **A DC must point DNS at itself.** A leftover AWS-resolver entry broke its own SRV-record resolution and stopped clients joining. It also needs a DNS forwarder to the AWS resolver so it can still resolve external names.
- **Domain joins need UDP 389, not just TCP.** Joins kept failing with `ERROR_NO_SUCH_DOMAIN` until I realized the DC locator uses CLDAP over UDP 389 — the security group only allowed TCP.
- **Harden after joining, not before.** Several of these controls (Protected Users, SMB signing, NTLM) interfere with the join itself, so order matters.
- **Protected Users breaks jump-host RDP.** Because those accounts can't fall back to NTLM, RDP over a loopback tunnel fails. Had to keep the built-in Administrator out of Protected Users as a break-glass account — in a real environment you'd use a proper privileged-access workstation instead.
- **Windows Server 2022 has LAPS built in.** Installing the old gallery module conflicts with the native one — use the built-in cmdlets.

## Known simplification
The break-glass Administrator account sits outside Protected Users for jump-host access. A production build would replace that with a Kerberos-capable privileged-access workstation.
