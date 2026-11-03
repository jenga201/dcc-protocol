# SXDCC Transport Extension — NAT Traversal, QUIC, and Relay Specification

| Field | Value |
|---|---|
| Title | SXDCC Transport Extension (NAT Traversal) |
| Short name | SXDCC-TRANSPORT |
| Version | 1.0-draft |
| Status | Draft |
| Date | 2026-11-03 |
| Authors | NIBL engineering |
| Builds on | SXDCC (`SXDCC.md`), XDCC (`XDCC.md`), DCC (`DCC.md`), IRC (RFC 1459 / 2812), TLS 1.3 (RFC 8446), QUIC (RFC 9000, RFC 9001), ICE (RFC 8445), STUN (RFC 8489), TURN (RFC 8656) |
| Audience | Implementers of SXDCC clients and bots, and operators of relay / bounce hosts |
| Reference implementation | dinoga `internal/sxdcc` (baseline `tcp` transport); QUIC/ICE/tunnel paths per this document (in progress) |

---

## 1. Abstract

This document extends `SXDCC.md` with a **transport layer** that makes the data
channel robust against Network Address Translation (NAT) and firewalls, without
changing SXDCC's security model. It adds:

* a pluggable **data transport** — `tcp` (the TLS 1.3-over-TCP channel already in
  `SXDCC.md`) and `quic` (a new TLS 1.3-over-UDP channel);
* an **ICE candidate exchange** that replaces the single `ip:port` of classic DCC
  with a set of reachable addresses, selected by standard connectivity checks; and
* two **relay modes** — a persistent reverse tunnel ("bounce") that gives the bot a
  stable public reachability with zero inbound ports, and an on-demand TURN relay —
  used only when no direct path exists.

The properties of SXDCC — server authentication by SPKI pin, confidentiality and
integrity by TLS 1.3, per-transfer `datatoken` binding, and IRC as an untrusted
rendezvous — are **preserved on every path**. The extension therefore pursues two
goals, in order:

1. **Reliability** — a transfer succeeds whenever the bot can make a single outbound
   connection, regardless of NAT type, CGNAT, or UDP-blocking, by falling back to a
   relay path.
2. **Privacy and cost** — when the two networks permit it, migrate to a direct
   hole-punched QUIC path so that no relay observes transport metadata and no relay
   pays the data transit.

Security is a constant of the TLS layer, not a function of path choice; the path
only trades **reliability** against **metadata privacy and relay cost**.

## 2. Conventions and Terminology

The key words **MUST**, **MUST NOT**, **SHALL**, **SHALL NOT**, **SHOULD**,
**SHOULD NOT**, **MAY**, and **OPTIONAL** are interpreted per RFC 2119. All terms
defined in `DCC.md`, `XDCC.md`, and `SXDCC.md` apply.

| Term | Meaning |
|---|---|
| Transport | The network layer that carries SXDCC control and data bytes: `tcp` (TLS 1.3 over TCP) or `quic` (TLS 1.3 over UDP). |
| Candidate | One reachable `<ip:port>` at which an endpoint can be contacted, tagged by type (host / server-reflexive / relay). |
| Server-reflexive candidate | An endpoint's **external** `ip:port` discovered via STUN. |
| Rendezvous | The IRC channel; used only to bootstrap, exactly as in `SXDCC.md`. |
| Connectivity check | The STUN-based probe defined by ICE (RFC 8445) that determines whether a candidate pair passes traffic. |
| Bounce (reverse tunnel) | A persistent, byte-forwarding relay to which the **bot** maintains one outbound connection; clients connect to the bounce to reach the bot. Also called a BNC or reverse proxy. |
| TURN relay | An on-demand relay (RFC 8656) both peers dial into for a single transfer. |

### 2.1 Transport model

The control channel remains a TLS 1.3-over-TCP connection as in `SXDCC.md` §9. The
**data** channel is transport-agnostic:

* `tcp` — the existing `SXDCC.md` §10 data channel (TLS 1.3 over a direct active or
  passive TCP connection, or over a bounce).
* `quic` — a QUIC connection (TLS 1.3 embedded in UDP) established over the
  candidate selected by ICE.

The two relay modes differ in architecture, not in what they carry:

* The **bounce** relays the *existing* `tcp` transport, byte-for-byte, making the
  bot reachable with no inbound ports. It is persistent and deployed once.
* A **TURN relay** relays *QUIC datagrams* per transfer. It is on-demand, symmetric,
  and used as the fallback candidate when both endpoints are transient peers.

## 3. Scope

### 3.1 In scope

* The `quic` data transport and its reuse of the SPKI-pin identity and `datatoken`.
* The ICE candidate exchange and selection of a reachable data path.
* The reverse-tunnel (bounce) transport and its trust boundary.
* The TURN relay candidate as the on-demand fallback.
* Opportunistic migration to a direct path at transfer setup.

