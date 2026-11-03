# Direct Client-to-Client (DCC) Protocol Specification

| Field | Value |
|---|---|
| Title | Direct Client-to-Client (DCC) |
| Short name | DCC |
| Version | 1.0 |
| Status | Draft |
| Date | 2026-11-02 |
| Authors | NIBL engineering |
| Builds on | Internet Relay Chat (RFC 1459, RFC 2812), Client-to-Client Protocol (CTCP) |
| Audience | Implementers of IRC clients, bots, file-transfer servers, and the XDCC / SXDCC layers |
| Reference implementation | PircBotX `org.pircbotx.dcc` package (see References) |

---

## 1. Abstract

Direct Client-to-Client (DCC) is a set of conventions, layered on top of Internet
Relay Chat (IRC), that let two IRC clients establish a **direct TCP connection**
between themselves in order to exchange large amounts of data. The IRC network is
used only to carry the small control ("handshake") messages that advertise the
connection parameters (address, port, and metadata). The actual data then flows
**peer-to-peer** between the two endpoints, bypassing the IRC servers.

DCC defines two primary services:

* **DCC CHAT** — a direct, line-oriented bidirectional chat channel between two users.
* **DCC SEND** — a stream-oriented, acknowledgement-driven file transfer.

This document specifies the control messages, framing, byte order, the file-transfer
acknowledgement protocol, resume, passive/reverse connection establishment, and the
normative requirements needed to build interoperable implementations. Extensions
that are widely deployed (DCC RESUME/ACCEPT, passive/reverse DCC, TSEND/turbo DCC,
DCC XMIT) are described in their own sections and marked as extensions.

## 2. Conventions and Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this
document are to be interpreted as described in RFC 2119.

The following terms are used throughout this specification:

| Term | Meaning |
|---|---|
| IRC | Internet Relay Chat (RFC 1459 / RFC 2812). Provides the out-of-band signalling channel. |
| CTCP | Client-to-Client Protocol: a framing convention that carries tagged control messages inside IRC `PRIVMSG` / `NOTICE` text, delimited by `\x01` (ASCII SOH, control-A). |
| DCC control message | The logical payload carried inside CTCP framing, e.g. `DCC SEND …`. |
| Initiator / sender | The party that offers a resource (a chat or a file) and listens (or, in passive mode, connects out). |
| Target / receiver | The party that connects out (active mode) to receive the resource. |
| Network byte order | Big-endian — the byte order used for all integer fields placed on the wire. |
| Pack | A single unit of offered content identified by a number in the XDCC layer (defined in `XDCC.md`). |

### 2.1 Transport summary

* **Control channel** — the IRC `PRIVMSG` / `NOTICE` stream. DCC handshakes are CTCP
  messages; the receiver MUST strip the surrounding `\x01` delimiters before parsing.
* **Data channel** — a raw TCP connection between the two endpoints identified by the
  `ip` and `port` fields of a control message. The data channel carries stream bytes
  only; it has no DCC header of its own.

## 3. Overview and Scope

### 3.1 In scope

* The CTCP handshake messages for DCC CHAT and DCC SEND.
* The encoding of IPv4 addresses and TCP ports inside those messages.
* The file-transfer data phase and its acknowledgement protocol.
* DCC RESUME / DCC ACCEPT.
* Passive / reverse DCC connection establishment.
* The DCC CHAT line protocol.

### 3.2 Out of scope

* The XDCC application layer (pack lists, queues, slots) — see `XDCC.md`.
* The secure TLS transport layer (SXDCC) — see `SXDCC.md`.
* IRC protocol mechanics (routing, nick registration, channel membership).
* Any transport-level authentication, confidentiality, or integrity layer. Classic
  DCC provides **none** by itself (see §13).

## 4. Actors and Roles

| Actor | Role |
|---|---|
| Sender | Advertises an address/port pair via CTCP and offers a chat or a file. In active DCC it is the TCP **server**; in passive DCC it becomes the TCP **client**. |
| Receiver | Learns the address/port pair via CTCP. In active DCC it is the TCP **client**; in passive DCC it becomes the TCP **server**. |

In a single transfer each endpoint has exactly one role. Roles are determined by
the handshake (active vs passive), not by the IRC nick precedence.

