# Build Notes

A phase-by-phase log of building the lab including the real problems I hit and the solutions to them.

**Legend:** ✅ done · 🔶 in progress · 📋 planned

---

## Phase 0 — AWS Account & Access ✅
- Root MFA enabled; dedicated IAM admin user with MFA; root locked away.
- AWS CLI authenticated via `aws login` (SSO temporary credentials) instead of long-lived access keys -> no static secrets on disk.
- **Lesson:** SSO temp creds = Better security hygiene than the access-key default.

## Phase 1 — S3 Log Bucket ✅
- Central versioned, encrypted, private bucket for CloudTrail / Config / Flow Logs, with a bucket policy granting only those services read/write access.

## Phase 2 — Cloud Logging Baseline ✅
- CloudTrail (multi-region, management + data + insight events), GuardDuty (S3 + malware protection), AWS Config with the CIS conformance pack.
- High-severity GuardDuty findings routed via an **EventBridge** rule to SNS.
- **Lesson:** GuardDuty findings are *events*, not metrics. EventBridge is the correct tool, not a CloudWatch metric alarm (the two are often conflated because EventBridge was formerly "CloudWatch Events").

## Phase 3 — Network Foundation ✅
- Three VPCs (Security / Corporate / Attacker), subnets per tier, IGWs, a single NAT in Corporate, and a Transit Gateway attaching **only** Security + Corporate.
- **Attacker VPC has no TGW attachment** — isolated by design.
- **Lesson — Security VPC internet egress:** routing Security's outbound traffic through Corporate's NAT via the TGW requires custom TGW route tables (to be revised); For the moment I gave Security VPC its own IGW with lockdown inbound connections.
- **Lesson — DHCP option set ordering:** apply the VPC DHCP option set (DNS → DC01) *before* launching member instances, so they pick up correct DNS at boot.

## Phase 4 — Security Groups ✅
- Seven least-privilege SGs, each scoped to a tier. Corporate endpoints use an **explicit ingress-egress allow-list** (SIEM, DNS, updates, internal lateral). No full outbound permited.
- A pre-staged **isolation SG** allows one-click incident containment (SIEM telemetry only).
- **Lesson:** egress filtering can be used as a detection feature; an attacker's C2 callback to a random internet port is *blocked and logged*. Visible as a REJECT in VPC Flow Logs.

## Phase 5 — Private Access (no bastion) ✅
- EC2 Instance Connect Endpoints for SSH/RDP to private instances; no public IPs, no bastion host.
- **Lesson:** for RDP over an EICE tunnel, the target's SG must allow the connection from the **endpoint's own private IP**. Added new rule.

## Phase 6 — AWS WAF ✅ (ALB association pending)
- Regional Web ACL: AWS managed rule groups + a rate-based rule, all metrics + logging on.
- Managed groups set to **Count** initially so attacks remain visible end-to-end before switching to Block — otherwise the IDS/SIEM never sees the attack traffic during testing.

## Phase 7 — Hardened Active Directory (DC01) ✅
The defensive centerpiece. Windows Server 2022, static `10.1.1.10`, forest `corp.soclab.local`.
- CIS hardening: **Protected Users**, **AES-only Kerberos (RC4 disabled)**, **NTLMv1 disabled**, **SMBv1 disabled + SMB signing required**, **LLMNR + NetBIOS disabled**, full audit policy, **PowerShell script-block/module/transcript logging**, command-line in 4688, fine-grained password policies (25-char service accounts), **native Windows LAPS**, print spooler disabled, LDAP signing required, account lockout policy.
- Tiered OU model (Tier 0/1/2) with a separate Tier-0 admin account.
- **Lesson — DC DNS:** a domain controller must point DNS at **itself**; a stale AWS resolver entry (`10.x.0.2`) breaks its own SRV-record resolution. Add a **DNS forwarder** (to the AWS resolver) so it still resolves external names.
- **Lesson — LAPS:** Server 2022 ships **Windows LAPS natively**; installing the legacy gallery module conflicts with it. Use the built-in cmdlets.

## Phase 8 — Endpoints Domain-Joined ✅
- Three Windows endpoints joined into the respective department OUs.
- **Lesson — the multi-day blocker:** domain joins failed with `ERROR_NO_SUCH_DOMAIN` even though DNS resolved and TCP/389 was open. Root cause: the **DC locator uses CLDAP over UDP 389**, which the security group didn't allow. **TCP 389 alone was not enough** because the join's discovery ping is UDP. Adding UDP 389 to the SG fixed it instantly.
- **Lesson — hardening order:** harden the DC *after* endpoints are joined; pre-hardening (NTLM/SMB-signing/Protected Users) can interfere with the join.
- **Lesson — Protected Users + jump-host RDP:** members of Protected Users can't authenticate over a loopback-tunnel RDP path (NLA falls back to NTLM, which Protected Users forbids → `STATUS_ACCOUNT_RESTRICTION`). Removed the built-in Administrator from Protected Users for break-glass access; In production a Privileged Access Workstation would be used.

