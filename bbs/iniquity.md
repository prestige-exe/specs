# Iniquity BBS

Origin: Mike "Fiend" Fricker, with early releases in the mid 90s.
The product was sold to Michael Pike, who shipped versions through
the 1.0x line; the most widely circulated late build is 1.00 Alpha
26r4 from 1997. The 2.x line continued under different hands
after the source was leaked, with the Iniquity Development Team
maintaining community releases. Mark Lewis is associated with
later 2.x work on the codebase.

Iniquity arrived after the first wave of Telegard-family boards and
positioned itself as the visually richer alternative. By the late
90s it was the preferred BBS package for the warez and demoscene-
adjacent boards that cared about display quality and pipe-art
support. iCE colour and SAUCE awareness were on by default;
embedded ANSi animation and large-format login matrices were
expected.

## Lineage

Iniquity is not a Telegard descendant. It was written from
scratch, in Borland Pascal, by Fricker. It does borrow the
underground visual conventions established by Telegard and
Renegade: pipe-code colours, two-letter MCI substitution tokens,
lightbar menus, and the same general menu-driven user experience.
A caller arriving on an Iniquity board could navigate it without
relearning anything; the file formats underneath were Iniquity's
own.

## File layout

An Iniquity install is rooted at a single directory, conventionally
`C:\INIQUITY\` or `C:\IQ\`. Subdirectories separate code, data,
text, and per-node scratch:

| Path | Contents |
| --- | --- |
| `\IQ\` | Executables and configuration |
| `\IQ\DATA\` | Binary system files (users, messages, files, security) |
| `\IQ\MENUS\` | Menu definitions |
| `\IQ\TEXT\` | `.ANS` / `.ASC` display files (per-language subdirectories supported) |
| `\IQ\GFILES\` | G-files |
| `\IQ\NODEx\` | Per-node working directory including drop files |
| `\IQ\MSGS\` | Message base storage |
| `\IQ\FILES\` | File base catalogues |

The user database, message bases, and file bases are stored as
fixed-length binary records in Pascal-typed files. As with the
Telegard family, the canonical byte layouts are documented in the
source rather than as portable specifications; tools reading them
historically reused the record declarations directly.

## Menu files

Iniquity menus are binary files holding a header plus an array of
command records. Conceptually each command record carries the
same fields as the Telegard / Renegade family (description,
hotkey, ACS, command type, data, flags), but the binary layout
and the set of available command types are Iniquity-specific. The
in-program menu editor is the canonical authoring tool.

Iniquity adds a richer set of display-binding options per
menu: a header file, a footer file, an idle-prompt file, and a
choose-prompt file can each be associated with the menu, so the
visual frame around a menu screen can be authored as ANSi art
independently of the command list itself.

## Pipe codes

Iniquity uses the familiar `|nn` pipe-code colour syntax shared
across the underground BBS family:

| Range | Meaning |
| --- | --- |
| `|00` through `|15` | Foreground colour, 0 = black, 15 = bright white |
| `|16` through `|23` | Background colour, 16 = black, 23 = light grey |

iCE colour is enabled by default. Display files with the high bit
of the attribute byte set are interpreted as selecting a bright
background rather than blinking foreground.

## MCI codes

MCI substitution uses the same `|` prefix followed by a letter
pair convention as the Telegard family, with an extended alphabet
specific to Iniquity. Categories:

| Category | Examples |
| --- | --- |
| User fields | Name, alias, real name, location, level, flags, sex, age |
| System fields | BBS name, sysop name, node number, current date and time |
| Session fields | Time left, time used, baud, connect type |
| Statistics | Call count, post count, upload and download counts, ratios |
| Control | Centre, pause, clear, abort if non-ANSi, conditional show |

Iniquity introduced compact conditional MCI directives ("show
this run of text only if the caller has flag X" and "show this
text only if security level is above N") that let a single
display file render differently for different access tiers
without per-tier files.

## SAUCE and art support

Iniquity is SAUCE-aware (see [[../metadata/sauce-00.5]]). Display
files with a SAUCE record have the SAUCE block stripped from the
rendered output, and the metadata is exposed to the sysop tools.
This let sysops drop scene art into the text directory without
hand-editing each file.

The package also recognises BinaryText (`.BIN`) and XBin (`.XB`)
display files, the higher-colour-depth formats popular among
scene artists by the late 90s. Iniquity is one of the few period
BBS packages that displayed these natively without external
viewers.

## Drop files

Iniquity writes a configurable set of drop files to the node
working directory before launching a door. Native support
includes:

- `DOOR.SYS` (GAP format, the default). See [[door-sys]].
- `DORINFO1.DEF` (QuickBBS/RemoteAccess format). See [[dorinfo]].
- `CHAIN.TXT` (WWIV format).
- `PCBOARD.SYS` (PCBoard format).

Multi-node Iniquity boards keep each node's drop files in that
node's working directory so concurrent door sessions stay
isolated. The door definition specifies which drop files to write
and the command line passed to the door executable.

## Versions

| Version | Date | Notes |
| --- | --- | --- |
| 1.00 Alpha | 1995 onwards | Pre-release alphas under Fricker |
| 1.00 Alpha 26 | 1997 | Last widely circulated alpha under Michael Pike, multiple revisions through r4 / r5 |
| 1.01 | 1997 | Michael Pike's first numbered release |
| 2.x | post-leak | Community continuation under the Iniquity Development Team |

The "1.0 Alpha 26" series is what most surviving production
Iniquity boards ran; later builds were less widely adopted before
the scene migrated to other platforms.

## Cultural position

Iniquity is the visual high-water mark of the DOS underground BBS
era. Where Renegade was the workhorse and Oblivion/2 was the
experimental playground, Iniquity is what a board ran when the
sysop wanted display quality and was willing to give up the
ubiquity of Renegade's door ecosystem in exchange.

## References

- Iniquity BBS archives, software.bbsdocumentary.com/IBM/DOS/INIQUITY/.
- Iniquity 3 modern reimagining, github.com/iniquitybbs/iniquity
  (preserved module names for runtime MCI processing, pipe-codes,
  ctrl-codes, BinaryText, XBin, SAUCE).
- iq.throwbackbbs.com, surviving production Iniquity instance with
  documentation.
- See also [[telegard]], [[renegade]], [[vision-2]], [[oblivion-2]]
  for sibling underground BBS packages; [[../metadata/sauce-00.5]]
  for the SAUCE metadata Iniquity is aware of; [[door-sys]] /
  [[dorinfo]] for the drop files Iniquity emits.
