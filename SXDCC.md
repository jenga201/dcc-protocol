# Secure eXtended DCC (SXDCC) Protocol Specification

| Field | Value |
|---|---|
| Title | Secure eXtended DCC (SXDCC) |
| Short name | SXDCC |
| Version | 1.0-draft |
| Status | Draft |
| Date | 2026-11-02 |
| Authors | NIBL engineering |
| Builds on | XDCC (`XDCC.md`), DCC (`DCC.md`), IRC (RFC 1459 / 2812), TLS 1.3 (RFC 8446) |
| Audience | Implementers of SXDCC clients and bots in any language/runtime |
| Reference implementation | none yet (this document is the contract) |

---

## 1. Abstract

SXDCC is a confidential, integrity-protected successor to XDCC. It keeps the XDCC
command vocabulary and pack-centric delivery model but replaces the two cleartext
channels of classic XDCC — IRC for control and raw TCP for file data — with
**TLS 1.3** channels. IRC is retained only as an **untrusted rendezvous channel**
to bootstrap a connection; every secret-bearing message (commands, pack lists,
queue state, and file bytes) flows over mutually established TLS, so an observer of
the IRC network learns only that a client contacted a bot.

SXDCC satisfies two design goals, in order:

1. **Security** — server authentication by pinned public key, confidentiality and
   integrity by TLS 1.3 AEAD, forward secrecy, and MITM-resistant data-channel
   binding.
2. **Ease of use** — no public CA and no DNS dependency; it uses self-signed
   certificates with **IP** subject alternative names, a minimal 32-byte trust
   anchor, and a first-contact bootstrap that works headless.

## 2. Conventions and Terminology

The key words **MUST**, **MUST NOT**, **SHALL**, **SHALL NOT**, **SHOULD**,
**SHOULD NOT**, **MAY**, and **OPTIONAL** are interpreted per RFC 2119. Terms from
`DCC.md` and `XDCC.md` apply.

| Term | Meaning |
|---|---|
| Bot | The SXDCC server: a host that offers packs over TLS. |
| Client | The SXDCC endpoint that connects to a bot to list and download packs. |
| Rendezvous channel | The IRC connection; used only to advertise a session invite. |
| Control channel | A long-lived TLS 1.3 connection used for commands and status. |
| Data channel | A short-lived TLS 1.3 connection used to stream one pack's bytes. |
| SPKI | Subject Public Key Info — the DER-encoded public key of the server certificate. |
| SPKI pin | The cryptographic binding `sha256:<base64url( SHA-256(SPKI) )>`. 43 characters, no padding. |
| Token | A single-use, high-entropy, short-lived value used for channel binding. |
| TOFU | Trust On First Use — recording an identity the first time it is seen and enforcing it thereafter. |

## 3. Scope

### 3.1 In scope

* The IRC rendezvous message and its trust bootstrap.
* The TLS 1.3 control-channel protocol (command vocabulary, framing, status).
* The TLS 1.3 data-channel protocol (per-pack raw stream, binding, completion).
* Server identity: self-signed leaf certificate, IP SAN, SPKI pinning.
* First-contact trust: IRC-anchored TOFU with an optional explicit pin override.

### 3.2 Out of scope

* Modifying the IRC server: SXDCC requires **no IRC server changes** and adds no
  IRC extensions. The IRC network remains an unchanged, dumb relay.
* Client (mutual) authentication and per-user authorization — out of scope; an
  optional plugin (see §13.2). v1 has no client-auth gate and gates connections by
  token only.
* Multiplexing control and data onto one connection — a future extension (§13.3).
* Key/certificate rollover without re-pinning — see §7.5.

## 4. Architecture

