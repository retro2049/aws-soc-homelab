# Detections

This is the detection content I wrote for the lab. Right now two layers are live: Wazuh rules for host/AD activity, and Suricata for network traffic. Splunk SPL and Sigma exports are pending.

## Wazuh rules

I wrote 16 custom rules on top of Wazuh's built-in set. They focus on the things that actually matter in an AD environment: credential theft, lateral movement, and ransomware behaviour. Each is mapped to a MITRE technique.

| Rule | Level | What it catches | MITRE |
|------|-------|-----------------|-------|
| 100001 | 12 | Kerberoasting (RC4 service ticket requested) | T1558.003 |
| 100015 | 12 | AS-REP roasting (TGT requested with no pre-auth) | T1558.004 |
| 100002 | 14 | Pass-the-Hash (NTLM network logon) | T1550.002 |
| 100020 | 15 | DCSync (replication rights used) | T1003.006 |
| 100003 | 10 | Suspicious PowerShell (encoded commands, downloads) | T1059.001 |
| 100004 | 15 | LSASS access (credential dumping) | T1003.001 |
| 100005 | 12 | Brute force followed by a successful login | T1110 |
| 100016 | 12 | Password spray (many users failing from one IP) | T1110.003 |
| 100006 | 12 | New Windows service installed (lateral movement) | T1021.002 |
| 100007 | 12 | New user account created | T1136.001 |
| 100008 | 13 | User added to local Administrators | T1078 |
| 100010 | 15 | Shadow copies deleted (ransomware prep) | T1490 |
| 100011 | 15 | Mass file changes (ransomware encrypting) | T1486 |
| 100021 | 14 | Golden Ticket (abnormal TGT lifetime) | T1558.001 |
| 100022 | 13 | NTLM relay (odd short workstation name) | T1557.001 |
| 100023 | 13 | GPO modified | T1484.001 |

A few notes:

- **DCSync (100020)** keys on the three replication GUIDs in Event 4662. Those are what an attacker has to use, so it's hard to evade.
- **Password spray vs brute force** — spray is many different users from one source (100016); brute force is the same user hammered until it works (100005). Different shapes, different rules.
- Some of these only fire because of telemetry I turned on while hardening the DC — command line logging in 4688, PowerShell script-block logging, and Sysmon. Without that, the events never reach Wazuh in the first place.

The full rule file is in `wazuh-rules/local_rules.xml`.

## Suricata

Suricata 8.0 running the "Emerging Threats Open" ruleset (~50k rules). It sniffs a copy of all corporate endpoint traffic via VPC traffic mirroring, and its alerts get forwarded into Wazuh so I can see host and network detections in one place. This is the network layer; it catches things host agents can't, like scans and web exploitation on the wire.

Config and enabled sources are in `suricata/`.

## Logstash pipelines

These pull AWS logs from S3 into Elasticsearch: CloudTrail and VPC Flow Logs. I added tags while parsing to make hunting easier: risky IAM actions, blocked connections, sensitive ports, and traffic to/from my Kali box. Configs are in `logstash-pipelines/`.

## Still to come (in the build plan)

- Splunk SPL detections + forwarding Wazuh alerts in over HEC.
- Sigma rule exports.
