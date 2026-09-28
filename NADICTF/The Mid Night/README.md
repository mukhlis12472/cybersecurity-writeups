# The Mid Night — Network Forensics CTF Write-Up

## Challenge Information

| Category | Details |
|---|---|
| Challenge | The Mid Night |
| Category | Digital Forensics / Network Forensics |
| Points | 500 |
| Difficulty | Medium |
| Tools | Wireshark |
| Evidence | `capture.pcapng`, `sslkeylog.txt` |
| Status | Solved |

---

## Challenge Description

> An employee at Aventix Dynamics resigned abruptly.
>
> Aventix's DLP monitoring logs TLS session keys as a standard compliance measure. You as an analyst get the network capture and the keylog, and have to figure out what the employee took with them on the way out.

Two files were provided:

```text
capture.pcapng
sslkeylog.txt
```

The objective was to analyze the employee's network activity and determine what confidential information was taken before leaving the company.

---

# Investigation

## 1. Opening the Network Capture

The investigation started by opening:

```text
capture.pcapng
```

in **Wireshark**.

Initially, much of the interesting network traffic was protected using TLS.

This meant that although the network connections could be observed, the actual HTTP requests and transferred files could not easily be inspected.

However, the challenge also provided:

```text
sslkeylog.txt
```

This file contains TLS session secrets that can be supplied to Wireshark to decrypt the captured TLS sessions.

---

## 2. Loading the TLS Session Keys

In Wireshark, I navigated to:

```text
Edit
→ Preferences
→ Protocols
→ TLS
```

Under:

```text
(Pre)-Master-Secret log filename
```

I selected:

```text
sslkeylog.txt
```

After applying the configuration, Wireshark successfully decrypted the TLS traffic.

A simple display filter confirmed that HTTP traffic was now visible:

```text
http
```

The packet details also displayed:

```text
Decrypted TLS
```

confirming that the TLS key log had been loaded successfully.

---

# HTTP Traffic Analysis

## 3. Searching for POST Requests

Since the challenge asks what the employee **took with them**, I focused on outgoing data transfers.

HTTP `POST` requests were especially interesting because they can be used to submit or upload data.

I applied:

```text
http.request.method == "POST"
```

This reduced the traffic to POST requests.

Several different requests appeared, including:

```text
POST /login
POST /portal/share
POST /api/v1/ingest/...
```

The `/login` requests represented authentication activity and were not immediately relevant.

The `/portal/share` and `/api/v1/ingest/` requests were much more interesting.

---

## 4. Investigating `/portal/share`

I filtered specifically for document-sharing activity:

```text
http.request.uri == "/portal/share"
```

This revealed several document-sharing requests from different internal hosts.

Examples included packets such as:

```text
84
621
930
1330
2228
```

To inspect each transaction, I used:

```text
Right Click
→ Follow
→ HTTP Stream
```

---

# Comparing Document Transfers

## 5. Normal Transfer

One of the first streams contained:

```http
POST /portal/share HTTP/1.1
Host: portal.aventixdynamics.local
```

The request body contained:

```text
document_id=DOC-3140&destination=partner-sync.aventixlogistics.local&note=SR-10188
```

This appeared consistent with normal business activity.

The destination was:

```text
partner-sync.aventixlogistics.local
```

and the corresponding file transfer was:

```text
Logistics_Manifest_W31.pdf
```

Nothing about this transaction immediately indicated data exfiltration.

---

## 6. Another File-Drop Transfer

Another `/portal/share` request contained:

```text
document_id=DOC-2291
destination=filedrop.vendorhub.local
note=SR-10201
```

The corresponding `/api/v1/ingest/` traffic showed:

```text
Aventix_Vendor_SOW_Nordvale.pdf
```

This was worth investigating, but further analysis revealed an even more suspicious transfer later in the capture.

---

# Identifying the Suspicious Transfer

## 7. Packet 2228

Packet `2228` contained another:

```http
POST /portal/share HTTP/1.1
```

Following its HTTP stream revealed:

```text
document_id=DOC-4471&destination=cloudbackup-relay.net&note=
```