```
                       IRC (unmodified, untrusted relay)
                              ▲  PRIVMSG: sxdcc <verb> [args]
                              │  NOTICE: SXDCC CONNECT <ip> <port> <token> <pin>
                              ▼
        ┌─────────────────────────────────────────────────────┐
        │  CLIENT                                          BOT │
        │                                                      │
        │  control channel ──── TCP + TLS 1.3 ────────────────▶│  long-lived;
        │     · AUTH <token>                                   │  carries LIST / SEND /
        │     · LIST / SEND #N / CANCEL ...                    │  QUEUE / SEND offer /
        │                                                      │  DONE
        │  data channel     ──── TCP + TLS 1.3 ────────────────▶│  one per pack;
        │     · DATA <datatoken>  then raw file bytes to EOF    │  raw stream, no ACKs
        └─────────────────────────────────────────────────────┘
```

* **Control channel** — established first; the client authenticates the bot via the
  pinned certificate and presents the rendezvous token. All XDCC-style commands and
  status travel here, line-framed (§9).
* **Data channel** — established per pack; negotiated over the already-authenticated
  control channel and bound by a single-use `datatoken` (§10).

## 5. Actors and Roles

| Actor | Role |
|---|---|
| Bot | Listens for TLS connections; serves packs; owns the long-term key that the SPKI pin binds to. |
| Client | Connects out; lists and requests packs; verifies the bot identity from the pin; receives data connections. |

## 6. Cryptographic Requirements

* The transport SHALL be **TLS 1.3 or later** (RFC 8446). TLS 1.2 MUST NOT be
  negotiated for SXDCC.
* Key exchange SHALL use an ephemeral (EC)DHE group; implementations MUST support
  **X25519**. Forward secrecy is thereby guaranteed per handshake.
* Cipher suites MUST include at least one of `TLS_AES_256_GCM_SHA384` or
  `TLS_CHACHA20_POLY1305_SHA256`; implementations SHOULD prefer AES-256-GCM first
  and fall back to ChaCha20-Poly1305.
* The server certificate SHALL be a **self-signed leaf** (a chain of length 1). It
  SHALL carry exactly one **IP** subject alternative name matching the address on
  which the bot listens. It SHALL NOT require a DNS name, and the client SHALL NOT
  consult the operating system trust store.
* Server keys SHOULD be **Ed25519** (or ECDSA P-256). RSA keys SHALL NOT be used
  for new deployments.

### 6.1 SPKI pin

The trust anchor is the SHA-256 digest of the DER-encoded `SubjectPublicKeyInfo` of
the server certificate, rendered as:

```
sha256:<base64url( SHA-256(SPKI) )>
```

The `<base64url>` encoding uses the URL-safe alphabet and **no padding** (43
characters for a 32-byte digest), so the pin is a single portable token in any
language. Implementations MUST compare pins with a **constant-time** comparison
(e.g. `subtle.ConstantTimeCompare`, `hmac.compare_digest`); they MUST NOT use `==`.

## 7. Server Identity and Trust Bootstrap

### 7.1 Client verification algorithm

On every TLS connection (control and data), the client SHALL:

1. complete the TLS 1.3 handshake, pinning the server certificate to the expected
   SPKI pin (§7.2);
2. verify the certificate's IP SAN equals the address to which it connected;
3. verify the certificate chain has depth 1 (self-signed leaf); and
4. fail closed (abort the connection) if any check fails — never prompt the user.

### 7.2 The expected pin

The expected pin is resolved in one of two ways:

1. **Explicit override (strongest).** If the client is started with an explicit pin
   (`SXDCC_PIN` environment variable or equivalent `--pin` configuration), that pin
   SHALL be used unconditionally for every connection. This eliminates the
   first-contact exposure window.
2. **IRC-anchored TOFU (default).** Otherwise the client learns the pin from the
   rendezvous message (§8) and records it against the bot's IRC identity in a local
   trust store.

### 7.3 IRC-anchored TOFU store

* The trust store maps an **IRC identity** to a single SPKI pin. The identity key
  SHALL be the bot's registered account name when available (IRCv3 `account-tag` or
  the `account` field of WHOX); otherwise it SHALL be the `nick!user@host` mask.
* On the **first** rendezvous with an unknown identity, the client SHALL record the
  pin and proceed. This is the only exposure window.
* On a subsequent rendezvous with a **known** identity, the client SHALL compare the
  advertised pin to the recorded pin. A match proceeds; a mismatch SHALL fail
  closed and surface an operator alert (log / metric), with **no interactive
  prompt**. This is the SXDCC analogue of SSH's "host key changed" and must be
  actionable by automation, not by a human at the keyboard.