### 3.2 Out of scope

* Mid-stream migration of an in-flight transfer (future; see QUIC connection
  migration, RFC 9000 §9).
* Modifying the control-channel command vocabulary (`SXDCC.md` §9) other than the
  additive `CANDIDATE` exchange defined here.
* Client (mutual) authentication — unchanged from `SXDCC.md` §13.2.
* The multiplexed single-connection framing reserved in `SXDCC.md` §13.3; QUIC
  streams replace it for the `quic` transport.

## 4. Architecture

```
                        IRC (unmodified, untrusted rendezvous)
                              ▲  PRIVMSG: sxdcc <verb>
                              │  NOTICE: SXDCC CONNECT <ip> <port> <token> <pin>
                              ▼
                       ┌──────────────────────┐
                       │        BOUNCE        │   optional, persistent, byte-forwarding
                       └────┬────────────┬────┘
                 outbound   │            │  inbound (client)
                 tunnel     │            │
                            ▼            ▼
            ┌───────────────┐            ┌───────────────┐
            │      BOT      │            │     CLIENT    │
            └───┬───────┬───┘            └───┬───────┬───┘
                │       │                    │       │
      control   │       │  data (tcp/relay)  │       │
      (TLS/TCP) │       │◄──────────────────►│       │
                │       │                    │       │
                │       │  data (quic/direct)│       │
                │       │◄═══════════════════►│       │  via hole-punched UDP
                └───────┘                    └───────┘
```

* The **control channel** is established first (directly or through the bounce) and
  carries the ICE candidate exchange.
* The **data channel** is established on the selected candidate pair: direct QUIC
  when possible, otherwise `tcp` through the bounce, or QUIC through TURN.

## 5. Actors and Roles

| Actor | Role |
|---|---|
| Bot | SXDCC server; always the TLS server on control and data; owns the long-term key bound by the SPKI pin; optionally holds a persistent outbound tunnel to a bounce. |
| Client | Connects out to the bot (directly or via a bounce); verifies the bot's SPKI pin; supplies its own candidates. |
| Bounce | A public, byte-forwarding relay that pairs the bot's outbound tunnel with inbound client connections. It MUST NOT terminate TLS. |
| TURN relay | An on-demand relay that forwards QUIC datagrams between the two peers for one transfer. |

## 6. Identity and Trust

The certificate, SPKI pin, IP SAN, and TOFU behaviour are **unchanged** from
`SXDCC.md` §6–§7. The bot's identity is bound to its SPKI pin regardless of
transport; the client verifies the pin on the control channel and on every data
channel, direct or relayed.

For the `quic` transport, QUIC's handshake embeds TLS 1.3 (RFC 9001), so:

* the bot's self-signed leaf certificate and IP SAN are presented inside QUIC's
  TLS handshake;
* the client verifies the SPKI pin exactly as it does for the `tcp` transport;
* X25519 and the cipher-suite requirements of `SXDCC.md` §6 SHALL apply unchanged.

Because the bounce relays bytes without terminating TLS, and TURN relays encrypted
QUIC datagrams, **neither relay ever sees plaintext or the pin-derived secret**; the
trust relationship remains only between client and bot.

## 7. Baseline Reachability — the Reverse Tunnel (Bounce)

When the bot has no public inbound path (no published ports, behind NAT), it uses a
bounce to remain reachable. This is the **reliability baseline**: it works whenever
the bot can hold one outbound connection, which it already does for IRC.

### 7.1 Establishment

1. The **bot** opens a persistent TLS 1.3 (or raw TCP) connection **outbound** to
   the bounce and authenticates to the bounce with an operator-provisioned secret.
2. The **client** connects **inbound** to the bounce's public address and port.
3. The **bounce** pairs the two connections by an operator or token-mediated rule
   and forwards bytes in both directions unchanged.

### 7.2 Trust boundary

* The bounce SHALL be a **byte-forwarding relay only**: it SHALL NOT terminate TLS,
  SHALL NOT present as the bot, and SHALL NOT re-encrypt.
* The client therefore still sees the bot's certificate and verifies the SPKI pin
  end-to-end. A compromise of the bounce can **observe** metadata (who talks to
  whom, volume, timing) but can **neither read nor forge** content and cannot
  impersonate the bot.
* The bot's IRC rendezvous (`SXDCC CONNECT …`) SHALL advertise the **bounce's**
  address when the bot uses a bounce.

## 8. Candidate Exchange (ICE)

For the `quic` transport, the single `ip:port` of classic DCC is replaced by a
candidate set exchanged over the authenticated control channel, followed by ICE
connectivity checks (RFC 8445).

### 8.1 Negotiation

On receiving `SEND #<pack>`, a bot supporting `quic` SHALL reply with the
`transport=quic` form, dropping the fixed `ip:port` fields:

