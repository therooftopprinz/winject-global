# winject-global VPN protocol — security layer (draft)

This document specifies the **security layer**: **packet framing**, **Noise_IK**, **cryptographic suites** (**`suite_id`**), and handshake and encrypted record shapes carried over **UDP**. It defines what peers exchange to establish a **confidential, authenticated channel** between static keys.

The **transport layer** (inner payloads after decryption, mesh control plane, path selection, multiplexing, and related semantics) will be specified separately.

## 1. Byte order and constants

- Multi-byte integers are **big-endian** unless noted.
- **`PROTOCOL_VERSION`**: `1`. Peers MUST use **`version == 1`** for datagrams defined in this document.

## 2. Packet framing

Every UDP datagram is **`header` (§2.1)** immediately followed by the **`msg_type`–specific fields defined** in §2.2–§2.6.

**`session_id`** is an **8-byte** opaque label. The **responder** chooses it when accepting **`HANDSHAKE_IK_MSG1`**, binds it to that handshake’s state (**`suite_id`** and Noise handshake state), and sends it in **`HANDSHAKE_IK_MSG2`**. The **initiator** learns **`session_id`** from **`HANDSHAKE_IK_MSG2`** and uses it in **`TRANSPORT`** datagrams for the established session. Responders SHOULD generate it with a CSPRNG and MUST NOT reuse a **`session_id`** while a session bound to it could still be valid (lifetime and teardown are specified with the transport layer).

Receivers MUST drop packets with **`version ≠ PROTOCOL_VERSION`**, unknown **`msg_type`**, or **UDP length** inconsistent with §2.2–§2.6.

### 2.1 `header` (all `msg_type` values)

| Size | Field | Description |
|------|--------|-------------|
| 1 | `version` | MUST equal **`PROTOCOL_VERSION`**. |
| 1 | `msg_type` | Message Type |

### 2.2 `HANDSHAKE_IK_MSG1` (`msg_type == 1`)

Together they carry **Noise IK** message 1 (pattern **`IK`**, **`-> e, es, s, ss`**): **`e[]`** is the cleartext ephemeral public key (**`e`** token); **`s[]`** is the AEAD output for the initiator’s static public key (**`s`** token after **`es`**).

**`e_size`** MUST equal **`DHLEN`** for **`suite_id`**. **`s_size`** MUST equal **`DHLEN + tag_len`** ( **`tag_len`** is **16** for ChaCha20-Poly1305–style AEADs). Receivers MUST verify both lengths against **`suite_id`** and that the **UDP length** equals **`header` (§2.1)** plus the sum of all fields in the table below. For **Curve25519** with **`tag_len = 16`**: **`e_size = 32`**, **`s_size = 48`**. **`e_size`** and **`s_size`** MUST each fit in **`u8`** ( **≤ 255** ).

| Size | Field | Description |
|------|--------|-------------|
| 2 | `suite_id` | Non-zero; selects DH, cipher, and hash per deployment-defined suite table (§3). |
| 1 | `e_size` | **`u8`**; length of **`e`**; MUST equal **`DHLEN`**. |
| 1 | `s_size` | **`u8`**; length of **`s`**; MUST equal **`DHLEN + tag_len`**. |
| `e_size` | `e` | Initiator ephemeral DH public key (**Noise `e`**). |
| `s_size` | `s` | AEAD ciphertext for the initiator’s **static** DH public key (**Noise `s`**): **`DHLEN`** bytes plaintext plus **`tag_len`** tag bytes. |

### 2.3 `HANDSHAKE_IK_MSG2` (`msg_type == 2`)

The **`e`** bytes are exactly the **Noise IK** second handshake message: pattern **`IK`**, direction **`<- e, ee, se`**. For suites where that message consists only of the responder’s cleartext ephemeral **`e`**, **`e_size`** MUST equal **`DHLEN`** for the **`suite_id`** from the **`HANDSHAKE_IK_MSG1`** this message completes (e.g. **32** for **Curve25519**). **`e_size`** MUST NOT exceed **255** ( **`u8`** ). Receivers MUST verify **`e_size`** against the negotiated suite and that the **UDP length** equals **`header` (§2.1)** plus the sum of all fields in the table below.