## Phase 9 — Wazuh SIEM ✅
- Wazuh 4.14.5 all-in-one (manager + indexer + dashboard), static `10.0.1.10`, custom detection rules (Kerberoasting, AS-REP, DCSync, PtH, password spray, Golden Ticket, NTLM relay, GPO abuse, etc.).
- Kept the installer's default agent-auth ciphers (which already exclude RC4/3DES/MD5).

## Phase 10 — Wazuh Agents ✅
- Agents on all four Windows hosts; Windows event channels (Security, Sysmon, PowerShell) pushed centrally via the manager's shared `agent.conf`.
- **Lesson:** agent **groups must be created on the manager *before*** installing agents that reference them, or registration fails with `Invalid group`.

## Phase 11 — Elastic Stack + Logstash ✅
- Elasticsearch + Kibana + Logstash 9.4; Logstash pipelines ingest **CloudTrail** and **VPC Flow Logs** from S3.
- **Lesson — keep TLS on:** kept Elasticsearch's auto-generated TLS enabled; clients just use `https://` + the Certificate Authority. Less secure to disable it on a security platform.
- **Lesson — Logstash cert perms:** Logstash runs as its own user and can't read Kibana's Certificate Authority file; gave it a readable copy.
- **Lesson — plugin conflict:** the CloudWatch-Logs input plugin conflicts with the bundled AWS integration (and corrupted the Gemfile on a failed install). Skipped it; WAF logs will route via S3 instead.
- **Lesson — clean ingestion:** split the pipelines by a metadata tag so events only land in their own index; dropped `NODATA/SKIPDATA` flow-log records and guarded numeric conversions (the `-` placeholder broke `BigDecimal`); stripped CloudTrail's `requestParameters`/`responseElements` to stop Elasticsearch field-mapping errors.

## Phase 12 — Suricata Network IDS ✅
- Standalone Suricata 8.0 with the ET Open ruleset (~50k rules), monitor interface `ens6`, fed by **VPC Traffic Mirroring** from all corporate endpoints; alerts forwarded into Wazuh via `eve.json`.
- **Lesson — tooling pivot:** Security Onion now requires Oracle Linux 9 (no longer Ubuntu), so I switched to standalone Suricata (same engine). Ligher footprint.
- **Lesson — VXLAN:** mirrored traffic arrives encapsulated as VXLAN on **UDP 4789** and is dropped unless the monitor interface's SG explicitly allows it.
- **Lesson — agent enrollment across VPCs:** the Wazuh manager SG must allow 1514/1515 from the **Security VPC** range too — the Suricata box enrolls from the Security side, not Corporate.

## Phase 13 — Corporate Applications 🔶
- DVWA + OWASP Juice Shop on a DMZ host (public-facing, behind WAF/ALB — pending).
- **Lesson:** DVWA's setup uses `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`, which modern MySQL rejects — removed `IF NOT EXISTS` (two occurrences) to let the schema build.
- **Observation:** within minutes of exposing the public DMZ IP, automated scanners began probing (Shellshock/path-traversal). Real-world illustration of why the WAF + IDS layers exist.
- **Remaining:** Finance DB (PII), mail server, WAF→ALB attachment, WAF-logs→S3 Logstash pipeline.

---

## Recurring operational lessons
- **DC01 must be running** for any Corporate host to resolve DNS (internal + forwarded external) — it's the VPC's DNS server. A stopped DC silently breaks `apt`/joins/everything DNS.
- **Security groups are a recurring root cause** — most "won't connect" issues traced to a missing port/source (UDP 389 for joins, UDP 4789 for mirroring, 1514/1515 cross-VPC for agents). Check the SG before anything else.
- **Double check tool documentation** — several upstream tools had minor changes (Security Onion OS requirement, Windows LAPS being native, DVWA SQL syntax, Elastic TLS defaults).

## Cost approach
Built manually first (capturing working config), then teardown of expensive rebuildable resources (NAT, TGW attachments, EICE, eventually GOAD/Suricata) between sessions; stateful instances stopped, not destroyed. Terraform (planned) will codify the ephemeral layer for one-command stand-up/teardown.
EOF
