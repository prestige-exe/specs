# TIC – File-echo companion file

Original TIC format and the `TICK` processor by Barry Geller, late 1980s.
Field extensions and the de-facto modern grammar from `AllFix` by
Harald Harms (Harms Software Engineering). Documented as FSC-0028 in
the FTSC document tree, with current practice further captured in
FSC-0087.

A `.TIC` file is the text manifest that accompanies a payload file as
it travels across a FidoNet file echo. Where echomail moves text
messages along a propagation tree, file echoes move binary archives
(art packs, software releases, nodelist updates) and a TIC is the
"envelope" of metadata, routing, and integrity information that lets a
downstream system decide whether to accept the file, where to file it,
and which downlinks to forward it to.

Pairs arrive together: `ARTPACK.ZIP` plus `ARTPACK.TIC`. The TIC is
plain CP437 text, one keyword per line, terminated by CR LF. Keywords
are case-insensitive and order-independent. A single space separates
the keyword from its value.

## Anatomy of a TIC

```
File ARTPACK.ZIP
Area ANSI
Desc ACiD May 1995 release
From 1:170/400
To 2:280/1029
Origin 1:170/400
Size 743421
Date 799459930
Crc 5D71E9D0
Replaces ARTP*.ZIP
Pw secret
App ALLFIX 6.00.023
Created by ALLFIX, v6.00.023
Path 1:170/400 799459930 Wed May 02 03:45:30 1995 UTC
Path 1:1/0 799512214 Wed May 02 18:23:34 1995 UTC
Seenby 1:1/0
Seenby 2:280/1
Seenby 2:280/1029
```

## Keywords

| Keyword | Value | Description |
| --- | --- | --- |
| `File` | filename | The payload file's name. 8.3 historically; long names appear in modern TICs and require the receiver to accept them. |
| `Area` | tag | The file-echo tag (matches the tosser's area definition). The TIC will not be imported if the receiver does not subscribe to this area. |
| `AreaDesc` | text | Human-readable description of the area; informational. |
| `Desc` | text | One-line description of the payload file. Multiple `Desc` lines are concatenated for multi-line descriptions. |
| `LDesc` | text | Long description line. Multiple lines stack. |
| `From` | `zone:net/node[.point]` | The sender of this hop (the system that emitted the TIC to you). |
| `To` | `zone:net/node[.point]` | The intended recipient (you). |
| `Origin` | `zone:net/node[.point]` | The system that originally hatched the file into the echo. Unchanged through the tree. |
| `Size` | decimal bytes | The payload file's size in bytes. |
| `Date` | Unix timestamp | File modification time as seconds since 1970-01-01. |
| `Crc` | 8 hex digits | CRC-32 of the payload file, used to detect corruption in transit. |
| `Path` | `<addr> <unixtime> <ctime>` | One line appended by each system that processes the TIC. The whole stack of Path lines, oldest first, is the propagation history. |
| `Seenby` | `zone:net/node[.point]` | One line per system that has seen the file. Receivers add their own address and the addresses of all downlinks they are about to forward to, so a system never receives a duplicate copy of the same file. |
| `Pw` | password | Per-link password for this file echo. Stripped before the TIC is forwarded; never appears in onward TICs. |
| `Replaces` | filespec | When the payload is imported, remove all existing files matching this filespec from the area first. Used for nodelist segments and similar rolling releases. |
| `App` | text | Application name and version that emitted the TIC. Informational. |
| `Created` | text | Free-form "created by" tag, conventionally `Created by <App>, v<n>`. |
| `Hatch` | text | Present on the original hatch TIC (the very first one), naming the operator who released the file into the echo. |
| `Magic` | name | Magic-name alias the file is also addressable as (e.g. `NODELIST`, `FILELIST`); used by file-request servers. |
| `Release` | id | Release identifier for a multi-file release set. |
| `ReleaseTo` | filespec | Files the release supersedes. |
| `TO` | text | Alias for `To` accepted by some processors. |

`From`, `To`, and `Origin` are 5D addresses; the `@domain` part is
optional and historically often absent.

## Propagation

A file echo, like a message echo, is a propagation tree rooted at the
system that hatched the file. At each hop:

1. The sending system writes a TIC listing itself in `From`, the
   receiver in `To`, its outbound password (if any) in `Pw`, appends
   its own `Path` line, and adds the receiver's address plus all of
   its other downlink addresses to the `Seenby` list it will write
   into the *next* outbound TIC.
2. The receiver verifies the `Crc` against the payload, checks `Pw`
   against the configured password for `From`, and looks up `Area` to
   determine where to file the payload locally.
3. The receiver writes a new TIC for each of its downlinks: copying
   the original `Origin` / `Desc` / `Crc` / `Size` / `Date` / `Replaces`,
   inserting its own `From`, the downlink's `To`, the outbound `Pw`,
   appending its own `Path` line, and merging the seen-by list (its own
   address plus every downlink address) into the outbound `Seenby`.
   Downlinks already listed in `Seenby` are skipped.

The `Path` stack is informational (audit trail). `Seenby` is the
loop-suppression mechanism: a system never forwards to an address
that already appears in `Seenby`.

The CRC field is CRC-32, transmitted as 8 uppercase hex digits with no
leading `0x`. Receivers that find a CRC mismatch reject the file and
do not forward it, which protects the echo from being poisoned by a
single corrupt link.

## Hatching

The act of creating a file echo entry is "hatching". The original
TIC carries a `Hatch` line and is constructed by the hatching tool
(`HATCH.EXE` in Barry Geller's original distribution, the `hatch`
command in AllFix and modern tossers). The hatcher's address goes in
`Origin`. From that moment the file is in the echo and will be
distributed by every node on the tree.

## Relationship to other formats

TIC is independent of the message-base format (JAM, Squish, MSG) and
the mailer protocol (FTS-0001, EMSI, BinkP). The TIC and its payload
ride inside whatever the mailer delivers: a BinkP `M_FILE` transfer,
a ZModem batch, an HST file-attach. The receiving mailer drops both
files into the inbound directory, and a separate "tick" processor
(TICK, AllFix, FileMgr, TickIT, sbbsecho's TIC handler) is invoked to
parse the TIC, import the payload, and emit new TICs to downlinks.

SAUCE-stamped artpacks were distributed this way through the 1990s and
early 2000s. ACiD's monthly acquisition packs, ICE / iMPURE / Mistigris
releases, and the SAUCE-bearing nodelist segments all moved across the
file-echo network as `.TIC` + payload pairs. The TIC carried the
`Desc`, the artpack ZIP carried the SAUCE-stamped art inside.

## References

- FSC-0028: TIC File Forwarding, original FidoNet Standards Committee
  document. Based on Barry Geller's TICK program.
- FSC-0087: Current Practices in File Forwarding. Documents the TIC
  format extensions actually in use, including the AllFix dialect.
- AllFix documentation, Harms Software Engineering, http://allfix.com.
- TickIT module, http://wiki.synchro.net/module:tickit. Reference
  implementation of a TIC importer for Synchronet BBS.