## 5. Connection Establishment

### 5.1 Address and port encoding

Inside a DCC control message, the endpoint address and port are transmitted as
**decimal integers written in network (big-endian) byte order**.

* **IPv4 address** — encoded as a single decimal integer equal to the 32-bit
  big-endian representation of the address. For example, `192.168.1.2`
  (bytes `C0 A8 01 02`) is transmitted as the decimal `3232235778`.
* **TCP port** — encoded as the decimal value of the 16-bit port number in network
  byte order (e.g. port `3306` is transmitted as `3306`; the field is the native
  port value, not byte-swapped for display).
* A **port of `0`** is reserved as the signal for **passive / reverse DCC**
  (see §5.4).
* Implementations SHOULD use the 32-bit `unsigned long` (dotted-quad IPC) form for
  IPv4 for maximum interoperability. IPv6 addresses follow the extension schemes in
  §9.

### 5.2 Active DCC

The initiator (sender) opens a listening TCP socket on an ephemeral or configured
port, then sends a CTCP DCC message advertising `ip` and `port`. The target
(receiver) then **connects out** to `ip:port`.

```
+----------+   (1) CTCP DCC SEND/CHAT <ip> <port>    +----------+
|  SENDER  | --------------------------------------> | RECEIVER |
| (server) |                                         | (client) |
+----------+                                         +----------+
     ^                                                      |
     |              (2) TCP connect to ip:port              |
     +------------------------------------------------------+
```

### 5.3 DCC handshake message grammar

All DCC control messages share the form:

```
DCC <TYPE> <args...>
```

`<TYPE>` is a single case-sensitive word (`SEND`, `CHAT`, `RESUME`, `ACCEPT`, …).
Arguments are separated by one or more space characters. A filename argument that
contains spaces MUST be quoted with double quotes (`"`). Implementations MUST accept
both quoted and unquoted filenames.

### 5.4 Passive / reverse DCC

Many endpoints sit behind NAT or firewalls and cannot accept an inbound connection
in active mode. Passive (reverse) DCC) inverts the roles: the **receiver** opens the
listening socket and the **sender** connects out.

Passive DCC is signalled in one of two ways:

1. A `DCC SEND` handshake whose `port` field is `0`, optionally carrying a
   `token`; or
2. A handshake that carries a `token` field (the presence of the token marks the
   handshake as passive).

On accepting a passive offer, the receiver sends back a `DCC SEND` handshake with
its own listening `ip` and `port`, echoing the same `token` so the sender can match
the acceptance to the original offer.

## 6. DCC CHAT

### 6.1 Handshake

```
DCC CHAT <protocol> <ip> <port>
```

* `<protocol>` MUST be `chat` for standard DCC CHAT. (The `wboard` value selects
  DCC Whiteboard, a vendor extension.)
* `<ip>` / `<port>` are as defined in §5.1.

### 6.2 Data phase

Once the TCP connection is established, the two peers exchange lines:

* Each line is **CRLF-terminated** (`\r\n`).
* A line matching the CTCP action form — `\x01ACTION <text>\x01` — is an emote and
  MUST be rendered as an action rather than as literal text.

### 6.3 Passive chat

The passive form appends a `chatToken`:

```
DCC CHAT <protocol> <ip> 0 <chatToken>
```

The receiver opens the listening socket and replies with `DCC CHAT <protocol> <ip>
<port> <chatToken>`, echoing the token.

## 7. DCC SEND

### 7.1 Handshake

```
DCC SEND <filename> <ip> <port> [<filesize>] [<token>]
```

| Field | Required | Meaning |
|---|---|---|
| `<filename>` | REQUIRED | Name of the file. Quote with `"` if it contains spaces. |
| `<ip>` | REQUIRED | Sender's listening address (encoded per §5.1). |
| `<port>` | REQUIRED | Sender's listening port; `0` indicates a passive offer (§5.4). |
| `<filesize>` | OPTIONAL | Total size of the file in bytes (decimal). Widely supported and RECOMMENDED. |
| `<token>` | OPTIONAL | A unique decimal integer identifying a passive transfer. |

The receiver MUST treat an offer with `port == 0` **or** with a present `<token>` as
a passive transfer.

### 7.2 Data phase and acknowledgements

