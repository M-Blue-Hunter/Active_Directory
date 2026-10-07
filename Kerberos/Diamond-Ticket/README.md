# Diamond Ticket

## Overview

Diamond Ticket is a Kerberos ticket forgery technique that abuses a legitimate Ticket Granting Ticket (TGT) and modifies selected ticket attributes to impersonate another identity.

This simulation is performed in an isolated Active Directory lab to study Kerberos authentication behavior, Windows security telemetry, and SIEM detection opportunities.

The objective of this scenario is not only to reproduce the attack, but also to understand the telemetry generated during each stage and develop reliable detections.

---

## MITRE ATT&CK

| Technique                       | ID        | Description                           |
| ------------------------------- | --------- | ------------------------------------- |
| Steal or Forge Kerberos Tickets | T1558     | Abuse Kerberos authentication tickets |
| Golden Ticket                   | T1558.001 | Forge or manipulate Kerberos TGTs     |

---

## Attack Flow

```text
Compromised Domain Account
          |
          v
     Obtain TGT
          |
          v
 Read / Use Kerberos Key Material
          |
          v
 Modify TGT Attributes
          |
          v
   Diamond Ticket
          |
          v
 Inject Ticket into Session
          |
          v
 Request Service Ticket
          |
          v
 Access Kerberos Service
          |
          v
 Windows Security Telemetry
          |
          v
    SIEM Detection
```

---

## Lab Environment

| Component         | Value                         |
| ----------------- | ----------------------------- |
| Domain            | `dc2.purplelab.local`         |
| Domain Controller | `192.168.56.126`              |
| Attacker          | Kali Linux                    |
| Kerberos Tooling  | Rubeus / Impacket             |
| SIEM              | Splunk                        |
| Environment       | Isolated Active Directory Lab |

---

## Prerequisites

The following are required for this simulation:

* Active Directory domain
* Domain Controller with Kerberos enabled
* Valid domain credentials
* Kerberos authentication
* Windows Security Auditing
* Kerberos-related events forwarded to the SIEM
* Controlled lab environment

---

## Simulation

The complete attack procedure is documented separately.

See:

[`simulate.md`](./simulate.md)

The simulation covers:

1. Obtaining the required Kerberos authentication material
2. Creating the modified TGT
3. Injecting the ticket into the current session
4. Requesting a Kerberos service ticket
5. Accessing a target service
6. Validating the resulting Windows telemetry

---

## Detection

The detection logic is maintained separately from the attack simulation.

See:

[`detection.spl`](./detection.spl)

The detection focuses on identifying anomalous Kerberos activity associated with manipulated or forged tickets.

---

## Expected Telemetry

The following Windows Security events may be relevant during the simulation:

| Event ID | Description                                          |
| -------- | ---------------------------------------------------- |
| 4768     | A Kerberos authentication ticket (TGT) was requested |
| 4769     | A Kerberos service ticket (TGS) was requested        |
| 4624     | An account successfully logged on                    |
| 4672     | Special privileges assigned to a new logon           |

The exact events generated depend on the authentication flow and the service accessed during the simulation.

---

## Detection Hypothesis

A Diamond Ticket may be detectable through inconsistencies between the expected properties of a user's Kerberos authentication and the observed ticket or authentication activity.

Potential detection signals include:

* Unexpected Kerberos authentication behavior
* Unusual ticket lifetime
* Abnormal ticket encryption characteristics
* Inconsistent authentication source
* Unusual account-to-service relationships
* Kerberos activity that does not match the normal behavior of the account
* Correlation between TGT issuance and subsequent service-ticket activity

Detection should rely on multiple signals rather than a single event.

---

## Validation

After running the simulation, validate the following:

```text
Attack Execution
       |
       v
Windows Event Generation
       |
       v
Log Collection
       |
       v
Splunk Search
       |
       v
Detection Trigger
       |
       v
Analyst Validation
```

The detection should be tested against both:

* The malicious simulation
* Legitimate Kerberos authentication

This allows false positives to be identified and the detection logic to be tuned.

---

## References

* MITRE ATT&CK — T1558: Steal or Forge Kerberos Tickets
* MITRE ATT&CK — T1558.001: Golden Ticket

---

## Disclaimer

This project is intended for authorized security research, purple-team exercises, detection engineering, and educational purposes in controlled environments only.
