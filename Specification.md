# HTTP Catcher Session File Format — Reverse‑Engineered Spec (Updated v2)

*Status: Informational / Reverse‑Engineered Draft*  
*This document is non‑normative and may contain inaccuracies.*  
*Updated:* 2025-09-22T15:48:58Z  

---

## Abstract

This document describes the **binary session file format** used by **HTTP Catcher**.  
It reflects the **latest findings** from live dumps and a working parser, including:
- acceptance of **leading timestamps** only within a calibrated time window,
- semantics of the **Request Main Info** (RMI) frame and its optional `ts_leading`,
- recognition of **tail‑only fragments** (payload missing, footer only),
- **CONNECT** tunnel specifics,
- **segment layouts with and without leading timestamps**,
- and **derived timing formulas** (DNS, connect, TLS, send, wait, receive) suitable for HAR export.

> This draft summarizes behavior observed in builds tested to date. Future app versions may differ.

---

## 1. Design Principles (as observed)

- **Marker‑driven parsing.** Parsers MUST identify frame types via **marker values**; do not trust order.  
- **Mixed layouts.** Some frames optionally prepend a **64‑bit leading timestamp** (`ts_leading`).  
- **Footer repetition.** Segment footers repeat critical fields (marker, request id, timestamp, length).  
- **Journaling tolerance.** Files may contain **incomplete** (tail‑only) fragments; parsers SHOULD surface them as truncated.  
- **Calibrated timestamps.** Timestamps are treated as **ms since epoch** (observed) and validated against a **baseline window** to reject stray 64‑bit integers.

---

## 2. File Layout Overview

1. **File Header** (once)  
2. Zero or more **Connection Frames** (may interleave with others)  
3. One or more **Request/Response Segments** (order not guaranteed)  
4. Optional **loose timestamps** and **tail‑only fragments**

Parsers must be robust against arbitrary interleaving.

---

## 3. File Header

| Field | Size | Value | Notes |
|------:|-----:|:-----:|------|
| Magic | 16 bit | `0x0100` | MUST be present at file start |

---

## 4. Connection Frame

```
+0x00  Unknown (24‑bit)
+0x03  Pad (8‑bit)                  == 0x00
+0x04  Connection ID (32‑bit)
+0x08  Timestamp #1 (64‑bit)        == ts1
+0x10  Timestamp #2 (64‑bit)        == ts2
+0x18  Port (16‑bit)
+0x1A  Hostname length (16‑bit)     == Lh
+0x1C  Hostname (Lh bytes, UTF‑8)
+...   IP length (16‑bit)           == Li
+...   IP bytes (Li bytes)
+...   Marker (32‑bit)              == 0x00001801
+...   Connection ID (32‑bit)       == repeat, MUST match
+...   Timestamp post #1 (64‑bit)   == ts_post1
+...   Timestamp post #2 (64‑bit)   == ts_post2
```

### 4.1 Semantics (observed)

- `ts1`, `ts2`, `ts_post1`, `ts_post2` are **ms** timestamps (observed).  
- In practice:
  - **DNS resolution:** `dns_ms = ts2 − ts1`  
  - **Connection build:** `conn_build_ms = (ts_post2 or rmi.ts_leading if ts_post2 == 0) − ts_post1`  
  - **TLS configuration:** `tls_conf_ms = rmi.ts_leading − ts_post2` (only if `ts_post2 > 0`)  
- The **first** `ConnectionFrame.ts1` defines the **baseline** for validating timestamps (see §9).

---

## 5. Request Main Info (RMI)

Two layouts exist, both share the same marker `0x00000702`.

### 5.1 RMI with leading timestamp
```
+0x00  ts_leading (64‑bit)          -- request start (observed)
+0x08  Marker (32‑bit)              == 0x00000702
+0x0C  Request ID (32‑bit)
+0x10  Flags (16‑bit)               -- often 0x0101 (purpose unknown)
+0x12  Proto (8‑bit)                -- 0=http/https, 1=WebSocket (observed)
+0x13  Connection ID (32‑bit)
```

### 5.2 RMI without leading timestamp
```
+0x00  Marker (32‑bit)              == 0x00000702
+0x04  Request ID (32‑bit)
+0x08  Flags (16‑bit)
+0x0A  Proto (8‑bit)
+0x0B  Connection ID (32‑bit)
```

### 5.3 Semantics