### 7.4 Rationale

This design deliberately avoids a private CA. Because the client is not compiled
with a pre-provisioned root, a CA would itself have to be discovered and trusted on
first contact — strictly more machinery for no benefit. A single self-signed leaf
plus a 32-byte SPKI pin is the minimal anchor, and the pin is the most portable
value to transport across heterogeneous implementations.

### 7.5 Key rollover

Pin the **public key**, not the certificate: the bot MAY renew its self-signed
certificate (new serial/validity) around the **same** key without changing the pin.
Rotating the **key** changes the SPKI and therefore the pin; clients discover the
new pin through the same §7.2 path (explicit override, or a controlled
IRC-announced transition). No-touch key rotation is intentionally out of scope for
v1.

## 8. IRC Trigger and Rendezvous

SXDCC uses two IRC messages, both plain `PRIVMSG`/`NOTICE` (never CTCP, so networks
that filter or rate-limit CTCP DCC relay them unchanged). Both run in cleartext over
the untrusted IRC channel.

1. **Request (client → bot).** The XDCC command, mirrored from classic XDCC but with
   an `sxdcc` prefix instead of `xdcc`:
   ```
   sxdcc <verb> [<args>]
   ```
   Examples: `sxdcc send #1234`, `sxdcc list`, `sxdcc batch #10-#20`. Only the verb
   and numeric arguments (pack number, range) traverse IRC; filenames and file
   content are **never** present on this channel.
2. **Invitation (bot → client).** The bot answers with the rendezvous:

```
SXDCC CONNECT <ip> <port> <token> <pin>
```

| Field | Meaning |
|---|---|
| `SXDCC CONNECT` | Literal invitation verb. |
| `<ip>` | Bot's public address, DCC 32-bit network-order integer (decimal) for IPv4 (`DCC.md` §5.1); the advertised DCC address (`usenatip` when behind NAT). |
| `<port>` | Bot's TLS listener, selected from the DCC transfer range (`tcprange`). |
| `<token>` | Control-channel token (§9.2): `base64url`, ≥128 bits, single-use, short TTL. |
| `<pin>` | The bot's SPKI pin (`sha256:<base64url>`). |

The IRC channel is **untrusted**: the `pin` is an assertion the client must bind to
the bot's IRC identity via the §7.3 TOFU store. The `sxdcc` prefix is the secure-mode
marker; a bot MAY serve classic `xdcc` and `sxdcc` concurrently, routing solely on
the prefix.

## 9. Control Channel

### 9.1 Handshake and session start

1. The client opens a TCP connection to `<ip>:<port>` and performs the §7.1 TLS
   verification.
2. The client sends the first application line, to bind this TLS session to the
   rendezvous:
   ```
   AUTH <token>
   ```
3. The bot validates the token (single-use, unexpired) and replies:
   ```
   OK <session-id>
   ```
   or `ERR 401 unauthorized`. If the token is invalid, replay, or expired, the bot
   SHALL send the error and close.

### 9.2 Tokens

* `<token>` and `<datatoken>` SHALL be at least 128 bits of CSPRNG output, encoded
  as `base64url` without padding, and SHALL be accepted at most once. A token SHALL
  expire; the TTL SHOULD be short (e.g. ≤ 120 s for the rendezvous token).
* Because the token travels over IRC (untrusted) but the **datatoken** travels over
  the authenticated control channel, only the rendezvous token may be observed by the
  IRC network.

### 9.3 Framing

The control channel uses **CRLF-terminated** lines, UTF-8 encoded. A line SHALL NOT
exceed 8192 bytes. This mirrors the DCC/XDCC text model and keeps the channel
trivially debuggable with existing TLS tooling.

### 9.4 Command vocabulary

The XDCC vocabulary is carried over the authenticated control channel. The initial
command MAY be the one named by the IRC `sxdcc` trigger (§8), in which case the bot
executes it after `AUTH` without the client repeating it; the client MAY also issue
further commands over the control channel for the rest of the session.

