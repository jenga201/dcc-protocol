# eXtended DCC (XDCC) Protocol Specification

| Field | Value |
|---|---|
| Title | eXtended DCC (XDCC) |
| Short name | XDCC |
| Version | 1.0 |
| Status | Draft |
| Date | 2026-11-02 |
| Authors | NIBL engineering |
| Builds on | IRC (RFC 1459 / 2812), CTCP, DCC (`DCC.md`) |
| Audience | Implementers of XDCC bots, IRC clients, indexers, and the SXDCC layer |
| Reference implementations | iroffer-dinoex (C), PircBotX-based clients and indexers (niblirc), irc-test-client |

---

## 1. Abstract

XDCC is a **de facto** file-distribution convention built on top of IRC and DCC. It
defines:

* a small **command vocabulary** that a requesting user sends to an XDCC **bot** over
  IRC `PRIVMSG`;
* a **pack list** data format that enumerates the files a bot offers;
* a **queue / slot** model for serializing concurrent downloads; and
* the binding of each download to the DCC SEND handshake defined in `DCC.md`.

XDCC originated in 1994 as an ircII script ("Xabi DCC" / "eXtended DCC") and has no
single formal RFC; interoperability therefore relies on the conventions documented
here and the widely deployed behaviours of iroffer and mIRC-compatible clients.

## 2. Conventions and Terminology

The key words **MUST**, **MUST NOT**, **SHALL**, **SHALL NOT**, **SHOULD**,
**SHOULD NOT**, **MAY**, and **OPTIONAL** are interpreted per RFC 2119. Terms
defined in `DCC.md` (§2) apply here as well.

| Term | Meaning |
|---|---|
| XDCC bot | An IRC-connected program that serves files ("packs") on demand. |
| Pack | A single downloadable unit, identified by a **pack number**. One pack is delivered as one DCC SEND transfer (or, for a batch, one of a sequence). |
| Pack list | The ordered enumeration of packs a bot offers. |
| Requesting user | The IRC user who sends an XDCC command to a bot. |
| Slot | A concurrent-transfer position on the bot. |
| Queue | An ordered wait-list for transfer slots when none are free. |

## 3. Overview and Scope

### 3.1 In scope

* The `PRIVMSG` command vocabulary (`xdcc list`, `xdcc send`, …).
* The pack list line format.
* The queue / slot model and its observable messages.
* The binding of a pack to a DCC SEND transfer.

### 3.2 Out of scope

* The DCC transport itself — see `DCC.md`.
* TLS / confidentiality — see `SXDCC.md`.
* Bot administration, upload (FTP/HTTP), and anti-abuse policies beyond the
  protocol-visible queue/slot behaviour.

## 4. Actors and Roles

| Actor | Role |
|---|---|
| XDCC bot | Advertises packs, accepts `xdcc` commands, queues requesters, and initiates DCC SEND transfers to deliver packs. |
| Requesting user | Discovers a bot's packs (via list or channel advertisment), requests a pack, and receives it over DCC SEND. |
| Indexer / crawler | A non-downloading client that issues `xdcc list` and parses the pack list (e.g. niblirc). It MAY not complete any DCC transfer. |

## 5. Transport and Layering

XDCC reuses IRC for control and DCC for data:

```
+-----------------------------------------------------------------------+
| XDCC  :  xdcc list / xdcc send   (PRIVMSG command vocabulary)         |
+-----------------------------------------------------------------------+
| DCC   :  DCC SEND / RESUME / ACCEPT   (CTCP handshakes; data channel) |
+-----------------------------------------------------------------------+
| IRC   :  PRIVMSG / NOTICE  (routing & the control channel itself)     |
+-----------------------------------------------------------------------+
| TCP / TLS                                                             |
+-----------------------------------------------------------------------+
```

* All XDCC commands are sent to the bot via IRC `PRIVMSG` (a private message or
  query). The command is the plain-text message body.
* The bot uses `PRIVMSG` / `NOTICE` to reply with status and list text.
* Delivery always terminates in a DCC SEND handshake (active or passive).

## 6. Command Vocabulary

Commands are **whitespace-delimited**, case-insensitive tokens sent as a single
`PRIVMSG`:

```
PRIVMSG <bot> :xdcc <verb> [<args...>]
```

