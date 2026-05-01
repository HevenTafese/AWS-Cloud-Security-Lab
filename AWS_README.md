# AWS Cloud Security Lab
### GuardDuty · CloudTrail · Security Hub · S3 Honeypot · EC2 Attack Simulation

A hands-on cloud threat detection environment built on AWS, simulating real-world attack scenarios and detecting them using native AWS security services. The lab covers the full defensive lifecycle: logging, detection, aggregation, attack simulation and honeypot deployment.

---

## Overview

Most cloud security resources explain what GuardDuty does. This lab demonstrates it working. I deployed a Windows Server EC2 instance in AWS eu-west-2, configured CloudTrail to log all account activity, enabled GuardDuty as the detection engine, and used Kali Linux to run reconnaissance and brute force attacks against the live instance. I then deployed an S3 honeypot bucket containing fake credentials and accessed it from Kali to trigger a high-severity GuardDuty finding. Security Hub aggregated every finding into a single dashboard.

The architecture mirrors what a cloud SOC analyst works with in a production AWS environment.

---

## Architecture

```
Kali Linux (Attacker)
        |
        | Nmap reconnaissance · Hydra RDP brute force · S3 credential access
        |
        ▼
EC2 Windows Server (Target)          S3 Honeypot Bucket
        |                                    |
        | RDP port 3389 exposed              | Fake credentials exposed
        |                                    |
        ▼                                    ▼
    CloudTrail (Activity logging — eu-west-2, multi-region)
        |
        ▼
    GuardDuty (Threat detection engine — 30-day trial)
        |
        ▼
    Security Hub (Aggregated findings dashboard)
```

---

## Lab Components

| Component | Service | Purpose |
|---|---|---|
| Activity logging | AWS CloudTrail | Records every API call and account action across all regions |
| Threat detection | AWS GuardDuty | Analyses CloudTrail logs and detects suspicious behaviour automatically |
| Findings dashboard | AWS Security Hub | Aggregates GuardDuty findings with severity scoring and compliance checks |
| Cloud target | AWS EC2 (Windows Server 2022, t3.micro) | Attack target representing a corporate cloud endpoint |
| Honeypot | AWS S3 bucket with public policy | Decoy bucket containing fake credentials to attract and detect attackers |
| Attacker | Kali Linux (VirtualBox) | Runs Nmap, Hydra and curl against the AWS environment |

---

## Phases

### Phase 1 — Logging Infrastructure

CloudTrail was configured as a multi-region trail covering eu-west-2 with logs delivered to a dedicated S3 bucket. This provides the activity record that GuardDuty analyses.

![CloudTrail active and logging](docs/screenshots/cloudtrail.png)

### Phase 2 — Threat Detection

GuardDuty was enabled with S3 Protection active. It began monitoring CloudTrail logs, DNS queries and VPC flow logs immediately with no additional configuration required.

![GuardDuty successfully enabled](docs/screenshots/guardduty-enabled.png)

### Phase 3 — Cloud Target Deployment

A Windows Server 2022 EC2 instance was launched in eu-west-2 with RDP access enabled. An IAM role with SSM permissions was attached and the instance was connected to via Remote Desktop.

![EC2 instance launched](docs/screenshots/ec2-launched.png)

![Windows Server running in AWS](docs/screenshots/windows-server.png)

### Phase 4 — Attack Simulation

Two attacks were run from Kali Linux against the EC2 instance.

**Reconnaissance:** Nmap service scan against the public IP confirmed RDP on port 3389.

```bash
nmap -Pn -sV <EC2-PUBLIC-IP>
```

**Brute force:** Hydra ran a credential attack against RDP using the rockyou wordlist with 14 million attempts.

```bash
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt rdp://<EC2-PUBLIC-IP> -t 4 -V
```

GuardDuty detected both attacks in real time.

![GuardDuty detecting live Hydra brute force](docs/screenshots/attack-detection.png)

### Phase 5 — Honeypot Deployment

An S3 bucket named `soc-lab-backup-credentials` was created with a public read policy and a fake credentials file uploaded containing plausible but fabricated usernames and passwords. The bucket was designed to look like an exposed backup left by a careless administrator.

Kali accessed the bucket directly:

```bash
curl https://soc-lab-backup-credentials.s3.eu-west-2.amazonaws.com/password.txt
```

GuardDuty raised a **High severity** finding immediately: `Amazon S3 Public Anonymous Access was granted for the S3 bucket soc-lab-backup-credentials`.

![Honeypot triggered — High severity GuardDuty finding](docs/screenshots/honeypot-finding.png)

### Phase 6 — Security Hub Aggregation

Security Hub consolidated all GuardDuty findings into a single dashboard with severity classification. By the end of the session the environment had generated 20+ findings across Critical, High, Medium and Low categories.

![Security Hub findings dashboard](docs/screenshots/security-hub-results.png)

---

## Findings Generated

| Finding | Severity | Source |
|---|---|---|
| Amazon S3 Public Anonymous Access granted | High | S3 Honeypot access from Kali |
| API DescribeMetricFilters invoked using root credentials | Low | Account activity |
| API GetInstanceProfile invoked using root credentials | Low | Account activity |
| API ListObjects invoked using root credentials | Low | Account activity |
| Amazon S3 Block Public Access disabled | Low | Honeypot configuration |

---

## Tools Used

| Tool | Version | Purpose |
|---|---|---|
| AWS GuardDuty | Native | Automated threat detection |
| AWS Security Hub | Native | Findings aggregation |
| AWS CloudTrail | Native | Activity logging |
| AWS EC2 | Windows Server 2022 t3.micro | Cloud attack target |
| AWS S3 | Native | Honeypot deployment |
| Kali Linux | 2024.x | Attack simulation |
| Nmap | 7.98 | Reconnaissance |
| Hydra | 9.6 | RDP brute force |

---

## Repository Structure

```
aws-cloud-security-lab/
├── README.md
├── configs/
│   ├── s3-honeypot-policy.json      # Bucket policy used for honeypot
│   └── iam-ssm-role-policy.json     # IAM role attached to EC2 instance
└── docs/
    └── screenshots/
        ├── cloudtrail.png
        ├── guardduty-enabled.png
        ├── ec2-launched.png
        ├── windows-server.png
        ├── attack-detection.png
        ├── honeypot-finding.png
        └── security-hub-results.png
```

---

## Key Observations

GuardDuty detected honeypot access faster than the brute force findings. The S3 anomaly triggered a high-severity alert within seconds of the curl request while the RDP brute force findings took several minutes to propagate through CloudTrail into GuardDuty. This reflects how AWS prioritises data exfiltration indicators over network-level attacks in its detection logic.

The root credential usage findings appeared without any deliberate attack. Simply navigating the AWS console as the root user generated low-severity alerts, which demonstrates why production environments use IAM users rather than root accounts for day-to-day operations.

---

## Cost

Every service used falls within the AWS free tier or trial period. GuardDuty provides a 30-day free trial. EC2 t3.micro is covered under the 12-month free tier. The EC2 instance was terminated after the lab was complete.

Total cost: £0

---

## References

- [AWS GuardDuty Documentation](https://docs.aws.amazon.com/guardduty)
- [AWS Security Hub Documentation](https://docs.aws.amazon.com/securityhub)
- [AWS CloudTrail Documentation](https://docs.aws.amazon.com/cloudtrail)
- [MITRE ATT&CK Cloud Matrix](https://attack.mitre.org/matrices/enterprise/cloud/)

---

*All attacks were conducted against resources owned and controlled within this AWS account. No external systems were targeted.*
