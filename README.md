# aws-soc-homelab
Production like SOC home lab on AWS with SIEM, hardened AD, network IDS, hybrid identity. (In progress)

![Status](https://img.shields.io/badge/status-in_progress-yellow?style=flat-square)
![AWS](https://img.shields.io/badge/cloud-AWS-orange?style=flat-square&logo=amazonaws)
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh_4.14-blue?style=flat-square)
![Elastic](https://img.shields.io/badge/Elastic_Stack-9.4-005571?style=flat-square&logo=elastic)
![Suricata](https://img.shields.io/badge/IDS-Suricata_8.0-red?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)

#Summary

Production style Security Operations Center built from scratch on AWS, spanning both defense and offense as follows: 
- Segmented multi-VPC network
- Four tool SIEM/IDS detection stack
- A CIS hardened Active Directory domain
- WAF protected application tier
- An isolated offensive lab - GOAD for Domain Controller attack scenarios
- Adversary emulation tooling (Atomic Red Team, Caldera) driving documented purple team attack and detect scenarios mapped to MITRE ATT&CK.

# STATUS: ACTIVELY BUILDING. 

This is a living project -> **I'm documenting it as I build**, including the real problems I hit and how I solved them. The build log in `build-notes/` reflects genuine troubleshooting. Sections marked "planned" in Build Status (below) are next on the roadmap.


# Why I built this

To **learn/practice** core cybersecurity concepts in **offensive/defensive/cloud** security operations end to end: network design, SIEM engineering, detection writing, AD hardening, and the offensive techniques those defenses are meant to stop. I learn fastest by building real systems and debugging real failures, and this repo is the evidence of that process. The same skills can be trasfered to other cloud vendors (Azure/GCP) and tools (Sentinel, Defender), which I'm confident I can pick up quickly.


# Architecture

Three isolated VPCs connected by a Transit Gateway, with an attacker VPC deliberately left unattached for isolation.


```mermaid
flowchart TB

%% =======================
%% Styles (portfolio clean look)
%% =======================
classDef security fill:#fdecea,stroke:#c62828,stroke-width:2px,color:#111;
classDef corporate fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#111;
classDef attacker fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#111;
classDef network fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#111;
classDef service fill:#ede7f6,stroke:#5e35b1,stroke-width:2px,color:#111;
classDef hidden fill:transparent,stroke:transparent,color:transparent;

%% =======================
%% Entry Layer
%% =======================
Kali["Local Kali VM<br/>Offensive Security Testing"]
VPN["AWS Client VPN"]

Kali --> VPN

%% =======================
%% AWS ACCOUNT
%% =======================
subgraph AWS["AWS Account"]

    %% Security VPC
    subgraph SEC["Security VPC<br/>10.0.0.0/16"]

        subgraph SEC_NET[" "]
            direction LR

            SIEM["SIEM Subnet"]

            subgraph SEC_TOOLS[" "]
                direction TB
                WZ["Wazuh"]
                ELK["Elastic"]
                SUR["Suricata IDS"]
            end

        end

        SIEM --> WZ
        SIEM --> ELK
        SIEM --> SUR

    end

    %% Transit Gateway
    TGW["Transit Gateway"]

    %% Corporate VPC
    subgraph CORP["Corporate VPC<br/>10.1.0.0/16"]

        %% First row
        subgraph CORP_NET[" "]
            direction LR
            IT["IT Subnet"]
            FIN["Finance Subnet"]
            SALES["Sales Subnet"]
            DMZ["DMZ Subnet"]
        end

        %% Second row
        subgraph CORP_SERVERS[" "]
            direction LR
            DC["DC01<br/>Hardened AD"]
            DB["Database"]
            MailServer["MailServer"]
        end

        %% Third row
        DVWA["DVWA + Juice Shop"]

    end

    %% Attacker VPC
    subgraph ATT["Attack VPC<br/>10.2.0.0/16<br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/>"]

        GOAD["GOAD<br/>"]

        Spacer --> GOAD
    end

    %% Corporate relationships
    DC --> IT
    DB --> FIN
    MailServer --> SALES
    DVWA --> DMZ

    %% Connectivity
    SEC <--> TGW
    TGW <--> CORP

    GOAD -. No Transit Gateway Attachment .- TGW

    %% AWS Security Services
    SERVICES["CloudTrail • GuardDuty • AWS Config • WAF • VPC Flow Logs"]

end

%% =======================
%% External Access
%% =======================
VPN --> AWS

%% =======================
%% Styling assignment
%% =======================
class SEC security;
class CORP corporate;
class ATT attacker;
class TGW,VPN network;
class SERVICES service;
class Spacer hidden;

%% Make DVWA + Juice Shop match server styling
class DVWA service;
```


See `architecture/` for more info.



# Build status

| 			Area 	   		    |   Status 	        |              Notes 	           |
|---------------------------------------------------|-------------------|----------------------------------|
| AWS account hardening (MFA, IAM, CLI via SSO)     | ✅ Done 		| 	       ---	           |
| Logging baseline (CloudTrail, GuardDuty, Config)  | ✅ Done 		| CIS conformance pack 	           |
| Multi-VPC network + Transit Gateway 		    | ✅ Done 		| Attacker VPC isolated by design  |
| Security groups (least privilege, egress filtered)| ✅ Done 		| 7 tiered SGs 		           |
| Hardened Active Directory (DC01) 		    | ✅ Done 		| CIS baseline — see `docs/`  	   |
| Windows endpoints domain joined 		    | ✅ Done 		| 3 endpoints, tiered OUs          |
| Wazuh SIEM + agents 			            | ✅ Done 		| Custom detection rules           |
| Elastic Stack + Logstash pipelines 		    | ✅ Done 		| CloudTrail + VPC Flow Logs 	   |
| Suricata network IDS + VPC traffic mirroring      | ✅ Done 		| Alerts forwarded to Wazuh 	   |
| Vulnerable web apps (DVWA, Juice Shop) 	    | ✅ Done    | 	       --- 		   |
| Splunk SIEM (HEC + AWS add-on + SPL detections)   | 🔶 In progress        | Enterprise SIEM + SPL queries    |
| Finance DB + mail server 			    | ✅ Done 	| 	       ---		   |
| AWS Client VPN 				    | 📋 Planned 	|              ---	           |
| Hybrid identity (Entra ID + Okta) 		    | 📋 Planned	|              ---	           |
| GOAD offensive lab (isolated VPC)		    | 📋 Planned 	|  	       --- 		   |
| AWS WAF on application load balancer              | 🔶 In progress       | Rules + rate limiting + logging  |
| Adversary emulation - Atomic Red Team             | 📋 Planned        | Atomic tests per MITRE technique |
| Adversary emulation - Caldera                     | 📋 Planned        | Automated attack chains          |
| Purple team scenarios (MITRE ATT&CK mapped)       | 📋 Planned        | Attack -> detect -> harden       |
| Lambda cost scheduler			            | 📋 Planned 	|              --- 		   |
| Documented attack/detection scenarios 	    | 📋 Planned 	|   	       ---		   |
| Terraform IaC					    | 📋 Planned        | Capturing working config as I go |				   |

# Skills demonstrated

- Cloud architecture: multi-VPC segmentation, Transit Gateway routing, NAT/IGW design, least-privilege security groups with egress filtering.

- SIEM engineering: Wazuh deployment + custom rules, Elastic/Logstash ingestion pipelines for AWS log sources, multi-tool correlation.

- Network detection: Suricata IDS with ET Open ruleset, VPC traffic mirroring (VXLAN), alert forwarding into a central SIEM (Wazuh).

- Active Directory security: CIS hardening - Protected Users, AES-only Kerberos, NTLMv1/SMBv1 disabled, LLMNR/NetBIOS off, tiered admin model, LAPS, full audit + PowerShell logging, fine-grained password policies.

- Cloud security posture: CloudTrail, GuardDuty, AWS Config with CIS conformance pack.

- Linux/Windows administration, troubleshooting, and secure by default configuration.

- Offensive & Purple-team: adversary emulation with Atomic Red Team and Caldera, attacks against an isolated GOAD lab (Kerberoasting, AS-REP roasting, DCSync, NTLM relay, Golden Ticket, password spraying, web exploitation on DMZ apps), each mapped to MITRE ATT&CK and paired with the detection that catches it and the hardening that stops it.


# Tech stack

AWS (VPC, EC2, TGW, IAM, CloudTrail, GuardDuty, Config, S3, WAF, AWS CLI);
 Wazuh 4.14;
  Elastic Stack 9.4;
   Suricata 8.0;
    Splunk;
     Atomic Red Team; 
      Caldera;
       MITRE ATT&CK; 
        GOAD;
         Windows Server 2022 (Active Directory + Endpoints);
          Ubuntu 24.04 (SIEM + Company Apps);
           Kali Linux (attack scenarios);
            Terraform;
             GitHub


# Engineering challenges solved

Real problems hit during the build (full detail in `build-notes/`):

- AD domain-join failures: "ERROR_NO_SUCH_DOMAIN" -> traced to the DC locator needing CLDAP over UDP 389, which the security group didn't allow; TCP 389 alone is insufficient.

- VPC DNS design: a domain controller must point DNS at itself and use a forwarder (to the AWS resolver) for external resolution; the VPC DHCP option set must be applied before launching members.

- Logstash ingestion: resolved plugin/Gemfile conflicts, TLS cert permissions, cross-index contamination, and CloudTrail field-mapping errors to get clean AWS log ingestion.

- VPC traffic mirroring: mirrored traffic arrives as VXLAN on UDP 4789 and requires an explicit security-group rule on the monitor interface.

- Tooling pivots: switched the network IDS from Security Onion (now Oracle Linux only) to standalone Suricata when the platform requirements changed.


# Repository structure


| Directory | Contents |
|-----------|----------|
| `architecture/` | design decisions |
| `build-notes/` | phase-by-phase build log with troubleshooting |
| `detections/` | `wazuh-rules/`, `suricata/`, `logstash-pipelines/` |
| `docs/` | deep-dives (AD hardening, etc.) |
| `runbooks/` | incident-response procedures |
| `screenshots/` | evidence of working components |



# Roadmap

Web apps + DB → WAF on ALB → Client VPN → Hybrid Identity → GOAD Offensive lab → Adversary emulation (Atomic Red Team, Caldera) → Documented purple-team scenarios → Full Terraform IaC.

-----------------------------------------------------------------------------------------------------------

*Built and maintained by DH. Notes reflect an active, in-progress learning project.*