This immediately stood out.

The document was:

```text
DOC-4471
```

and its destination was:

```text
cloudbackup-relay.net
```

Unlike the normal internal or partner destinations observed previously, this looked like an external relay.

The `note` field was also empty:

```text
note=
```

This made the transaction particularly interesting.

---

# Correlating the Transfer

## 8. Searching `/api/v1/ingest`

To identify which actual file corresponded to the suspicious document, I filtered for the ingest API:

```text
http.request.uri contains "/api/v1/ingest"
```

Several files appeared:

```text
Logistics_Manifest_W31.pdf

Aventix_Vendor_SOW_Nordvale.pdf

Invoice_Batch_2026-07.pdf

Employee_Handbook_Rev9.pdf

Aventix_Q3_Client_Roadmap_CONFIDENTIAL.pdf
```

The most suspicious request appeared at packet:

```text
2316
```

This occurred shortly after the `DOC-4471` share request.

---

# Inspecting Packet 2316

## 9. Following the HTTP Stream

I selected packet `2316` and used:

```text
Right Click
→ Follow
→ HTTP Stream
```

The decrypted request revealed:

```http
POST /api/v1/ingest/Aventix_Q3_Client_Roadmap_CONFIDENTIAL.pdf HTTP/1.1
Host: cloudbackup-relay.net
```

Additional headers provided even stronger evidence:

```text
X-Aventix-Document: DOC-4471
X-Aventix-Classification: Restricted
X-Aventix-Submitted-By: d.morrison
Content-Type: application/pdf
```

This correlated perfectly with the earlier `/portal/share` request:

```text
DOC-4471
        ↓
cloudbackup-relay.net
        ↓
Aventix_Q3_Client_Roadmap_CONFIDENTIAL.pdf
```

The document was explicitly classified as:

```text
Restricted
```

This identified it as the likely exfiltrated file.

---

# Extracting the PDF

## 10. Exporting HTTP Objects

Because the complete PDF was transferred over HTTP inside the decrypted TLS session, it could be recovered directly from the packet capture.

In Wireshark, I selected:

```text
File
→ Export Objects
→ HTTP
```

I then searched for:

```text
Aventix_Q3_Client_Roadmap
```

Two objects appeared.

One was:

```text
Content-Type: application/pdf
Size: ~87 KB
Packet: 2316
```

and another was the JSON response from the receiving server.

I selected the `application/pdf` object and saved it as:

```text
Aventix_Q3_Client_Roadmap_CONFIDENTIAL.pdf
```

---

# Examining the Exfiltrated Document

## 11. PDF Contents

Opening the extracted PDF revealed:

```text
Q3 CLIENT ROADMAP — CONFIDENTIAL
```

The document was:

```text
Aventix Dynamics
Strategy & Client Delivery
Document DOC-4471
Revision 6
```

This confirmed that the exported PDF was indeed the same `DOC-4471` observed in the suspicious network request.

The document contained confidential business information including:

- client delivery commitments
- renewal information
- pricing floors
- client account information
- internal business risks

The document explicitly warned that some information should not leave the company.

---

# Finding the Flag

## 12. Renewal Posture

On the first page, section:

```text
3. Renewal posture
```

contained an internal delegation reference.

The text included:

```text
NADI{TH3_FL4G_15_H3R3}
```

This matched the expected CTF flag format.

---

# Final Flag

```text
NADI{TH3_FL4G_15_H3R3}
```

The flag was submitted successfully.

---

# Investigation Timeline

The complete investigation path was:

```text
capture.pcapng
      │
      ▼
Encrypted TLS Traffic
      │
      ▼
Load sslkeylog.txt
      │
      ▼
TLS Successfully Decrypted
      │
      ▼
Filter HTTP POST Requests
      │
      ▼
http.request.method == "POST"
      │
      ▼
Investigate /portal/share
      │
      ▼
Packet 2228
      │
      ▼
DOC-4471
      │
      ▼
destination=cloudbackup-relay.net
      │
      ▼
Correlate /api/v1/ingest Requests
      │
      ▼
Packet 2316
      │
      ▼
Aventix_Q3_Client_Roadmap_CONFIDENTIAL.pdf
      │
      ├── Classification: Restricted
      ├── Document: DOC-4471
      └── Submitted By: d.morrison
      │
      ▼
Export HTTP Object
      │
      ▼
Open Extracted PDF
      │
      ▼
Section 3 — Renewal posture
      │
      ▼
NADI{TH3_FL4G_15_H3R3}
```