- `ts_leading` is **present** predominantly when **`request_id == connection_id`** (first request on a connection).  
- On subsequent requests, `ts_leading` is usually absent.  
- Parsers SHOULD validate `ts_leading` against the baseline window.

---

## 6. Request/Response Segments

Segments can appear in two layouts: **with** or **without leading timestamp**.

### 6.1 Segment with leading timestamp
```
+0x00  ts_leading (64‑bit)          -- optional start ts
+0x08  Declared length (24‑bit)     == LEN
+0x0B  Status (8‑bit)               -- type code (§6.4)
+0x0C  Request ID (32‑bit)
+0x10  Payload (LEN bytes)
+...   Footer (see §6.3)
```

### 6.2 Segment without leading timestamp
```
+0x00  Declared length (24‑bit)     == LEN
+0x03  Status (8‑bit)
+0x04  Request ID (32‑bit)
+0x08  Payload (LEN bytes)
+...   Footer (see §6.3)
```

### 6.3 Footer (common)
```
+...   End marker (32‑bit)          == 0x00000C05 (request) or 0x00000C08 (response)
+...   Request ID (32‑bit)          -- repeat, MUST match
+...   Timestamp (64‑bit)           == ts_post
+...   Pad (8‑bit)                  == 0x00
+...   Declared length (24‑bit)     == LEN (repeat, MUST match)
```

### 6.4 Status (type) codes
| Value | Context   | Meaning                    |
|------:|-----------|----------------------------|
| `3`   | Request   | Header (start line + hdrs) |
| `4`   | Request   | Body chunk                 |
| `6`   | Response  | Header (status + hdrs)     |
| `7`   | Response  | Body chunk (mid)           |
| `9`   | Response  | Body chunk (final)         |
| `8`   | Response  | Trailer headers            |

Notes:  
- A zero‑length final body (status `9` with `LEN=0`) is valid.  
- `ts_post` is always in the footer; `ts_leading` only sometimes at start.

---

## 7. Tail‑Only Fragments

Footer‑only remnants (no prologue/payload).  
- Identified by end marker followed by footer fields.  
- Subtype is unknown.  
- Missing byte count = repeated LEN.  
- Preserve footer timestamp if within baseline; else omit.

---

## 8. CONNECT Tunnels

- `CONNECT` request → tunnel.  
- Success `200` response header marks tunnel established.  
- Following `0x0000080A` markers or tail‑only fragments tied to same req_id = tunnel bytes.

---

## 9. Timestamp Calibration

- All timestamps appear to be ms since Unix epoch.  
- **Baseline:** first ConnectionFrame.ts1.  
- Accept ts only if `baseline <= ts <= baseline + 1 day`.  
- Outside values → treat as absent.

Loose standalone ts64 may appear; surface as `TimestampMark` if within window.

---

## 10. Derived HAR Timings

- **dns:** conn.ts2 − conn.ts1  
- **connect:** (conn.ts_post2 or rmi.ts_leading if ts_post2==0) − conn.ts_post1  
- **ssl:** rmi.ts_leading − conn.ts_post2 (if both valid)  
- **send:** req_body.ts_post − req_header.ts_post (else 0)  
- **wait:** resp_header.ts_post − (req_body.ts_post or req_header.ts_post)  
- **receive:** resp_final_or_last_body.ts_post − resp_header.ts_post  

Deltas <0 or with missing endpoints → null.

---

## 11. Robustness

- Marker‑driven parsing, not order.  
- Accept both segment layouts (with/without leading ts).  
- Enforce length repeat consistency.  
- Implement baseline window filter.  
- Surface tail‑only fragments with truncated flag.  
- Detect CONNECT + tunnel markers.

---

## 12. Known Unknowns

- Meaning of RMI Flags.  
- Purpose of ConnectionFrame 24‑bit prefix.  
- Epoch definition across versions (assumed ms Unix).

---

## 13. Changelog

- **v0.4** — Segments documented in both layouts (with/without leading ts); clarified ts_post vs ts_leading roles.  
- **v0.3** — Added baseline timestamp filter, clarified RMI ts_leading, connect/TLS timing formulas.  
- **v0.2** — Added TLS marker & CONNECT handling.  
- **v0.1** — Initial draft.

---

## 14. Disclaimer

This is a **reverse‑engineered** document. Not official. Future versions of HTTP Catcher may differ.