| Size | Field | Description |
|------|--------|-------------|
| 8 | `session_id` | Responder-chosen session identifier for this handshake (semantics at the start of §2). |
| 1 | `e_size` | **`u8`**; length of **`e`**; MUST equal **`DHLEN`** for the negotiated suite. |
| `e_size` | `e` | Responder ephemeral DH public key (**Noise `e`**); **Noise IK** message 2 bytes for that suite. |

### 2.4 `TRANSPORT` (`msg_type == 3`)

After **`header` (§2.1)** come **`session_id`**, then **`payload_size`** ( **`u16`**, big-endian ) and **`payload[payload_size]`** ( **`payload_size`** octets). Receivers MUST verify that the **UDP length** equals **`header` (§2.1)** plus the sum of all fields in the table below. **`payload_size`** MUST NOT exceed **65535**.

| Size | Field | Description |
|------|--------|-------------|
| 8 | `session_id` | Established session; **`negotiated_suite_id`** is looked up from session state, not carried here. |
| 2 | `payload_size` | **`u16`**; length in bytes of **`payload`** immediately following. |
| `payload_size` | `payload` | Octets of the encrypted transport record / inner payload (transport semantics are defined separately). |

### 2.5 `REPORT_REQUEST` (`msg_type == 4`)

**`REPORT_REQUEST`** and **`REPORT_RESPONSE`** (§2.6) are **cleartext** link probes: they carry **no** **`session_id`** and require **no** established Noise session. Peers use them to check **link state** (reachability, **RTT**) and to exchange **responder-side link statistics** even when there is **no active transport session** on this link.

After **`header` (§2.1)** comes **`sender_time_us`**. Receivers MUST verify that the **UDP length** equals **`header` (§2.1)** plus the sum of all fields in the table below.

| Size | Field | Description |
|------|--------|-------------|
| 8 | `sender_time_us` | **`u64`**; sender timestamp in microseconds (clock domain and monotonicity are implementation-defined; SHOULD be suitable for RTT measurement on the sender). |

### 2.6 `REPORT_RESPONSE` (`msg_type == 5`)

Sent in reply to **`REPORT_REQUEST`**. **`sender_time_us`** MUST echo the **`sender_time_us`** value from the request being answered. Receivers MUST verify that the **UDP length** equals **`header` (§2.1)** plus the sum of all fields in the table below.

| Size | Field | Description |
|------|--------|-------------|
| 8 | `sender_time_us` | Echo of **`sender_time_us`** from the corresponding **`REPORT_REQUEST`**. |
| 8 | `responder_time_us` | **`u64`**; responder timestamp in microseconds when sending this reply (same clock conventions as §2.5). |
| 8 | `drop_rate_pkt_s` | **`u64`**; observed **drop rate** toward the requester, in **packets per second** (measurement window and semantics are implementation-defined). |
| 8 | `throughput_pkt_s` | **`u64`**; observed **throughput** toward the requester, in **packets per second** (same). |
| 8 | `throughput_byt_s` | **`u64`**; observed **throughput** toward the requester, in **bytes per second** (same). |

## 3. Cryptographic suite (`suite_id`)

**`suite_id`** is a non-zero **`u16`** that names a **cryptographic suite**: the **DH** primitive (group / curve and **`DHLEN`**), the **AEAD cipher** (and thus **`tag_len`**), and the **hash** used by Noise for that handshake and the derived session.

The mapping **`suite_id` → (DH, cipher, hash)** is **not** negotiated in-band. It is **defined by deployment configuration** (a fixed table or policy on each peer). Operators MUST deploy **compatible** suite tables so initiator and responder interpret the same **`suite_id`** the same way. Receivers MUST reject handshakes with an unknown **`suite_id`** (or one their policy does not allow).
