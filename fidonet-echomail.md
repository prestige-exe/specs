# FidoNet Echomail – FTS-0004 / FTS-0005

FTS-0004 ("EchoMail Specification") by Bob Hartman, Spark Software,
1991. FTS-0005 ("The Distribution Nodelist") supplements FTS-0004 with
the topology required to make echomail work between nets.

Echomail is FidoNet's distributed conference / discussion system: a
message posted on one BBS appears on every BBS subscribed to the same
"conference" (or "echo") within hours. It is the technical ancestor of
Usenet from the BBS side – a flat conference system that propagated by
nightly modem polls between nodes.

## Scope

This document covers:

- The echomail message format that travels inside FTS-0001 packets.
- The `SEEN-BY` and `^APATH` control lines that prevent duplicate
  propagation.
- The conference naming convention.
- The bundling and tossing workflow.

It does not cover routing topology, hub/star relationships, or the
mechanics of subscribing to an echo from a feed – those are
operational matters above the FTS-0004 layer.

## An echomail message in a packet

An echomail message uses the FTS-0001 type-2 packet and message header
unchanged, with two differences from netmail:

1. The first line of the message body is an `AREA:` tag.
2. The Attribute word's Private bit (0x0001) is clear.

```
AREA:GENERAL.CHAT
   ... message text ...
--- editor tag
 * Origin: My BBS (1:123/45)
SEEN-BY: 123/45 67 89 123/100 250/107 380/3
^APATH: 123/45 250/107 380/3
```

The `AREA:` line is unprefixed (no `^A`); the leading characters
`A`, `R`, `E`, `A`, `:` mark it as the conference identifier. Everything
between `AREA:` and the tear-line `---` is the user-visible body.

### AREA tag

```
AREA:<echotag>
```

`<echotag>` is the conference name, conventionally:

- Up to 35 characters.
- Letters, digits, dot, underscore, dash. No spaces.
- Uppercase by convention.

Examples: `GENERAL.CHAT`, `R20_GENERAL`, `SYSOP.HELP`, `FIDONEWS`,
`COMP.OS.UNIX` (a Usenet-mirrored echo).

The conference name is the only key linking the message to its echo;
all routing decisions are made by tossers reading the `AREA:` line.

## SEEN-BY

The `SEEN-BY:` line(s) at the end of the body list every node address
that has already seen this message. Format:

```
SEEN-BY: <net>/<node> <node> <node> ... <net>/<node> <node> ...
```

The leading `SEEN-BY:` may be repeated on multiple lines; the
addresses are cumulative. Within a net, nodes that share the net
number can be listed as just `<node>` after a single `<net>/<node>`
entry. Lines must not exceed 79 characters; long lists wrap to new
`SEEN-BY:` lines.

A tosser MUST not send the message to any address that is already in
the SEEN-BY list. After processing, the tosser adds its own address
plus every outbound destination to the list before forwarding.

`SEEN-BY:` lines are unprefixed text (no `^A`). They are visible to
human readers though most editors hide them.

## ^APATH

```
^APATH: <addr> <addr> <addr> ...
```

PATH is a `^A`-prefixed kludge line listing the nodes the message has
traversed, *in order*. Each tosser appends its own address before
forwarding. Unlike `SEEN-BY`, `PATH` lists only nodes that actually
relayed the message, not every recipient. Unlike `SEEN-BY`, it is
prefixed with `^A` (0x01) and is therefore hidden from human readers
by most editors.

PATH is what loop-detection actually uses; SEEN-BY is the canonical
"don't send back" filter.

## Origin line

```
 * Origin: <free text> (<zone>:<net>/<node>[.<point>])
```

A single line, leading space, asterisk, "Origin:", a free-form site
name, and the originating address in parentheses. The Origin line is
the canonical end of the user-readable body; tossers consider
everything after the next blank line (typically the SEEN-BY) to be
control material.

## Tear line

```
---
```

or

```
--- <editor-id>
```

Three hyphens at the start of a line, optionally followed by a space
and the editor's identification. Marks the boundary between body and
the editor's "signature" – everything after the tear line and before
the Origin is the tagline emitted by the message editor.

## Tossing and scanning

In FidoNet terminology:

- **Scanning** is the operation that reads new echomail messages
  written by local users (in `*.MSG`, JAM, Squish, etc.) and packs
  them into outbound packets.
- **Tossing** is the inverse: reading inbound `.PKT` files,
  extracting messages, deciding which echoes they belong to, and
  writing them into the local message bases.

A tosser implements:

1. Read inbound `.PKT`.
2. For each message, read AREA: tag.
3. Discard messages whose own SEEN-BY already includes our address.
4. Write the message to the local message base for that echo.
5. For each downstream link subscribed to this echo, append a copy of
   the message to the outbound packet for that link, with our address
   added to SEEN-BY and PATH.

The scanner mirrors this for locally-written messages.

## Bundling

Outbound `.PKT` files are typically compressed and concatenated into
larger "bundles" before being shipped. Bundle naming convention:

```
<NNNNNNNN>.<XXX>
```

where `<NNNNNNNN>` is a hex form of the destination address and
`<XXX>` is one of:

| Extension | Compression |
| --- | --- |
| `.PKT` | None – raw packet |
| `.MO`, `.TU`, `.WE`, `.TH`, `.FR`, `.SA`, `.SU` | Day-of-week tags for ARC bundles |
| `.ZIP` | PKZIP |
| `.ARJ` | ARJ |
| `.LZH` | LHA |
| `.RAR` | RAR |
| `.ARC` | SEA ARC |

The day-of-week tags pre-date ZIP and were originally used to roll
ARC bundles for a particular day's mail run.

## Dupe detection

Even with SEEN-BY and PATH, dupes happen – a node that disappears
mid-run, a link reset, or a propagation loop opened by a topology
change. Tossers maintain a "dupe database" keyed by:

- MSGID kludge line (if present, FTS-0009) – strongly preferred.
- CRC-32 of `(area, from, to, subject, datetime, first N bytes of
  body)` – fallback for messages without MSGID.

Dupes are silently dropped.

## Conference list (NODELIST / ELIST)

FTS-0005 covers the nodelist; the conference list is an informal
analogue:

- **ELIST**: a flat text file listing every echo carried in the local
  region, who moderates it, and the language/character set. Format
  defined in FSC-0064.
- **AREAS.BBS** (Squish convention): one line per area, format
  `<base-file> <echotag> [<link> <link> ...]`.

## What echomail enabled

- The classic FidoNet conferences – `SYSOP`, `COMM`, `R20_*`,
  `NET_DEV`, the language groups, the gateway echoes for Usenet news.
- The "FidoNews" weekly newsletter.
- Local-net "BBS_ADS" classifieds.
- Coordination of FidoNet itself (the `Z*C_NETMAIL` and `*C_DEVELOP`
  echoes).

Echomail's structural problems – flat namespace, late dupe detection,
fragile loop handling, propagation latency – are exactly the reasons
the IETF eventually settled on NNTP/IMAP instead. But within its
constraints (modem links, batched delivery, no central server),
FTS-0004 delivered a continent-wide conference system that worked.
