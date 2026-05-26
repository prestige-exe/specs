# Renegade BBS

Origin: Cott Lang, first public release June 1991. Written in Turbo
Pascal with assembly inserts, derived from the leaked Telegard 2.x
source tree, which itself traces back to WWIV 3.21 by way of Carl
Mueller's modifications. Lang stopped active development on
April 23, 1997. Maintenance continued through a succession of
caretakers (Miri Spence, Gary Hall, Jeff Herrings, T.J. McMillen,
Chris Hoppman, Lee Woodridge) and source has been on public Git
hosting since 2013.

Renegade became one of the dominant BBS packages in the
warez/H/P/A/V underground of the early-to-mid 90s. The look and
feel of a "scene Renegade board", with pipe colored prompts,
lightbar menus, and an ANSi login matrix, is part of the visual
vocabulary of the era.

## Lineage

```
WWIV 3.21 (Wayne Bell)
   |
Telegard (Carl Mueller, then Eric Oman, then Martin Pollard and
   |     Tim Strike)
   |
Renegade (Cott Lang, 1991)
```

Renegade inherited Telegard's user-record layout, its menu file
concept, and its pipe-code MCI syntax. By the time Renegade matured,
it had diverged enough that menu files and display files are not
binary-compatible with Telegard, but the conceptual model and most
of the pipe codes are shared.

## File layout

A Renegade installation is rooted at a single directory, by
convention `C:\RG\`. The runtime layout:

| Path | Contents |
| --- | --- |
| `\RG\` | Executables (`RG.EXE`, configurators) and `CONFIG.DAT` |
| `\RG\DATA\` | Binary system data (users, message conferences, file bases) |
| `\RG\MENUS\` | `.MNU` menu definition files |
| `\RG\TEXT\` | `.ANS`, `.ASC`, `.RIP` display files |
| `\RG\MSGS\` | Per-conference message data |
| `\RG\FILES\` | File-base catalogues and uploads |
| `\RG\GFILES\` | "G-files", bulletin text |
| `\RG\NODE1\`, `\NODE2\`, ... | Per-node scratch, including drop files |

The user database is stored in `USERS.DAT` (fixed-length Pascal
records) with an auxiliary `USERSXI.DAT` for extended fields added in
later versions. Message bases live in `MSGS.DAT` / `MSGHDR.DAT`
pairs per conference. File catalogues are `FILES.DAT`. None of
these are documented as portable formats; tools that need to read
them historically reused the Pascal record declarations from the
Renegade source.

## Menu files

Renegade menus are stored as `.MNU` files in `\RG\MENUS\`. Each
menu file describes one menu screen and consists of a header record
followed by one record per menu command. The records are stored as
Turbo Pascal typed-file records, not as text. The Renegade menu
editor (`MENUEDIT`, accessible from the sysop F-key menu) is the
canonical tool for editing them.

A single command record carries, conceptually, these fields:

| Field | Description |
| --- | --- |
| Long description | Text shown in the menu listing |
| Short description | One-word name for lightbar display |
| Hotkey | Character that triggers the command, or `//string` for typed commands |
| ACS string | Access Control String required to see and run the command |
| Command type | Two-letter code identifying the action |
| Command-line data | Arguments to the action (filename, message area number, door tag, etc.) |
| Flags | Bits for "hide from menu", "force pause", "clear screen", etc. |

The two-letter command type is the action: `OP` (open another
menu), `GA` (goodbye/logoff), `DD` (download), `OD` (open door),
`DG` (display a g-file), `MA` (change message area), `MS` (scan
messages), `MR` (read messages), `FB` (file base list), and many
others. The full command set is documented in the Renegade sysop
manual; over 200 distinct types exist across versions.

## Access Control String

The ACS is the conditional expression Renegade evaluates before
showing or running a command, displaying a prompt, or letting a
caller into a file or message area. The syntax is a compact stream
of tokens, each one letter followed by a value:

| Token | Meaning |
| --- | --- |
| `s<n>` | Security level at least `n` |
| `f<n>` | User flag `n` is set (A through Z map to flags 1 through 26) |
| `g<n>` | Group membership `n` |
| `a<n>` | Age at least `n` |
| `b<n>` | Baud rate at least `n` |
| `t<n>` | Time left at least `n` minutes |
| `u<n>` | User record number equals `n` |
| `c<n>` | Co-sysop flag `n` |
| `!` | Logical NOT applied to the next token |
| `|` | Logical OR between sub-expressions |
| `()` | Grouping |

An ACS of `s50f1g3` reads "security at least 50, AND user flag 1
set, AND in group 3". `!s100` means "security below 100". `s10|s90`
means "security 10 or above 90". An empty ACS evaluates true: the
command is accessible to everyone.

## MCI codes

MCI codes are control sequences embedded in display files
(`.ANS`, `.ASC`) and prompts. Renegade processes them when sending
the file to the caller. Two prefixes exist:

- `|` followed by two digits selects a colour or invokes a numbered
  macro. `|07` is light grey on black, `|15` is bright white,
  `|17` is blue background, and so on through the IBM CGA palette.
- `|` followed by a letter pair invokes a substitution. `|UN`
  inserts the caller's user name, `|BD` inserts birthdate, `|TL`
  inserts time left.

The substitution categories include user fields (name, alias, real
name, location, level, flags), system fields (BBS name, sysop name,
current time and date, current node number), session fields (time
left, time used, baud), and conditional / control directives
(centre next line, pause for keystroke, clear screen, abort if
non-ANSi).

Inherited from Telegard, extended significantly by Renegade. A
modest Renegade install recognises roughly 100 distinct two-letter
MCI tokens; later community builds added more.

## Drop files

Renegade writes one or more drop files to the node's working
directory (`\RG\NODEx\`) before launching an external door. The
drop file is regenerated for each door launch. The set is
configurable; the package ships with support for:

- `DOOR.SYS` (GAP format, 51 lines, the default and most widely
  supported). See [[door-sys]].
- `DORINFO1.DEF` (QuickBBS/RemoteAccess format). See [[dorinfo]].
- `CHAIN.TXT` (WWIV format, retained from the WWIV genetics).
- `PCBOARD.SYS` (PCBoard format, in later versions).

The door definition in the sysop menu specifies which drop files
to write and what command line to pass. Multi-node Renegade boards
keep each node's drop files in that node's working directory so
concurrent door sessions do not collide.

## ANSi look

Renegade was the host underneath a great deal of the underground
ANSi scene's released art. The product shipped no notable art of
its own, but the configurability of every prompt and every menu
screen, combined with the pipe-code MCI syntax that art editors
like TheDraw and ACiDDraw could emit natively, made it the path of
least resistance for sysops who wanted a custom-themed board.

## References

- Renegade BBS official documentation, www.rgbbs.info.
- Renegade source repositories on GitHub (hiraethbbs/renegade and
  community forks).
- Cott Lang, original release notes, June 1991.
- Telegard 2.x source notes (for shared MCI and ACS heritage).
- See also [[telegard]] for the parent lineage and [[door-sys]] /
  [[dorinfo]] for the drop file formats Renegade emits.