```
LIST
SEND <pack>
BATCH <start>-<end>
INFO <pack>
OWNER
CANCEL [<pack>]
QUIT
```

`<pack>` is a base-10 integer; implementations SHOULD accept both `SEND 1` and
`SEND #1`. Commands and their responses — including pack lists and filenames — SHALL
be exchanged only over the TLS control channel, never over IRC.

### 9.5 Status lines

Bot → client lines:

| Line | Meaning |
|---|---|
| `OK [<detail>]` | Success acknowledgement. |
| `ERR <code> <message>` | Failure (`401` unauthorized, `404` unknown pack, `409` busy, `429` throttled, `500` internal). |
| `QUEUE <position> [<eta>]` | Request queued; `position` is 1-based. |
| `READY` | A queued request became active. |
| `SEND #<pack> <filesize> "<filename>" <ip> <port> <datatoken>` | Open a data channel (active). |
| `SEND #<pack> <filesize> "<filename>" 0 <datatoken>` | Open a data channel (passive/reverse; `port` 0). |
| `DONE #<pack> [<size>] [sha256:<digest>]` | Pack fully streamed and verified. |
| `CLOSE [<reason>]` | Bot is closing the control channel. |

`<filename>` is REQUIRED on the `SEND` line and SHALL be double-quoted. Pack-list
responses (`LIST`) reuse the `XDCC.md` §7 line format (`#<pack> [<size>] <name>`) and,
like every filename, are returned only on the control channel — never over IRC — so
the filename metadata is not exposed to the IRC network.

## 10. Data Channel

Each pack is streamed on a **separate** TLS 1.3 connection, mirroring DCC SEND but
without the cleartext handshake or the 32-bit ACK.

