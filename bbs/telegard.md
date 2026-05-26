# Telegard BBS

Origin: Carl Mueller, first release 1988. Telegard began as a
modified copy of the WWIV 3.21 public-domain source by Wayne Bell,
re-branded under Mueller's name and seeded with sysop backdoors.
After Mueller was arrested for phone phreaking, the source was
handed to Eric Oman, who removed the backdoors and continued
development. Martin Pollard joined as co-developer around version
1.8 and was doing nearly all the work by 2.4. Tim Strike took over
the codebase from Pollard in the mid 90s and shipped the OS/2
variant Telegard/2 alongside the DOS line.

Telegard is the direct ancestor of Renegade, and through Renegade
indirectly of much of the underground BBS family. Its pipe-code
syntax, its menu file model, and its overall visual conventions
are what callers recognised in a Renegade or Iniquity board a
decade later.

## Lineage and the leak

```
WWIV 3.21 (Wayne Bell, public domain)
   |
Telegard 1.x (Carl Mueller, with backdoors)
   |
Telegard 2.x (Eric Oman, then Pollard)
   |---> source leaked, became Renegade (Cott Lang, 1991)
   |
Telegard 3.x (Pollard, then Tim Strike)
Telegard/2 (Tim Strike, OS/2 port)
```

The Telegard 2.x source leak is a load-bearing event in BBS
history. The leaked tree gave Cott Lang the foundation for
Renegade, which in turn seeded the underground BBS scene of the
following decade. The exact mechanism and version of the leak
remain disputed in scene oral history; treating it as having
happened in 1991 with a development snapshot, rather than a
clean release, matches the surviving evidence.

## File layout

A Telegard install lives in a single root directory, conventionally
`C:\TG\`, with subdirectories for data, menus, text, and per-node
working files. The functional groups:

| Path | Contents |
| --- | --- |
| `\TG\` | Executables (`TG.EXE`, configurators) |
| `\TG\DATA\` | Binary system files |
| `\TG\MENUS\` | `.MNU` menu definition files |
| `\TG\TEXT\` | `.ANS` / `.ASC` display files |
| `\TG\GFILES\` | G-files (bulletins, text databases) |
| `\TG\NODEx\` | Per-node working directory including drop files |
| `\TG\MSGS\` | Message base storage |
| `\TG\FILES\` | File base catalogues |

## User file

The user database is `USERS.LST`, a fixed-length record file
written from a Turbo Pascal record declaration. Each record holds
the caller's name, real name, password, location, security level,
flag byte(s), statistics (calls, uploads, downloads, posts), last
call timestamp, and configuration preferences. The exact byte
layout varies between Telegard 2.x and 3.x and is documented in
the Telegard source distribution rather than as a portable public
specification.

A separate user-index file accelerates lookup by name; a deleted
or inactive record is marked in the flag byte rather than removed,
so user numbers remain stable across the system's lifetime.

## Menu files (.MNU)

Telegard menus are stored as `.MNU` files in `\TG\MENUS\`. Each
file describes one menu and consists of a header followed by an
array of command records. The records are Turbo Pascal typed-file
records; the editor (`TGMENU` and later in-program menu editors)
is the canonical tool for editing them.

A command record carries:

| Field | Description |
| --- | --- |
| Description | Text shown when the menu is listed |
| Hotkey | The key that triggers the command |
| ACS | Access requirement to see and use the command |
| Command type | Mnemonic identifying the action (open menu, run door, post message, ...) |
| Data | Arguments for the action |
| Flags | Display and behaviour bits |

The command-type mnemonics, the ACS syntax, and most of the
behaviour-bit semantics were inherited wholesale by Renegade. See
[[renegade]] for the same conceptual model with annotated tokens;
Telegard's set is a strict subset of what Renegade later supports.

## Pipe codes and MCI

Telegard 2.x introduced the `|nn` pipe-code colour syntax that
became the default underground BBS art convention. The colour
mapping follows the IBM CGA attribute palette:

| Range | Meaning |
| --- | --- |
| `|00` through `|15` | Foreground colour, 0 = black, 15 = bright white |
| `|16` through `|23` | Background colour, 16 = black, 23 = light grey |

MCI substitution codes use the `|` prefix followed by a letter
pair. Categories include user fields (name, level, flags, location),
system fields (BBS name, sysop name, node, time, date), session
fields (time left, baud rate), and control directives (centre,
pause, clear screen). The exact alphabet of two-letter tokens
grew across Telegard versions; the 2.x set is the lowest common
denominator and is what Renegade inherited.

## TG.DAT drop file

Before launching an external door, Telegard writes `TG.DAT` to the
node's working directory. `TG.DAT` is Telegard's native drop file:
plain ASCII, line-based, conceptually similar to DOOR.SYS but with
a different field order and a different field set, exposing fewer
fields than DOOR.SYS but enough for a typical door to identify the
caller and the session state.

A door that wants to run under Telegard either reads `TG.DAT`
directly or uses one of the other drop files Telegard can also
emit:

- `DOOR.SYS` (GAP format). The most common compatibility target.
  See [[door-sys]].
- `DORINFO1.DEF` (QuickBBS/RemoteAccess format). See [[dorinfo]].
- `CHAIN.TXT` (WWIV format, retained from Telegard's WWIV ancestry).
- `CALLINFO.BBS` (Wildcat format, for older Wildcat-aware doors).
  See [[wildcat]].

The sysop selects which drop files to write per-door in the door
configuration. Most production Telegard boards wrote DOOR.SYS by
default for maximum compatibility and skipped TG.DAT, which means
TG.DAT is rare in surviving door archives even though it is
Telegard's native format.

## Message base

Telegard stores messages in a header file plus a text file per
conference, similar to the QuickBBS / Hudson layout. It supports
networking through external tossers (FidoNet via tossers like
FastEcho or TosScan, or the warez-scene private networks of the
era). Later versions added JAM message base support (see
[[../msgbase/jam]]) for compatibility with the broader BBS world.

## Visual conventions

The Telegard 2.x and 3.x ANSi look (a centred ANSi header, a
prompt line with colour-coded user statistics, lightbar menus
where supported) became the template for what an underground BBS
looked like for the rest of the decade. Renegade adopted it
wholesale; ViSiON/2 and Iniquity adopted enough of it that callers
could navigate without re-learning anything.

## References

- Telegard source distributions archived at Internet Archive
  (archive.org/details/telegard) and on GitHub (icculus/telegard).
- BBS Software Directory Telegard entry,
  software.bbsdocumentary.com/IBM/DOS/TELEGARD/.
- Talk:Telegard discussion on Wikipedia (historical authorship
  notes from Pollard and others).
- See also [[renegade]] for the direct descendant, and [[door-sys]]
  / [[dorinfo]] for the standard drop files Telegard can emit.
