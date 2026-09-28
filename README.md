# SOC Incident — SSH Intrusion & Credential Reuse Investigation

> **A controlled SOC investigation using Wazuh to detect, investigate, correlate and document suspicious SSH authentication activity in a personal cybersecurity lab.**

![SOC](https://img.shields.io/badge/Domain-SOC%20Analysis-blue)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh-purple)
![Platform](https://img.shields.io/badge/Platform-Linux-orange)
![Status](https://img.shields.io/badge/Incident-Closed-success)
![Environment](https://img.shields.io/badge/Environment-Authorized%20Lab-green)

---

## 1. Executive Summary

SOC Incident  is a controlled incident investigation performed inside an authorized personal cybersecurity lab.

The objective was to simulate and investigate suspicious SSH authentication activity using **Wazuh** as the SIEM/XDR platform.

The investigation began with failed SSH authentication attempts against a Linux account. The activity was followed by a successful SSH login. After the session ended, the victim changed the account password.

The simulated attacker subsequently attempted to reuse the old password. The repeated authentication attempts failed, and Wazuh generated multiple authentication-related events and a higher-severity correlation alert.

The investigation demonstrates a practical SOC workflow:

```text
Detection
   ↓
Alert Review
   ↓
Event Correlation
   ↓
Authentication Analysis
   ↓
Timeline Reconstruction
   ↓
Impact Assessment
   ↓
Containment
   ↓
Recovery
   ↓
Documentation
```

---

# 2. Incident Overview

| Field                       | Details                                                        |
| --------------------------- | -------------------------------------------------------------- |
| Incident ID                 | SOC-001                                                        |
| Incident Type               | Suspicious SSH Authentication / Unauthorized Access Simulation |
| Environment                 | Authorized Personal Lab                                        |
| SIEM                        | Wazuh                                                          |
| Manager                     | Ubuntu                                                         |
| Target / Agent              | Kali Linux                                                     |
| Target Hostname             | `KALI_BHAI`                                                    |
| Target IP                   | `192.168.1.61`                                                 |
| Manager IP                  | `192.168.1.50`                                                 |
| Target Account              | `Victim`                                                         |
| Protocol                    | SSH                                                            |
| Initial Activity            | Failed authentication followed by successful login             |
| Post-change Activity        | Repeated attempts using old password                           |
| Initial Failed Attempts     | 2                                                              |
| Later Old-password Attempts | 6 consecutive failures                                         |
| Final Access Status         | No successful access after password change                     |
| Investigation Status        | Closed                                                         |

---

# 3. Lab Architecture

```text
                    SOC INVESTIGATOR
                          │
                          │
                          ▼
                 ┌─────────────────┐
                 │ Ubuntu          │
                 │ Wazuh Manager   │
                 │ 192.168.1.50    │
                 └────────┬────────┘
                          │
                    Wazuh Monitoring
                          │
                          ▼
                 ┌─────────────────┐
                 │ Kali Linux      │
                 │ KALI_BHAI       │
                 │ Wazuh Agent     │
                 │ 192.168.1.61    │
                 └────────┬────────┘
                          │
                          ▼
                     SSH Service
                          │
                          ▼
                    Account: Victim
```

The environment was intentionally created for security testing and investigation.

---

# 4. Incident Scenario

The scenario models a possible SSH account compromise.

An authentication sequence was observed where:

1. An SSH connection was attempted.
2. Two incorrect passwords were supplied.
3. Authentication eventually succeeded.
4. An interactive SSH session was established.
5. Commands were executed inside the session.
6. The session was terminated.
7. The account password was changed by the victim.
8. The previous password was subsequently reused in six consecutive authentication attempts.
9. All six post-change attempts failed.

The important investigative question was:

> **Did the attacker retain access after the victim changed the password?**

The collected evidence indicates that the old credential no longer provided successful authentication after the password change.

---

# 5. Initial Authentication Activity

The first authentication sequence consisted of:

```text
Failed authentication
        ↓
Failed authentication
        ↓
Successful SSH authentication
        ↓
Interactive SSH session
```

Two failed attempts alone should not automatically be classified as brute-force activity.

The successful authentication became more important when correlated with the subsequent interactive activity.

---

# 6. Successful SSH Session
<img width="893" height="912" alt="Screenshot 2026-09-28 115709" src="https://github.com/user-attachments/assets/37d3f29c-eff3-4443-abb1-e9b7a9644b7e" />

After authentication, the session included the following commands:

```bash
ls
mkdir got_ssh
cd got_ssh
echo "hey i am here" > file.txt
```

The activity demonstrates that authentication was not merely an isolated login event.

The authenticated session was capable of:

* Listing directory contents
* Creating a directory
* Entering the created directory
* Creating a file
* Writing content to the file

The created artifact was:

```text
got_ssh/file.txt
```

with the message:

```text
hey i am here
```

This activity was performed inside the controlled lab.

---

# 7. Password Change & Follow-up Activity

After the SSH session, the victim changed the account password.

The simulated attacker subsequently attempted to authenticate using the previous password.

The result:

```text
Old password
    ↓
Authentication failure
    ↓
Old password
    ↓
Authentication failure
    ↓
Old password
    ↓
Authentication failure
    ↓
Old password
    ↓
Authentication failure
    ↓
Old password
    ↓
Authentication failure
    ↓
Old password
    ↓
Authentication failure
```

Total documented post-change attempts:

**6 consecutive failures**

This was the most significant part of the investigation because it demonstrated attempted reuse of a credential that was no longer valid.

---

# 8. Wazuh Detection Evidence

The Wazuh dashboard displayed multiple authentication-related events.

Observed events included:

| Time         | Rule | Level | Description                                 |
| ------------ | ---: | ----: | ------------------------------------------- |
| 11:52:07.513 | 5503 |     5 | PAM: User login failed                      |
| 11:52:09.514 | 5557 |     5 | unix_chkpwd: Password check failed          |
| 11:52:13.524 | 5760 |     5 | sshd: authentication failed                 |
| 11:52:15.531 | 5760 |     5 | sshd: authentication failed                 |
| 11:52:17.532 | 5557 |     5 | unix_chkpwd: Password check failed          |
| 11:52:19.534 | 5760 |     5 | sshd: authentication failed                 |
| 11:52:21.534 | 2502 |    10 | User missed the password more than one time |

<img width="1853" height="935" alt="Screenshot 2026-09-28 115532" src="https://github.com/user-attachments/assets/6ee5ffff-383e-4432-83eb-682718add9d8" />

### Important investigation note

The seven Wazuh rows above must **not** be interpreted as seven separate authentication attempts.

A single failed authentication can produce multiple telemetry events because different components participate in the authentication process.

For example:

```text
Authentication attempt
        │
        ├── PAM
        │
        ├── unix_chkpwd
        │
        └── sshd
               │
               ▼
        Wazuh collects events
```

Therefore, event correlation is required before calculating the actual number of authentication attempts.

---

# 9. Key Wazuh Rules
<img width="1334" height="404" alt="Screenshot 2026-09-28 143109" src="https://github.com/user-attachments/assets/fc20cfab-2587-457b-a72b-a1b875ea82f9" />

## Rule 5503

**Description:**

```text
PAM: User login failed
```

This provides evidence that the Linux PAM authentication mechanism rejected a login.

---

## Rule 5557

**Description:**

```text
unix_chkpwd: Password check failed
```

This indicates that the password validation process failed.

---

## Rule 5760

**Description:**

```text
sshd: authentication failed
```

This directly associates the failed authentication with the SSH service.

---

## Rule 2502

**Level:** 10

**Description:**

```text
syslog: User missed the password more than one time
```

This is particularly useful because it provides correlation around repeated password failures.

---

# 10. Investigation Methodology

The investigation followed a simplified SOC workflow.

### Step 1 — Detection

Review Wazuh alerts for authentication-related activity.

### Step 2 — Identify the Host

The events were associated with:

```text
KALI_BHAI
192.168.1.61
```

### Step 3 — Identify the Account

The investigated account was:

```text
Victim
```

### Step 4 — Identify the Authentication Service

The authentication service was:

```text
SSH / sshd
```

### Step 5 — Correlate Events

PAM, `unix_chkpwd`, SSH and correlation events were compared rather than counting every alert row independently.

### Step 6 — Reconstruct the Session

The successful SSH session was reviewed alongside the commands executed.

### Step 7 — Analyze Credential Reuse

The post-password-change authentication failures were examined.

### Step 8 — Determine Access Status

No successful authentication was observed during the documented six old-password attempts.

---

# 11. Timeline

```text
[Initial authentication]
        │
        ├── Failed password
        │
        ├── Failed password
        │
        └── Successful SSH login
                 │
                 ▼
        Interactive SSH session
                 │
                 ├── ls
                 ├── mkdir got_ssh
                 ├── cd got_ssh
                 └── echo "hey i am here" > file.txt
                 │
                 ▼
              Session ends
                 │
                 ▼
          Victim changes password
                 │
                 ▼
        Old password reused
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
      Fail     Fail      Fail
        │        │        │
        └────────┼────────┘
                 │
          6 consecutive
          failed attempts
                 │
                 ▼
        No successful access
```

---

# 12. Attack Chain

The simulated attack chain can be represented as:

```text
Credential Attempt
       ↓
Successful Authentication
       ↓
SSH Session
       ↓
Command Execution
       ↓
File Creation
       ↓
Password Changed
       ↓
Credential Reuse Attempt
       ↓
Repeated Authentication Failures
       ↓
Detection by Wazuh
```

---

# 13. MITRE ATT&CK Mapping

The activity can be discussed using relevant MITRE ATT&CK concepts.

| Activity                                                 | ATT&CK Concept                               |
| -------------------------------------------------------- | -------------------------------------------- |
| Password guessing / repeated authentication attempts     | T1110 — Brute Force                          |
| Valid credentials used for successful SSH authentication | T1078 — Valid Accounts                       |
| Remote SSH access                                        | T1021.004 — SSH                              |
| Command execution after login                            | T1059 — Command and Scripting Interpreter    |
| File creation during the session                         | Supporting host activity / artifact creation |

### Mapping limitation

MITRE ATT&CK mapping describes the behavior represented by the lab scenario. It does not independently prove the identity, motivation or real-world origin of an attacker.

---

# 14. Impact Assessment

The investigation identified the following activity during the successful session:

* SSH authentication
* Interactive shell access
* Directory listing
* Directory creation
* File creation
* File modification

The created file demonstrates that the authenticated session had the ability to modify data within the account's permitted environment.

No evidence collected for this incident establishes:

* Privilege escalation
* Persistence
* Malware installation
* Data exfiltration
* Destructive activity
* Successful authentication after the password change

Therefore, those activities are not claimed as part of this incident.

---

# 15. Containment

Recommended containment actions demonstrated by the scenario include:

1. Change the affected account password.
2. Invalidate the previously exposed credential.
3. Review active SSH sessions.
4. Review recent authentication logs.
5. Investigate repeated authentication failures.
6. Restrict SSH exposure where appropriate.
7. Review authorized keys if SSH key authentication is enabled.
8. Monitor the account for additional authentication attempts.

---

# 16. Recovery

After changing the password:

```text
Old credential
      ↓
No longer valid
      ↓
Repeated authentication failures
      ↓
Wazuh detection
      ↓
Continued monitoring
```

The six consecutive failed attempts demonstrate that the old password did not provide continued successful access after the credential change.

---

# 17. Detection Opportunities

This investigation highlights several useful SOC detections:

### Detection 1 — Repeated SSH failures

Monitor repeated SSH authentication failures from the same source.

### Detection 2 — Successful login after failures

Correlate:

```text
Multiple failures
       +
Successful authentication
```

This pattern deserves investigation because the successful login follows suspicious authentication activity.

### Detection 3 — Password reuse after credential change

Repeated authentication attempts using an invalidated credential can indicate continued unauthorized access attempts.

### Detection 4 — Authentication followed by suspicious commands

Correlate:

```text
Successful SSH
       +
Interactive shell
       +
Unexpected file activity
```

---

# 18. SOC Analyst Lessons

This incident provided practical experience with:

* SIEM alert investigation
* Wazuh
* Linux authentication
* SSH logs
* PAM
* `sshd`
* Authentication-event correlation
* Timeline reconstruction
* Credential abuse analysis
* Incident documentation
* MITRE ATT&CK mapping
* Basic incident response

A key lesson was:

> **One alert does not necessarily equal one attack action.**

SOC analysts must understand how telemetry is generated before calculating attempt counts or declaring an incident pattern.

---

# 19. Evidence

The repository contains screenshots and supporting documentation for the investigation.

Recommended evidence are attached in :

```text
evidence/
└── screenshots/  
```

Screenshots should be retained in their original form where possible.

Sensitive information unrelated to the investigation should be redacted before public publication.

---

# 20. Incident Classification

| Category                      | Assessment                    |
| ----------------------------- | ----------------------------- |
| Incident                      | Suspicious SSH authentication |
| Environment                   | Authorized personal lab       |
| Initial access                | Successful SSH authentication |
| Credential abuse              | Observed                      |
| Command execution             | Observed                      |
| File modification             | Observed                      |
| Credential reuse              | Attempted                     |
| Post-change successful access | Not observed                  |
| Detection                     | Wazuh                         |
| Investigation                 | Completed                     |
| Status                        | Closed                        |

---

# 21. Final Analyst Conclusion

SOC Incident  successfully demonstrated a complete investigation workflow using Wazuh.

The evidence showed two initial failed authentication attempts followed by a successful SSH login and an interactive session in which commands were executed and a file was created.

Following a password change, the previous password was reused in six consecutive authentication attempts, all of which failed.

Wazuh generated multiple authentication-related events through PAM, `unix_chkpwd`, `sshd`, and a repeated-password correlation rule. The investigation therefore required event correlation rather than simple alert counting.

The incident demonstrates how a SOC analyst can move from raw authentication telemetry to a structured incident timeline, assess the available evidence, document impact, and identify appropriate containment and monitoring actions.

---

# 22. Disclaimer

This investigation was conducted in an authorized personal laboratory environment for cybersecurity learning and SOC analyst skill development.

No unauthorized systems or accounts were targeted.

---
---
# Final Investigation Summary

SIEM: Wazuh

Endpoint:
KALI_BHAI / 192.168.1.61

Account:
Victim

Initial sequence:
2 failures → successful SSH

Session:
Command execution + file creation

Credential change:
Password changed

Follow-up:
6 old-password failures

Result:
No successful post-change authentication observed

Status:
CLOSED

SOC Incident — Completed.
---

## Author

**Ankush Gupta**

Cybersecurity | SOC | VAPT | Security Research

Areas demonstrated in this project:

`Wazuh` `SIEM` `Linux` `SSH` `Incident Response` `Log Analysis` `MITRE ATT&CK` `Threat Detection`
