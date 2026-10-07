# Sapphire Ticket

## Overview

Sapphire Ticket is a Kerberos ticket forgery technique that combines a legitimate Kerberos TGT with an S4U2Self + U2U exchange to obtain the PAC of a target identity and use it to create a modified Kerberos ticket.

Unlike a traditional Golden Ticket, the attack can leverage a legitimate KDC-issued ticket and the PAC of an existing privileged account.

This simulation is performed in an isolated Active Directory lab to study Kerberos authentication behavior, Windows security telemetry, and SIEM detection opportunities.

The objective is not only to reproduce the attack, but also to understand the telemetry generated throughout the attack chain and develop reliable detection logic.

---

## MITRE ATT&CK

| Technique                       | ID        | Description                                         |
| ------------------------------- | --------- | --------------------------------------------------- |
| Steal or Forge Kerberos Tickets | T1558     | Abuse Kerberos authentication tickets               |
| Golden Ticket                   | T1558.001 | Forge or manipulate Kerberos authentication tickets |
| Kerberos Service Tickets        | T1558.003 | Abuse Kerberos service tickets                      |

---

## Attack Flow

```text
Compromised Domain Account
          |
          v
     Obtain TGT
          |
          v
 S4U2Self + U2U Request
          |
          v
 Obtain Target User PAC
          |
          v
   Customize Ticket
          |
          v
     Sapphire Ticket
          |
          v
 Request Service Ticket
          |
          v
   Access Kerberos Service
          |
          v
      DCSync
          |
          v
Directory Replication
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
| Domain            | `lab.example.local`           |
| Domain Controller | `dc01.lab.example.local`      |
| Attacker          | Kali Linux                    |
| Kerberos Tooling  | Impacket                      |
| Target Service    | CIFS / SMB                    |
| SIEM              | Splunk                        |
| Environment       | Isolated Active Directory Lab |

---

## Prerequisites

The following are required for this simulation:

* Active Directory domain
* Domain Controller with Kerberos enabled
* Valid domain credentials
* Kerberos authentication
* Impacket
* Windows Security Auditing
* Directory Service auditing for DCSync detection
* Kerberos-related events forwarded to the SIEM
* Controlled laboratory environment

---

## Simulation

The complete attack procedure is documented separately.

See:

[`simulate.md`](./simulate.md)

The simulation covers:

1. Obtaining the required Kerberos authentication material
2. Obtaining a legitimate TGT
3. Requesting the target user's PAC through S4U2Self + U2U
4. Creating the Sapphire Ticket
5. Requesting a Kerberos service ticket
6. Accessing an SMB service using Kerberos authentication
7. Validating privileged directory replication access
8. Analyzing the resulting Windows telemetry

---

## Detection

The detection logic is intentionally kept private in this repository.

For access to the Splunk detection logic, correlation rules, or additional detection engineering details related to this technique, please feel free to contact me.

I am also open to discussing detection methodology, telemetry analysis, and improvements to the detection approach.

---

## Expected Telemetry

The following Windows Security events may be relevant during the simulation:

| Event ID | Description                                              |
| -------- | -------------------------------------------------------- |
| 4768     | A Kerberos authentication ticket (TGT) was requested     |
| 4769     | A Kerberos service ticket (TGS) was requested            |
| 4624     | An account successfully logged on                        |
| 4672     | Special privileges assigned to a new logon               |
| 4662     | An operation was performed on an Active Directory object |
| 5145     | A network share object was checked for access            |

For the DCSync stage, **Event ID 4662** can provide important telemetry when the appropriate directory-service auditing is configured.

---

## Detection Hypothesis

Sapphire Ticket activity may be detectable by correlating multiple authentication and directory-access signals rather than relying on a single event.

Potential detection signals include:

* Unusual Kerberos authentication behavior
* S4U-related authentication activity
* Unexpected account-to-service relationships
* Authentication involving a privileged identity from an unusual source
* TGT and TGS activity associated with unexpected identities
* Abnormal Kerberos service access
* SMB authentication followed by privileged directory access
* Directory replication activity originating from an unexpected host
* 4662 events associated with replication-related permissions
* Authentication behavior inconsistent with the normal activity of the account

A strong detection should correlate the attack chain and avoid relying on a single indicator.

---

## Validation

After running the simulation, validate the following:

```text
Attack Execution
       |
       v
Kerberos Authentication
       |
       v
Windows Event Generation
       |
       v
Log Collection
       |
       v
Splunk
       |
       v
Detection Trigger
       |
       v
Analyst Validation
```

The detection should be tested against both:

* The malicious simulation
* Legitimate Kerberos and directory replication activity

This allows false positives to be identified and the detection logic to be tuned.

---

## References

* MITRE ATT&CK — T1558: Steal or Forge Kerberos Tickets
* MITRE ATT&CK — T1558.001: Golden Ticket
* MITRE ATT&CK — T1558.003: Kerberos Service Tickets

---

## Disclaimer

This project is intended for authorized security research, purple-team exercises, detection engineering, and educational purposes in controlled environments only.

```
```