```
SEND #<pack> <filesize> "<filename>" <datatoken> transport=quic
```

This SHALL be followed by a candidate exchange, scoped by `<datatoken>`:

```
CANDIDATE <datatoken> <id> <type> <priority> <ip> <port>
...
CANDIDATES-END <datatoken>
```

Both endpoints SHALL send at least one `CANDIDATE` and SHALL terminate their list
with `CANDIDATES-END`. Either endpoint MAY begin sending candidates immediately
after the `SEND` reply.

### 8.2 Candidate fields

| Field | Meaning |
|---|---|
| `<id>` | A local, transfer-scoped identifier for the candidate (opaque string). |
| `<type>` | `host` (a directly bound address), `srflx` (server-reflexive, via STUN), or `relay` (a TURN relay candidate). |
| `<priority>` | Unsigned integer; higher is preferred. Computed as ICE (RFC 8445 §5.1.2) from type, local preference, and component. |
| `<ip>` | Dotted-quad IPv4 (the DCC 32-bit integer form of `DCC.md` §5.1 is also accepted). IPv6 MAY follow the extension conventions of `DCC.md` §9.2. |
| `<port>` | 16-bit TCP/UDP port in decimal. |

These fields map directly onto the ICE candidate attribute (RFC 8445 §5.1); an
implementation MAY instead exchange full ICE attributes if both peers agree.

### 8.3 Connectivity checks and selection

Both endpoints SHALL run STUN-based connectivity checks (RFC 8445) over the
candidate pairs, ordered by `<priority>`, and select the **highest-priority pair
that passes** for the data channel. The `relay` candidates give a guaranteed path
when direct candidates fail. The selected pair's transport determines the data
channel: a `host`/`srflx` pair uses direct QUIC; a `relay` pair routes QUIC
datagrams through the TURN relay.

## 9. QUIC Data Transport

The data channel MAY be a QUIC connection in place of the `tcp` channel of
`SXDCC.md` §10.

* **Roles.** The bot is the QUIC (TLS 1.3) server and presents its certificate; the
  client is the QUIC (TLS 1.3) client and verifies the SPKI pin.
* **Binding.** After the QUIC connection is established, the origin (dialer) opens
  one bidirectional stream and sends:

  ```
  DATA <datatoken>
  ```

  followed by the raw file bytes to EOF. The listener validates the single-use
  `<datatoken>` against the transfer agreed on the control channel and, if invalid,
  closes the stream without accepting data (`SXDCC.md` §10.2).
* **Reliability.** No DCC ACK records are used: QUIC provides reliable, ordered,
  integrity-protected delivery. Completeness is determined by byte count against
  `<filesize>` and, when supplied, the `sha256:` digest on `DONE`.
* **Multiplexing.** QUIC streams MAY carry the control and data channels over one
  connection, in which case a length-prefixed framing as reserved in `SXDCC.md`
  §13.3 SHALL be used to separate streams; this OPTIONAL behaviour supersedes the
  reservation in `SXDCC.md` §13.3.

## 10. Relay Modes and Failover

### 10.1 Path preference

Implementations SHALL prefer paths in this order for the data channel:

1. **Direct** QUIC over a hole-punched `host`/`srflx` candidate (best privacy, no
   relay cost).
2. **TURN relay** QUIC (`relay` candidate) when direct fails and both endpoints are
   transient peers.
3. **Bounce** (`tcp` transport over the reverse tunnel) as the persistent baseline
   for a server/bot deployment — and the last-resort path for UDP-blocked clients.

The bounce and TURN are alternative relay architectures (persistent server-side vs.
on-demand symmetric): an implementation deploys the one matching its topology and
need not run both.

### 10.2 Opportunistic migration

* At `SEND` time the bot SHALL offer both the baseline `tcp` (bounce) path and, when
  both peers support it, the `quic` candidate exchange.
* The endpoints SHALL run the candidate exchange and ICE checks in parallel with
  the baseline; the data channel opens on the highest-priority working path.
* If the direct path is unavailable, the transfer proceeds over the bounce and the
  bot SHOULD re-probe direct reachability for subsequent packs.
* Mid-stream migration of an in-flight transfer is out of scope for v1.

## 11. Normative Requirements

* REQ-1: An implementation SHALL preserve `SXDCC.md` §6–§7 identity and trust on
  every transport; the client SHALL verify the bot's SPKI pin on control and data.
* REQ-2: A bot supporting `quic` SHALL reply to `SEND` with the `transport=quic`
  form and SHALL exchange candidates scoped by `<datatoken>`.
* REQ-3: A candidate SHALL carry the fields of §8.2, and each endpoint SHALL
  terminate its candidate list with `CANDIDATES-END`.