| Command | Meaning | Notes |
|---|---|---|
| `xdcc list` | Request the pack list. | MAY be disabled or restricted. Often returned as one pack per line via PRIVMSG/NOTICE. |
| `xdcc send #<pack>` | Request the pack with the given number. | `#` is literal; `<pack>` is a base-10 integer. |
| `xdcc send -1` | A conventional "whole list" / whole-pack request used by some bots and indexers. | Non-normative; meaning is bot-defined. |
| `xdcc batch #<start>-#<end>` | Request a contiguous range of packs as one queue entry. | Bot-defined; delivered as sequential DCC SENDs. |
| `xdcc remove` / `xdcc cancel` | Cancel a queued or in-progress request. | SHOULD be supported. |
| `xdcc info #<pack>` | Request metadata for a pack. | OPTIONAL. |
| `xdcc owner` | Request the owner/maintainer identity of a bot. | OPTIONAL; used by indexers. |

### 6.1 Requesting a pack

The canonical download request is:

```
PRIVMSG <bot> :xdcc send #<pack>
```

The bot SHALL either:

1. begin delivery of the pack (enqueue it if no slot is free); or
2. reply with an error/status message if the pack number is unknown or the request
   is not permitted.

## 7. Pack List Format

The `xdcc list` response is one pack per line. The normative line shape is:

```
#<pack> <count>x [<size>] <filename>
```

| Field | Meaning |
|---|---|
| `#<pack>` | Pack number (base-10 integer). |
| `<count>x` | A multiplier / "gets" annotation carried by many bots but ignorable by parsers. Parsers SHALL tolerate its absence. |
| `[<size>]` | Human-readable size in square brackets, e.g. `[1.2G]`, `[512M]`, `[ 42K]`. |
| `<filename>` | File name; MAY contain spaces and MAY be quoted. |

Example:

```
** XDCC PACK LIST **
#1 1x [ 1.2G] series.episode.01.mkv
#2 1x [ 1.1G] series.episode.02.mkv
#3 3x [  42M] extras.part.rar
```

A parser SHALL extract the pack number (the integer immediately after `#`), the
size (text between the first `[` and `]`), and the filename (all remaining text
after `]`). Bots MAY prefix or suffix the listing with banner or status lines;
parsers SHALL ignore lines that do not begin with `#<integer>`.

Implementations use a numeric multiplier and a SI/IEC size unit for the size
field; because the exact unit table and multiplier semantics are not standardised,
the size field SHOULD be treated as **informational** and corrected against the
`filesize` field of the eventual DCC SEND handshake.

## 8. Queue and Slot Model

Bots limit the number of concurrent transfers (`slots`) and serialise excess demand
through a **queue**.

* A bot SHALL have a finite, non-negative number of slots.
* When a valid `xdcc send` arrives and a slot is free, the bot SHALL initiate a DCC
  SEND transfer for that pack.
* When no slot is free, the bot SHALL place the request in the queue in arrival
  order and inform the user of their position. The exact message text is
  bot-defined; a conforming client SHOULD treat any `PRIVMSG`/`NOTICE` mentioning a
  queue or slot as a transient status and keep waiting, rather than parse a
  hard-coded string.
* When a slot frees, the bot SHALL dequeue the oldest entry and begin its transfer.

```
 REQUEST ──▶ ┌─ slot free? ──yes──▶ START DCC SEND ──▶ COMPLETE
             │
             └── no ──▶ ENQUEUE (report position) ──▶ (slot frees) ──▶ START DCC SEND
```

## 9. Transfer Binding and Batch

* Each pack is delivered as exactly one DCC SEND transfer (protocol in `DCC.md`,
  §7). A bot MAY advertise the pack's total size in the `filesize` field.
* For `xdcc batch`, the bot SHALL deliver each pack of the range as a **separate,
  sequential** DCC SEND transfer on the same queue entry. The receiver SHOULD accept
  each transfer and SHALL NOT assume concurrent delivery.
* A transfer that fails mid-stream MAY be cancelled; resume is supported through the
  DCC RESUME/ACCEPT flow (`DCC.md`, §8) and is RECOMMENDED for large packs.

## 10. State Machine

The state of a single download request on the bot side:

```
                         ┌────────────── REQUESTED ──────────────┐
                         │   xdcc send #N                        │
                         ▼                                       │
             ┌──── Slots_available? ────no────▶ QUEUED ──(slot)──┘
             │            │
             │          yes
             ▼            ▼
        SENDING ◀───── DEQUEUED
             │
             │  DCC SEND completes / fails
             ▼
        COMPLETE / FAILED / CANCELLED
```

| State | Meaning |
|---|---|
| `REQUESTED` | A valid `xdcc send` was received. |
| `QUEUED` | Awaiting a free slot; user has been informed. |
| `SENDING` | A DCC SEND data phase is in progress. |
| `COMPLETE` | The pack was fully sent (and acknowledged). |
| `FAILED` | Transfer aborted by error or peer. |
| `CANCELLED` | User or bot cancelled via `xdcc remove`/`cancel`. |

## 11. Normative Requirements

* REQ-1: A bot SHALL accept `xdcc send #<pack>` as a `PRIVMSG` message body and
  respond by delivering or queueing the requested pack.
* REQ-2: A bot SHALL deliver each pack via the DCC SEND handshake defined in
  `DCC.md`.
* REQ-3: A bot SHALL serialise excess demand through a queue and SHALL emit an
  observable status message when a request is queued.
* REQ-4: A `xdcc list` response SHALL present one pack per line in the `#<pack>
  [<size>] <filename>` shape described in §7.
* REQ-5: A conforming parser SHALL ignore non-pack lines and SHALL tolerate a
  missing `<count>x` field and quoted filenames.
* REQ-6: A client SHALL treat the `filesize` field of the DCC SEND handshake as
  authoritative over the informational list size.
* REQ-7: A bot MAY disable or restrict `xdcc list`, `xdcc info`, and `xdcc owner`,
  and SHOULD reply with a status/error line when it does.

## 12. Security Considerations

XDCC inherits every DCC security limitation (`DCC.md`, §12): control and data are
cleartext, unauthenticated, and tamperable; the data channel lacks a checksum or
MAC; and the address of each peer is disclosed to the IRC network. Distributing
content via XDCC is itself frequently used for unauthorised file sharing, and XDCC
bots are commonly run on compromised hosts; implementers SHOULD apply input
validation to every numeric and string field (pack number, size, filename) to avoid
the historical client-side parsing bugs associated with DCC filenames and port `0`.

These properties motivate the secure successor, SXDCC (`SXDCC.md`), which wraps the
same command vocabulary and DCC transfer in an authenticated, confidential TLS 1.3
transport.

## 13. Examples

### 13.1 List and download

```
User                                  Bot
  │ PRIVMSG Bot :xdcc list             │
  │───────────────────────────────────▶│
  ◀─────────────────────────────────── │ NOTICE: ** XDCC PACK LIST **
  ◀─────────────────────────────────── │ NOTICE: #1 1x [ 1.2G] series.01.mkv
  ◀─────────────────────────────────── │ NOTICE: #2 1x [ 1.1G] series.02.mkv
  │ PRIVMSG Bot :xdcc send #1          │
  │───────────────────────────────────▶│
  ◀─────────────────────────────────── │ CTCP DCC SEND series.01.mkv <ip> <port> <filesize>
  │ TCP → ip:port, receive + ACK        │
  ════════════════════════════════════▶│
```

### 13.2 Queued request

```
User                                   Bot
  │ PRIVMSG Bot :xdcc send #5           │
  │────────────────────────────────────▶│
  ◀──────────────────────────────────── │ NOTICE: ** All slots full, queued as #3 **
  │ ... wait ...                        │
  ◀──────────────────────────────────── │ CTCP DCC SEND series.05.mkv <ip> <port> <filesize>
```

## 14. References

* RFC 1459 — Internet Relay Chat Protocol.
* RFC 2812 — Internet Relay Chat: Client Protocol.
* RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels.
* XDCC (Wikipedia) — history and de-facto usage of the XDCC convention.
* Direct Client-to-Client (Wikipedia) — the DCC substrate.
* iroffer-dinoex — XDCC bot reference implementation (configuration directives:
  `xdcc list`/`send`, `slotsmax`, `queuesize`, `xdcclistfile`, passive DCC).
* niblirc `updatepacklist` plugin — XDCC pack-list consumer and parsers.
* irc-test-client `XdccTestClient` — minimal PircBotX XDCC requester/receiver.