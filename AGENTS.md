---
id: irc/dcc-protocol
name: dcc-protocol (specs)
group: IRC
status: active
stack: [markdown]
docs: [IRC/dcc-protocol/XDCC.md, IRC/dcc-protocol/SXDCC.md, IRC/dcc-protocol/DCC.md, IRC/dcc-protocol/SDCC.md]
related: [irc/iroffer-dinoga, irc/zoitechat, irc/halloy, irc/irc-test-client, nibl/niblirc]
keywords: [xdcc, sxdcc, dcc, sdcc, protocol reference, wire format]
revision: 15848b61cfe7366606edfd56a792c50ee1ee48f6
reviewed: 2026-11-22
---

The authoritative XDCC/SXDCC/DCC/SDCC protocol reference implemented by the
`IRC/` clients and `irc/iroffer-dinoga`; parts will flow into production
`nibl/niblirc`. When protocol behaviour changes anywhere, update these specs in
the same change as the implementation — they are the contract all clients and
servers are measured against.