After the TCP connection is established, the sender streams the raw file bytes.
The receiver acknowledges receipt by writing back the **total number of bytes
received so far** as a **32-bit unsigned integer in network (big-endian) byte
order**:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 total bytes received (4 octets)               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* The sender SHOULD send ahead of the acknowledgements (pipelining) rather than
  waiting for each ACK, but MUST tolerate receivers that ACK every block.
* The ACK field is a running total, not a per-block delta.
* Because the classic ACK is 32-bit, transfers larger than 4 GiB wrap; see §9.2 for
  the known interop divergence and §13 for an integrity note.
* The transfer is considered **complete** when the sender has transmitted all
  `filesize` bytes and the receiver has acknowledged that many bytes, at which
  point the receiver closes the data connection.

### 7.3 Packet size

The block ("packet") size is an implementation choice and is **not** negotiated on
the wire. Reference implementations commonly use a default of 1024–8192 bytes;
implementations MUST stream the same logical byte sequence regardless of the block
size they use.

## 8. DCC RESUME and DCC ACCEPT

RESUME lets a receiver request that a transfer begin at a non-zero offset (for
example, to continue a partially received file).

```
DCC RESUME <filename> <port> <position> [<token>]
DCC ACCEPT <filename> <port> <position> [<token>]
```

* `<position>` is the byte offset, in decimal, from which to start (0 = whole file).
* Active form uses the sender's `<port>`; passive form uses `<port> 0` plus
  `<token>`.
* On a valid RESUME, the sender replies with ACCEPT echoing the filename, position,
  and (passive) token; the receiver then connects (active) or listens (passive) and
  the sender begins at `<position>`.

```
>> DCC SEND  <filename> <ip> <port> <filesize> <token>       (passive offer)
<< DCC RESUME <filename> 0 <position> <token>
>> DCC ACCEPT <filename> 0 <position> <token>
<< DCC SEND  <filename> <receiver-ip> <receiver-port> <filesize> <token>
   --- TCP connect, data begins at <position> ---
```

An implementation that does not support RESUME SHOULD ignore the request; a
receiver MUST NOT begin a resumed transfer until it observes the matching ACCEPT.

## 9. Extensions

### 9.1 TSEND (turbo DCC)

TSEND removes the acknowledgement requirement. Two signalling variants exist:

* Older clients replace `SEND` with `TSEND` in the handshake.
* Newer clients keep `SEND` and append `T` to the `<token>` (passive form).

A receiver MUST NOT send ACKs for a TSEND transfer; the sender MUST NOT require
them. TSEND transfers are therefore not reliable against anomalous closes.

### 9.2 DCC XMIT

A more structured variant with its own negotiation and file-metadata fields:

```
DCC XMIT <protocol> <ip> <port> [<name> [<size> [<MIME-type>]]]
```

`<protocol>` is `clear` in practice. DCC XMIT permits dotted-quad IPv4 and IPv6
forms, and carries an initial 32-bit modification time and a 32-bit seek offset,
which limits resume to the 2 GiB boundary. It is **not** required for conformance
and is noted here for interoperability.

### 9.3 DCC file servers (FSERV) and passive variants

A range of vendor mechanisms (DCC Server, DCC GET, RDCC, DCC REVERSE, passive DCC)
address NAT/firewall traversal. The normative minimum in this specification is the
passive/reverse form described in §5.4; other variants are OPTIONAL and MUST be
discussed between peers before use.

## 10. State Machine

The following states describe a DCC SEND transfer from the perspective of either
endpoint. The machine applies to CHAT with the data-phase lines replaced by the
CHAT line protocol.

```
                    OFFERED ──(decline/timeout)──▶ ABORTED
                       │
                       │  accept
                       ▼
                   CONNECTING ──(refused/timeout)──▶ FAILED
                       │
                       ▼
                   TRANSFERRING ──(error/socket close)──▶ FAILED
                       │
                       │  all bytes sent + ACKed
                       ▼
                      DONE
```

| State | Meaning |
|---|---|
| `OFFERED` | The CTCP handshake has been delivered to the receiver. |
| `CONNECTING` | The receiver is opening the data connection. |
| `TRANSFERRING` | File bytes (and ACKs) are flowing. |
| `DONE` | The full byte count was sent and acknowledged. |
| `FAILED` | The data connection failed or was closed prematurely. |
| `ABORTED` | The receiver declined or the offer timed out. |