---

# Key Evidence

| Evidence | Finding |
|---|---|
| Network capture | `capture.pcapng` |
| TLS key log | `sslkeylog.txt` |
| Suspicious share packet | `2228` |
| Document ID | `DOC-4471` |
| Destination | `cloudbackup-relay.net` |
| File-transfer packet | `2316` |
| Filename | `Aventix_Q3_Client_Roadmap_CONFIDENTIAL.pdf` |
| Classification | `Restricted` |
| Submitted By | `d.morrison` |
| Flag | `NADI{TH3_FL4G_15_H3R3}` |

---

# Useful Wireshark Filters

### Display HTTP Traffic

```text
http
```

### Find POST Requests

```text
http.request.method == "POST"
```

### Find Portal Sharing Activity

```text
http.request.uri == "/portal/share"
```

### Find File Ingestion Requests

```text
http.request.uri contains "/api/v1/ingest"
```

### Display a Specific Packet

```text
frame.number == 2316
```

These filters significantly reduced the amount of traffic that needed to be manually inspected.

---

# What I Learned

## 1. TLS Does Not Always Mean Traffic Is Unrecoverable

Normally, TLS prevents packet-capture analysis from viewing HTTP contents.

However, if TLS session secrets are available, Wireshark can decrypt supported captured sessions.

In this challenge, the provided:

```text
sslkeylog.txt
```

allowed the encrypted traffic to be reconstructed.

---

## 2. Filtering Is Essential

A PCAP may contain thousands of packets.

Instead of manually checking every packet, filters such as:

```text
http.request.method == "POST"
```

and:

```text
http.request.uri == "/portal/share"
```

made it possible to focus on relevant activity.

---

## 3. Correlation Is More Reliable Than a Single Indicator

The suspicious activity was not identified from the filename alone.

Multiple artifacts correlated:

```text
DOC-4471
```

appeared in the portal request and the subsequent upload.

The destination was:

```text
cloudbackup-relay.net
```

and the uploaded object was:

```text
Aventix_Q3_Client_Roadmap_CONFIDENTIAL.pdf
```

with:

```text
X-Aventix-Classification: Restricted
```

Together, these provided much stronger evidence.

---

## 4. HTTP Objects Can Be Reconstructed

Wireshark's:

```text
File → Export Objects → HTTP
```

feature can recover files transferred through HTTP when the traffic is available in plaintext or successfully decrypted.

This allowed the original PDF to be reconstructed directly from the PCAP.

---

## 5. Follow HTTP Stream Is Extremely Useful

Using:

```text
Follow → HTTP Stream
```

made it possible to view complete request and response conversations rather than examining individual packets.

This revealed important metadata such as:

```text
document_id
destination
classification
submitted-by
filename
```

---

# Conclusion

The challenge required investigating encrypted network traffic generated around an employee's departure from Aventix Dynamics.

Using the supplied TLS session key log, the TLS traffic was decrypted in Wireshark.

Analysis of `/portal/share` traffic identified:

```text
DOC-4471
```

being sent to:

```text
cloudbackup-relay.net
```

The subsequent file-transfer request revealed that the document was:

```text
Aventix_Q3_Client_Roadmap_CONFIDENTIAL.pdf
```

and was classified:

```text
Restricted
```

The PDF was reconstructed using Wireshark's HTTP object export functionality.

Inspection of the recovered document revealed the final flag:

```text
NADI{TH3_FL4G_15_H3R3}
```

---

## Challenge Solved

**Flag:**

```text
NADI{TH3_FL4G_15_H3R3}
```
