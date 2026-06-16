# Architecture & Design Decisions

## Network topology

| VPC | CIDR | Purpose | Internet path |
|-----|------|---------|---------------|
| Security | 10.0.0.0/16 | SIEM/IDS tooling (Wazuh, Elastic, Suricata) | Own IGW |
| Corporate | 10.1.0.0/16 | AD, Endpoints, DMZ apps | NAT → IGW |
| Attacker | 10.2.0.0/16 | GOAD Offensive lab  | Own IGW, **no TGW** |

## Subnets
- **Security:** SIEM `10.0.1.0/24`, Mgmt `10.0.2.0/24`
- **Corporate:** IT `10.1.1.0/24`, Finance `10.1.2.0/24`, Sales `10.1.3.0/24`, DMZ `10.1.4.0/24` + `10.1.5.0/24` (ALB needs 2 AZs)

## Key IPs
- DC01 (domain controller): `10.1.1.10`
- Wazuh `10.0.1.10` · Elastic `10.0.1.20` · Suricata `10.0.1.40`

## Core design decisions

**1. Attacker VPC is isolated by omission.** It has an internet gateway but **no Transit Gateway attachment**, so it cannot route to Corporate or Security. The only access path to GOAD is from an external Kali host.

2. Defense and offense are physically separated for the Domain Controller. The hardened DC01 (Corporate) is the *defensive showcase*. All offensive DC activity targets the separate, intentionally vulnerable GOAD lab.

3. Security VPC is not domain-joined. SIEM/IDS hosts stay independent of AD so that if the domain is compromised, the monitoring stack remains trustworthy.

4. Least-privilege, egress filtered security groups. Seven tiered Security Groups; Endpoints have explicit ingress-egress allow lists.

5. Centralized DNS through the domain controller. The Corporate VPC DHCP option set points all members at DC01 and forwards external queries to the AWS resolver. This dependency means DC01 must be running for any Corporate host to resolve DNS.

## Detection coverage (layered)

| Layer | Tool | Sees |
|-------|------|------|
| Host/endpoint | Wazuh agents | Windows events, Sysmon, file integrity |
| Network | Suricata + VPC mirroring | packet-level IDS on all corporate traffic |
| Cloud | Elastic + Logstash | CloudTrail API activity, VPC Flow Logs |
| Application | AWS WAF | L7 web attacks |

## Simplifications (vs. a real enterprise)
- Web apps, Finance DB, and mail server are standalone; production would AD-integrate them for centralized auth.
- Single-node SIEM instances; production would cluster for HA.
- Kibana/dashboards served over HTTP internally; production would enable TLS end to end.