RESUME is a modifier of the `CONNECTING` state: the receiver only enters
`TRANSFERRING` after a matching `ACCEPT`, beginning at `<position>`.

## 11. Normative Requirements

### 11.1 General

* REQ-1: All DCC control messages SHALL be carried as CTCP messages over IRC.
* REQ-2: All integer fields placed on the wire by the data phase SHALL use network
  (big-endian) byte order.
* REQ-3: An implementation SHALL treat a `port` of `0` as a passive/reverse offer.
* REQ-4: An implementation SHALL NOT interpret data-channel bytes as DCC control
  messages; control and data MUST remain separate channels.

### 11.2 DCC SEND

* REQ-5: A sender SHALL advertise the file size when known (the `filesize` field).
* REQ-6: A receiver SHALL acknowledge bytes during a non-TSEND transfer using the
  32-bit network-order running total defined in §7.2.
* REQ-7: A sender SHALL NOT consider a transfer complete until the receiver has
  acknowledged the full byte count, unless TSEND semantics were negotiated.

### 11.3 DCC CHAT

* REQ-8: A DCC CHAT peer SHALL terminate each line with CRLF.
* REQ-9: A DCC CHAT peer SHALL render `\x01ACTION …\x01` as an emote.

### 11.4 RESUME

* REQ-10: A sender that supports RESUME SHALL answer a valid `DCC RESUME` with
  `DCC ACCEPT` and begin the data phase at the requested `<position>`.

### 11.5 Interoperability

* REQ-11: An implementation SHALL accept both quoted and unquoted filenames, and
  SHALL quote filenames that contain spaces when sending.
* REQ-12: An implementation SHALL tolerate and ignore unknown optional trailing
  fields in a handshake (forward compatibility).

## 12. Security Considerations

DCC, as specified here, provides **no** security properties on the data channel:

* **Confidentiality** — data is transmitted in cleartext.
* **Integrity** — no checksum or MAC protects the file bytes; the ACK counter only
  guards against gross truncation and is not a security control.
* **Authentication** — the control and data channels are unauthenticated; an active
  attacker who can inject IRC traffic may redirect a transfer.
* **Address disclosure** — the `ip` field leaks the sender's address to the IRC
  network.
* **Historical exploit surface** — legacy clients have suffered filename-length and
  port-`0` parsing bugs ("DCC SEND exploit"); implementers MUST bound buffer sizes
  and validate all numeric fields.

These limitations are inherited by XDCC (`XDCC.md`). The SXDCC layer (`SXDCC.md`)
is defined precisely to close them.

## 13. Examples

### 13.1 Active DCC SEND

```
# Control channel (CTCP):
>> DCC SEND "episode 01.mkv" 3232235778 40000 2097152000
# Data channel: receiver connects to 192.168.1.2:40000 and streams bytes.
<< (receiver 4-octet ACKs of running total)
```

### 13.2 Passive DCC SEND with resume

```
PeerA (sender)                                 PeerB (receiver)
      │ CTCP DCC SEND file.bin 0 0 1048576 1700000001
      │──────────────────────────────────────────▶
      │ CTCP DCC RESUME file.bin 0 524288 1700000001
      ◀──────────────────────────────────────────│
      │ CTCP DCC ACCEPT file.bin 0 524288 1700000001
      │──────────────────────────────────────────▶
      │ CTCP DCC SEND file.bin 3232235779 50999 1048576 1700000001
      ◀──────────────────────────────────────────│
      │  TCP connect to 192.168.1.3:50999, begin at byte 524288
      ══════════════════════════════════════════▶│
```

## 14. References

* RFC 1459 — Internet Relay Chat Protocol.
* RFC 2812 — Internet Relay Chat: Client Protocol.
* RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels.
* Direct Client-to-Client (Wikipedia) — historical overview of DCC services and extensions.
* PircBotX, `org.pircbotx.dcc` package — `DccHandler`, `SendFileTransfer`,
  `ReceiveFileTransfer`, `SendFileTransferAcknowlegement`,
  `ReceiveFileTransferAcknowlegement`, `Chat` (reference implementation).