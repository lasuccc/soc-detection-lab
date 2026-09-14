# Home SOC Lab: SIEM Detection Engineering

A blue-team lab built on AWS to practise detection engineering end to
end: build a monitored network, attack it, and catch the attack with
custom detection rules.

The point of this project isn't standing up VMs. It's writing detection
rules that fire on real attacker behaviour, then proving each one works
by simulating the matching attack.

![Architecture](docs/architecture.svg)

## Stack

| Component | Role |
|-----------|------|
| Wazuh 4.14 | SIEM: manager, indexer, dashboard (Ubuntu 22.04) |
| Windows Server 2022 | Domain controller `lab.local`, monitored endpoint |
| Kali Linux | Attacker box |
| AWS EC2 / VPC | Isolated network, least-privilege security groups |

Everything runs inside one VPC (10.0.0.0/16). Security groups do the
network segmentation an on-prem firewall would handle, and inbound
access is locked to a single home IP rather than the open internet.

## Detections

Each rule was validated by running the matching attack and confirming
the alert fired from endpoint to dashboard.

| Rule ID | Detects | Event IDs | MITRE | Severity |
|---------|---------|-----------|-------|----------|
| 100100 | Brute force (5+ failed logons in 2 min) | 4625 | T1110 | 10 |
| 100200 | Account added to global group (e.g. Domain Admins) | 4728 | T1098 | 12 |
| 100201 | Account added to local group (e.g. Administrators) | 4732 | T1098 | 12 |

Rule definitions with comments are in [`rules/local_rules.xml`](rules/local_rules.xml).

## How it fits together

1. Windows generates a security event (e.g. 4625 on a failed logon).
2. The Wazuh agent ships it to the manager over ports 1514/1515.
3. A built-in Wazuh rule catches the individual event.
4. A custom rule correlates or matches it and raises an alert.
5. The alert surfaces in the dashboard for triage.

## Repo layout

```
rules/
  local_rules.xml            custom detection rules
  enable-audit-policies.ps1  Windows audit config the rules depend on
  simulate-attacks.ps1       attack simulations to validate each rule
docs/
  architecture.svg           network diagram
  incident-report-bruteforce.md   worked incident writeup
screenshots/                 dashboard evidence
```

## Setup outline

1. Create the VPC and three security groups (`sg-wazuh`, `sg-windows`,
   `sg-kali`), each scoped to a home IP for management access.
2. Launch the Wazuh instance (Ubuntu 22.04, t3.medium) and run the
   all-in-one installer.
3. Launch Windows Server 2022, promote to a domain controller
   (`lab.local`), and apply the audit policies in
   [`enable-audit-policies.ps1`](rules/enable-audit-policies.ps1).
4. Install the Wazuh agent on Windows, pointing at the manager's
   private IP.
5. Add the rules in [`local_rules.xml`](rules/local_rules.xml) to the
   manager and restart it.
6. Run [`simulate-attacks.ps1`](rules/simulate-attacks.ps1) and confirm
   each rule fires.

## What I took away from it

The rules working was the easy part. The lessons came from the things
that broke:

- IPv6 routing in the VPC silently killed package downloads until I
  forced IPv4.
- A missing outbound security-group rule dropped traffic with no error
  to explain why.
- A correlation rule that grouped on source IP never fired, because a
  local logon doesn't record one. That distinction (network vs local,
  and what fields each populates) is the difference between a rule that
  works and one that quietly doesn't.

See [`docs/incident-report-bruteforce.md`](docs/incident-report-bruteforce.md)
for a full worked example.

## Roadmap

- Lateral movement detection (Event ID 4624 Type 3, service creation 7045)
- Attack simulation with Atomic Red Team mapped to specific ATT&CK techniques
- Log forwarding from a second endpoint