* REQ-4: Endpoints SHALL select the data path using ICE connectivity checks ordered
  by candidate `<priority>` (RFC 8445).
* REQ-5: A direct `host`/`srflx` path SHALL be preferred over a `relay` path.
* REQ-6: A `quic` data channel SHALL bind to the transfer with the single-use
  `DATA <datatoken>` line and SHALL NOT use DCC ACK records.
* REQ-7: A bounce SHALL be a byte-forwarding relay and SHALL NOT terminate TLS or
  present the bot's identity.
* REQ-8: When a bounce is used, the bot's rendezvous SHALL advertise the bounce's
  address.
* REQ-9: An implementation SHALL fall back to a relay path rather than fail the
  transfer when no direct path passes connectivity checks.
* REQ-10: A `relay` candidate SHALL identify a TURN relay (RFC 8656) over which the
  QUIC datagrams are forwarded.

## 12. Security Properties and Threat Model

| Property | Mechanism |
|---|---|
| Confidentiality | TLS 1.3 AEAD on control and data, direct and relayed. |
| Integrity | TLS 1.3 AEAD; optional `sha256:` completion digest. |
| Server authentication | Pinned SPKI + self-signed leaf + IP SAN, unchanged, verified end-to-end. |
| Forward secrecy | TLS 1.3 (EC)DHE — X25519 — per handshake. |
| Data-channel binding | Single-use `datatoken` over the authenticated control channel. |
| Metadata minimization | Direct path eliminates relay observation; relay paths expose volume/timing/identities to the relay only. |

Residual threats, in addition to those of `SXDCC.md` §14:

* **Relay metadata.** A compromised or hostile bounce/TURN operator can observe
  that two parties are exchanging data and how much, but cannot read or forge
  content and cannot impersonate the bot (the SPKI pin is end-to-end).
* **Direct-path unreliability.** Hole-punching depends on the NAT behaviour of both
  endpoints (cone mappings; symmetric-to-symmetric typically fails). The relay
  fallback bounds this to degraded (relayed) throughput rather than failure.
* **Slot-stealing via relay.** A relay can drop or re-order connections (a
  denial-of-service race), analogous to the IRC-server race in `SXDCC.md` §14; it
  is closed only by the optional client-authentication plugin of `SXDCC.md` §13.2.

## 13. Examples

### 13.1 Direct QUIC with candidate exchange

```
Bot                                   Client
  ◀────────────────────────────────── │  SEND #3
  │ SEND #3 2097152000 "episode.mkv"  │
  │        abc123 transport=quic      │───────────────────────────────▶
  │ CANDIDATE abc123 0 host    2130706431  192.0.2.10  40443          │ ▶
  │ CANDIDATE abc123 1 srflx   1694498815  203.0.113.7 40443          │ ▶
  │ CANDIDATES-END abc123                                              │ ▶
  │                                   ◀─────────────────────────────   │
  │                     CANDIDATE abc123 0 srflx 1694498815 198.51.100.9 50123
  │                     CANDIDATES-END abc123
  │      (ICE connectivity checks; srflx ↔ srflx passes)               │
  │      QUIC handshake; client verifies bot's SPKI pin                │
  │      stream: DATA abc123  →  raw bytes to EOF                      │
  │      (control) DONE #3 2097152000                                  │
```

### 13.2 Bounce baseline fallback

```
Bot          Bounce        Client
  │ TCP ↗ (persistent)       │
  │◄─────────────────────────│  inbound TCP, relayed to bot's tunnel
  │   SEND #4 ... <datatoken> transport=tcp  (through bounce)
  │   TLS 1.3 end-to-end (bounce forwards bytes, never terminates TLS)
  │   DATA <datatoken> + bytes ─────────────────────────▶
```

### 13.3 TURN relay fallback

```
Bot          TURN          Client
  │ ALLOCATE ↗   │  ALLOCATE ↗
  │   CANDIDATE ... relay ... (TURN address)
  │   QUIC datagrams relayed through TURN (end-to-end TLS 1.3)
```

## 14. References

* RFC 8445 — Interactive Connectivity Establishment (ICE).
* RFC 8489 — Session Traversal Utilities for NAT (STUN).
* RFC 8656 — Traversal Using Relays around NAT (TURN).
* RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport.
* RFC 9001 — Using TLS to Secure QUIC.
* RFC 8446 — The Transport Layer Security (TLS) Protocol Version 1.3.
* RFC 1459 / RFC 2812 — Internet Relay Chat.
* RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels.
* `DCC.md`, `XDCC.md`, `SXDCC.md` — the underlying protocol layers.
* dinoga `internal/sxdcc` — the SXDCC server implementation (`tcp` transport).
* pion (`pion/ice`, `pion/stun`, `pion/turn`) and `quic-go` — Go reference
  implementations for ICE/STUN/TURN and QUIC.