The **TLS roles are independent of the TCP direction**: in every mode the **bot is
the TLS server** (it presents its certificate) and the **client is the TLS client**
(it verifies the bot's SPKI pin via §7.1). Only the TCP connect direction — who
binds the listening socket and who dials out — varies between active and passive,
exactly as in classic passive/reverse DCC.

### 10.1 Establishment

* **Active.** For `SEND #<pack> <filesize> "<filename>" <ip> <port> <datatoken>` (a
  non-zero `<port>`), the client dials `<ip>:<port>` as the TLS **client**; the bot
  accepts and acts as the TLS **server**.
* **Passive/reverse.** For `port` `0`, the roles of the *TCP* endpoints swap (the
  client listens, the bot dials) but the *TLS* roles do **not** swap. The client
  opens a listening socket and replies on the control channel:
  ```
  ACCEPT <datatoken> <ip> <port>
  ```
  The bot then dials `<ip>:<port>` as the TLS **server** (it waits for `ClientHello`
  and presents its certificate); the client accepts the connection as the TLS
  **client** (it sends `ClientHello` and verifies the bot's pin).

### 10.2 Transfer

1. **The origin of the data connection** (the dialer) sends the binding line:
   ```
   DATA <datatoken>
   ```
   In active mode the client is the origin and sends it; in passive mode the bot is
   the origin and sends it.
2. **The listener validates** the `datatoken` (single-use, bound to this pack),
   matching it to the token agreed on the control channel. If invalid or unknown, it
   closes the connection without sending or accepting data.
3. The bot streams the **raw file bytes** to EOF. No ACK records are used: TLS 1.3
   already provides reliable, ordered, integrity-protected delivery.
4. The client SHALL verify the received byte count equals `<filesize>` from the
   `SEND` line, and SHOULD verify the optional `sha256:<digest>` from the `DONE`
   line against a digest computed during receipt.
5. On EOF the data connection is closed. The bot sends `DONE #<pack> …` on the
   control channel.

## 11. State Machine

The client-side lifecycle of one pack:

```
                    ┌───────────────────────────────┐
                    │            IDLE               │
                    └───────────────┬───────────────┘
                                    │  SEND <pack>
                                    ▼
                    ┌───────────────────────────────┐
                    │           REQUESTED            │
                    └───────────────┬───────────────┘
                          ┌─────────┴─────────┐
                        slot free        slot busy
                          │                  │
                          ▼                  ▼
                      CONNECTING        QUEUED ──(READY)──▶ CONNECTING
                          │                  ▲
                          │(data TLS)        │
                          ▼                  │
                       STREAMING             │
                          │                  │
                        EOF+count OK         │
                          │                  │
                          ▼                  │
                    COMPLETE  · FAILED ◀─────┘ (error/timeout/cancel)
```

| State | Meaning |
|---|---|
| `IDLE` | Control channel up; no active request. |
| `REQUESTED` | `SEND` issued, awaiting slot/fetch decision. |
| `QUEUED` | `QUEUE <position>` received; waiting. |
| `CONNECTING` | Establishing the data TLS channel. |
| `STREAMING` | Receiving raw pack bytes. |
| `COMPLETE` | EOF reached, byte count (and optional digest) verified. |
| `FAILED` | Error, timeout, invalid token, or `CANCEL`. |

## 12. Normative Requirements

* REQ-1: SXDCC implementations SHALL use TLS 1.3 (or later) on both channels and
  SHALL NOT fall back to TLS 1.2.
* REQ-2: The client SHALL authenticate the server by SPKI pin on every connection
  (§7.1) and SHALL verify the certificate IP SAN.
* REQ-3: The client SHALL NOT prompt the user; all trust decisions SHALL be
  automatic and fail closed.
* REQ-4: Verification of the pin SHALL use a constant-time comparison.
* REQ-5: The rendezvous SHALL be a plain `NOTICE`/`PRIVMSG`, not a CTCP request.
* REQ-6: The client SHALL present the rendezvous token as the first control-channel
  line (`AUTH <token>`), and the bot SHALL reject invalid, replayed, or expired
  tokens.
* REQ-7: Every data channel SHALL be bound by a single-use `datatoken` communicated
  exclusively over the authenticated control channel.
* REQ-8: Data transfers SHALL NOT use DCC ACK records; completeness SHALL be
  determined by byte count (and optional `sha256:` digest).
* REQ-9: A conforming client SHALL keep the first-contact trust behaviour of §7.3
  (TOFU) when no explicit pin is configured.
* REQ-10: Filenames and the pack list SHALL be conveyed only over the TLS channels;
  the IRC `sxdcc` trigger SHALL carry at most a command verb and numeric arguments.
* REQ-11: The control-channel `SEND` line SHALL include the double-quoted pack
  filename.

## 13. Extensions and Open Items

### 13.1 Explicit pin override

Covered in §7.2 — present in v1.

### 13.2 Client authentication (not in v1; optional plugin)

SXDCC v1 has **no client-authentication gate**: the bot serves packs to any client
that completes the §6/§7 server-authentication handshake and presents a valid
rendezvous token. This is a deliberate choice to keep the protocol headless —
distinguishing *who* a client is would force an authorization decision that a
headless operator is not required to make at the protocol layer.

The rendezvous token therefore has a narrow, specific role only:

* **anti-unsolicited-connection** — an observer who learns the bot's `ip:port`
  cannot connect or enumerate packs without a fresh, single-use token; and
* **TLS↔rendezvous binding** — the token ties this TLS session to the IRC message
  that invited it.

It is **not** client authorization and **not** client identity: it is visible to the
IRC server and MUST NOT be treated as a client secret.

Implementations that later need to distinguish or restrict clients MAY add an
optional client-authentication plugin, additive to and interoperable with v1:

* **SPKI-pin auto-enrolment (continuity)** — record each client's self-signed
  certificate SPKI on first connection and require the same key thereafter;
  detects key substitution but leaves membership open.
* **IRC-account-anchored authorization** — bind a client's key to its registered IRC
  account and enforce an access list maintained on the IRC network.
* **TLS 1.3 external PSK or client certificate** — strongest mutual authentication,
  for operators willing to provision per-client credentials.

### 13.3 Single-port data multiplexing

An implementation MAY serve the data channel on the same address and port as the
control channel, so the bot needs no separate data-port range. In this mode the
bot's `SEND` line advertises the control address, and the client opens a second
TLS 1.3 connection to that address and sends `DATA <datatoken>` as its first
line. The bot's control listener demultiplexes on the first line — `AUTH <token>`
begins a control session, `DATA <datatoken>` begins a data connection — and
streams the pack on the latter. This keeps the bot's inbound footprint at a
single shared port while remaining compatible with the active data channel of §10.

### 13.4 CA-based no-touch rotation

Reserved. If operator policy later requires server-key rotation without re-pinning
clients, a private CA root MAY be introduced; clients would then pin the CA root
instead of the leaf SPKI. This is a breaking change to the §6 trust anchor.

## 14. Security Properties and Threat Model

| Property | Mechanism |
|---|---|
| Confidentiality | TLS 1.3 AEAD on control and data channels. |
| Integrity | TLS 1.3 AEAD; optional `sha256:` completion digest. |
| Forward secrecy | TLS 1.3 ECDHE (X25519) per handshake. |
| Server authentication | Pinned SPKI + self-signed leaf + IP SAN (no DNS, no public CA). |
| Data-channel binding | Single-use `datatoken` over the authenticated control channel. |
| Metadata minimization | IRC observes only the rendezvous. |

Residual threats:

* **The IRC server is untrusted.** It can observe (and, if malicious, modify) the
  rendezvous message. It cannot impersonate the **bot** (client verifies the pin),
  but it could race a consumer's single-use token to connect in the consumer's place
  (a slot-stealing / denial-of-service race, since membership is open in v1). This is
  bounded by token TTL and single-use, and is fully closed only by the optional
  client-authentication plugin of §13.2.
* **TOFU first contact.** Without an explicit pin, the first rendezvous with a new
  identity is trusted on sight. Operators with a secure side channel SHOULD provision
  the explicit pin to eliminate this window.

## 15. Examples

### 15.1 List and download (active data channel)

```
# IRC (cleartext, untrusted) — request + invitation:
Client → Bot  PRIVMSG: sxdcc list
Bot   → Client  NOTICE:  SXDCC CONNECT 3232235778 40443 9C6f…   sha256:<pin>

# Control channel (TLS 1.3):
Client → Bot  AUTH 9C6f…
Bot   → Client  OK s1
Bot   → Client  #1 [ 1.2G] series.01.mkv
Bot   → Client  #2 [ 1.1G] series.02.mkv
Client → Bot  SEND 1
Bot   → Client  SEND #1 1234567890 "series.01.mkv" 3232235778 40444 3Aaa…
# Data channel (TLS 1.3), client connects to ip:40444:
Client → Bot  DATA 3Aaa…
Bot   → Client  <1234567890 raw bytes> EOF
# Control channel:
Bot   → Client  DONE #1 1234567890 sha256:2fB7…
Client → Bot  QUIT
Bot   → Client  BYE
```

### 15.2 Queued request (passive data channel)

```
# IRC (cleartext, untrusted) — request + invitation:
Client → Bot  PRIVMSG: sxdcc send #2
Bot   → Client  NOTICE:  SXDCC CONNECT 3232235778 40443 9C6f…   sha256:<pin>

# Control channel (TLS 1.3):
Client → Bot  AUTH 9C6f…
Bot   → Client  OK s2
Bot   → Client  QUEUE 3
… later …
Bot   → Client  READY
Bot   → Client  SEND #2 1099511627 "series.02.mkv" 0 55cd…    # port 0 → passive
Client → Bot  ACCEPT 55cd… 3232235779 50100
# Bot connects to client at ip:50100 (TLS), then:
Bot   → Client  DATA 55cd…
Bot   → Client  <…> EOF
Bot   → Client  DONE #2 1099511627
```

## 16. References

* RFC 8446 — The Transport Layer Security (TLS) Protocol Version 1.3.
* RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels.
* RFC 7469 (historical) — Public Key Pinning Extension for HTTP (SPKI fingerprint
  form).
* XDCC (`XDCC.md`) — the application layer SXDCC reuses.
* DCC (`DCC.md`) — the address encoding reused by the rendezvous message.