# Secure DCC Chat (SDCC) Protocol Specification

| Field | Value |
|---|---|
| Title | Secure DCC Chat (SDCC) |
| Short name | SDCC |
| Version | 1.0-draft |
| Status | Draft |
| Depends on | `SXDCC.md` (§6, §7: transport security and trust bootstrap), `DCC.md` |

## 1. Abstract

SDCC is a secure, interactive administrator console for XDCC/IRC bots. It runs
the bot's admin command loop over an authenticated TLS 1.3 connection, replacing
cleartext admin DCC `CHAT` and `telnet` administration. SDCC shares the bot's
single inbound TLS listener with SXDCC and is demultiplexed by the connection's
first application line.

## 2. Conventions and Terminology

Terms are as in `DCC.md` and `SXDCC.md`, plus:

* **operator** — the human who administers the bot.
* **admin credential** — the operator's secret that gates the console (the
  `adminpass` configuration value).
* **SDCC session** — one persistent admin-console connection.

"Bot" is the server; "client"/"operator" is the connector.

## 3. Goals and non-goals

### 3.1 Goals

* Confidentiality and integrity of the admin console (no cleartext equivalent).
* Coexistence with SXDCC file transfer on a single inbound port.
* Always-up and non-blocking: a chat session must not consume a transfer slot or
  block transfers.

### 3.2 Out of scope

* File transfer (that is SXDCC).
* Peer-to-peer chat between IRC users.
* Client identity beyond the operator credential.

## 4. Architecture and listener demultiplexing

SDCC reuses the SXDCC host's TLS listener and trust bootstrap (`SXDCC.md` §6/§7:
TLS 1.3, SPKI pin `sha256:<base64url>`, IP subjectAltName, IRC-anchored TOFU
store). The bot demultiplexes each accepted connection on its first line:

| First line | Protocol |
|---|---|
| `AUTH <token>` | SXDCC control session (`SXDCC.md` §9) |
| `DATA <datatoken>` | SXDCC data connection (`SXDCC.md` §13.3) |
| `ADMIN <token>` | SDCC admin chat session (this spec) |

Any other first line is a protocol error; the bot SHALL close.

## 5. Trigger and rendezvous

```
Client → Bot  PRIVMSG: sdcc chat
Bot   → Client  NOTICE:  SDCC CONNECT <ip> <port> <token> <pin>
```

| Field | Meaning |
|---|---|
| `SDCC CONNECT` | Literal invitation verb. |
| `<ip>` | Bot's address, DCC 32-bit network-order integer form (`DCC.md` §5.1). |
| `<port>` | The shared TLS listener port. |
| `<token>` | Single-use rendezvous token (§7). |
| `<pin>` | Bot's SPKI pin. |

The IRC channel is untrusted — the `pin` is an assertion the client binds to the
bot's IRC identity (TOFU, `SXDCC.md` §7.3).

## 6. Handshake

```
Client → Bot  ADMIN <token>
Bot   → Client  PASS
Client → Bot  PASS <password>
Bot   → Client  WELCOME
Bot   → Client  >
```

1. The client opens TLS and verifies the pin (`SXDCC.md` §7.1), then sends
   `ADMIN <token>` as the first application line.
2. The bot validates the token (single-use, unexpired); on failure it sends
   `ERR 401 unauthorized` and closes.
3. On success the bot challenges with `PASS`.
4. The client answers `PASS <password>`. The bot verifies the credential in
   constant time. On failure: `ERR 401 unauthorized` and close. On success:
   `WELCOME`, then the interactive console with a `> ` prompt.

## 7. Credentials

* `<token>` — at least 128 bits of CSPRNG output, `base64url` without padding,
  single-use, short TTL (≤ 120 s). It is an anti-unsolicited-connection gate that
  binds the connection to the IRC `sdcc chat` trigger. It is **not** admin
  authorization and is visible to the IRC server.
* `<password>` — the operator `adminpass` secret. The bot MUST verify it in
  constant time. It SHALL be sent only over the authenticated TLS channel, SHALL
  NOT be logged, SHALL NOT appear on IRC, and MUST NOT contain CR or LF. If no
  admin credential is configured, the bot MUST refuse every `ADMIN` attempt with
  `ERR 401 unauthorized`.

## 8. Chat framing

After `WELCOME`:

* The bot emits a `> ` prompt line (a CRLF line whose content is `> `) whenever
  it is ready for the next command.
* A command is one CRLF line, UTF-8, at most 8192 bytes (`SXDCC.md` §9.3).
* The bot writes zero or more response lines, then a fresh `> ` prompt. The
  client treats all lines between two prompts as the response to the last
  command.
* `quit`, `exit`, or `close` ends the session with `bye` and the connection
  closes.

The available commands are the bot's admin registry — at minimum `status`,
`list`, `help`, `quit`, plus any operator plugins. This is the same command
vocabulary served today by telnet administration.

## 9. Concurrency and availability

An SDCC session is an independent, long-lived goroutine on the shared listener.
It consumes no transfer slot — `slotsmax` bounds only `SEND` (`SXDCC.md` §9.4) —
so any number of concurrent SDCC sessions and SXDCC transfers coexist on the one
port without mutual blocking. A session remains "always up" until either side
closes it; implementations SHOULD cap idle sessions to bound resource use.

## 10. Security properties

| Property | Mechanism |
|---|---|
| Confidentiality / integrity | TLS 1.3 AEAD on the session. |
| Server authentication | Pinned SPKI + self-signed leaf + IP SAN. |
| Credential secrecy | Password only over TLS; never over IRC. |
| Rendezvous binding | Single-use token ties the session to the IRC trigger. |
| Availability | Non-blocking goroutine; no transfer-slot consumption. |

* The IRC server can observe (and modify) the invitation, but it cannot pass the
  `PASS` challenge without the admin credential and cannot impersonate the bot
  (the client verifies the pin). An IRC observer can race the single-use token to
  reach the `PASS` challenge — a denial of availability, not a confidentiality
  breach — bounded by token TTL and single-use.

## 11. Example

```
# IRC (cleartext, untrusted):
Client → Bot  PRIVMSG: sdcc chat
Bot   → Client  NOTICE:  SDCC CONNECT 3232235778 30000 9C6f… sha256:<pin>

# SDCC session (TLS 1.3):
Client → Bot  ADMIN 9C6f…
Bot   → Client  PASS
Client → Bot  PASS hunter2
Bot   → Client  WELCOME
Bot   → Client  >
Client → Bot  status
Bot   → Client  packs: 7
Bot   → Client  >
Client → Bot  quit
Bot   → Client  bye
```

## 12. Normative requirements

1. The bot SHALL demultiplex `ADMIN` in the same first-line parse as
   `AUTH`/`DATA`.
2. The bot SHALL verify the rendezvous token (single-use, unexpired) and the
   admin credential in constant time; failures SHALL be `ERR 401 unauthorized`
   followed by close.
3. The password SHALL NOT be logged or emitted anywhere except the `PASS` line
   on the TLS channel.
4. The client SHALL verify the server SPKI pin before sending `ADMIN`.
5. An SDCC session SHALL NOT consume a transfer slot.

## 13. References

* `SXDCC.md` — transport security, trust bootstrap, token policy, framing.
* `DCC.md` — DCC addressing and 32-bit address encoding.
* RFC 8446 — TLS 1.3; RFC 7469 — Public Key Pinning ("SPKI" digest).