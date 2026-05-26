# Oblivion/2 BBS

Origin: Steve "Mercyful Fate" Berry. Oblivion/2 was the successor
to an earlier package called Illusion that Berry had worked on for
several years. The DOS Oblivion/2 line shipped through the mid and
late 90s; the most widely circulated release is in the 2.3x and
2.4x range from 1999, after which Berry made the system freeware.
A modern C++ rewrite under the same author lineage, Oblivion/2 XRM
(Extreme Remake) by M-griffin, preserves the original visual
language and configuration model on Linux, Windows, and macOS.

Oblivion/2 sat in the warez-scene BBS cluster alongside ViSiON/2,
Iniquity, and Renegade. It was distinguished from those by its
scripting model: rather than configuring behaviour through a fixed
menu editor with fixed command types, Oblivion/2 exposed an
embedded scripting language that let the sysop write the menu
logic itself.

## Lineage

Oblivion/2 is an independent codebase, not a Telegard descendant.
Berry wrote it from scratch in Borland Pascal. Like Iniquity, it
borrows the underground visual conventions (pipe-code colour,
two-letter MCI tokens, lightbar navigation) without sharing file
formats with the Telegard family.

## File layout

An Oblivion/2 install centres on a single root directory holding
the executable and configuration. Functional subdirectories:

| Path | Contents |
| --- | --- |
| `\OBV2\` | Executables and configuration |
| `\OBV2\DATA\` | Binary system files (users, messages, files, security) |
| `\OBV2\MENUS\` | Menu definitions and script files |
| `\OBV2\TEXT\` | `.ANS` / `.ASC` display files |
| `\OBV2\GFILES\` | G-files |
| `\OBV2\NODEx\` | Per-node working directory including drop files |
| `\OBV2\MSGS\` | Message base storage |
| `\OBV2\FILES\` | File base catalogues |

User records, message bases, and file bases are fixed-length
Pascal-typed files. As with the rest of the underground BBS
family, the byte layouts are documented in the source rather than
as portable specifications, and tools reading them have
historically reused the record declarations directly.

## MPL: Multi-Purpose Language

The defining feature of Oblivion/2 is MPL, the Multi-Purpose
Language. MPL is a small interpreted scripting language compiled
into bytecode by an external compiler (`MPLC.EXE`) and executed by
the BBS runtime when a menu command invokes a script.

Conceptually MPL is a Pascal-flavoured procedural language with:

- Variable declarations of typed local variables (string, integer,
  byte, boolean).
- Standard control flow (if/then/else, while, for, case).
- A library of built-in procedures for I/O against the caller
  (print a string, read a key, read a line, display a file with
  pipe-code processing), for user record access (read fields,
  modify fields, write back), and for system state (current node,
  time left, security level).
- A call mechanism to invoke other compiled MPL scripts.

A menu in Oblivion/2 can attach an MPL script to any command in
place of, or in addition to, the built-in action. This is how
sysops added features that other BBS packages required source
modifications for: custom file ratio rules, voting booths,
top-callers lists with custom layouts, paged sysop chat with
custom messages.

The compiler emits a `.MPX` bytecode file from a `.MPS` source
file. The BBS at runtime locates the `.MPX` by name and executes
it inside the current session context.

## Menu files

Oblivion/2 menus are binary files holding a header plus an array
of command records. Per command, the record carries the
underground BBS family fields (description, hotkey, ACS, command
type, data, flags) plus an optional MPL script name. When the
caller presses the hotkey, the BBS runs the built-in command type
if present and then the MPL script if attached, with the script
receiving control over the session.

## Pipe codes and MCI

Oblivion/2 uses the standard underground pipe-code colour syntax:

| Range | Meaning |
| --- | --- |
| `|00` through `|15` | Foreground colour, 0 = black, 15 = bright white |
| `|16` through `|23` | Background colour, 16 = black, 23 = light grey |

MCI substitution uses the same `|` prefix followed by a letter
pair convention. The token alphabet covers user fields, system
fields, session fields, statistics, and conditional control,
analogous to Iniquity's and Renegade's sets but with Oblivion/2's
own two-letter mnemonics.

iCE colour is enabled by default. Display files with SAUCE
metadata (see [[../metadata/sauce-00.5]]) have the SAUCE block
recognised and stripped from rendering.

## Drop files

Oblivion/2 writes a configurable set of drop files to the node
working directory before launching a door. Native support
includes:

- `DOOR.SYS` (GAP format, the most common). See [[door-sys]].
- `DORINFO1.DEF` (QuickBBS/RemoteAccess format). See [[dorinfo]].
- `CHAIN.TXT` (WWIV format).

Each node has its own working directory so concurrent doors do
not collide on drop files. The door definition specifies which
drop files to write and the launch command line, and may also
specify an MPL script to run before and after the door for setup
and cleanup.

## Versions

| Version | Date | Notes |
| --- | --- | --- |
| 2.0 | mid 90s | First widely circulated DOS release |
| 2.3x | 1998-1999 | Mature DOS line, MPL stabilised |
| 2.4x Beta | 1999 | Latest DOS development line, never finalised |
| Freeware | post-1999 | Berry released the existing binaries publicly |
| OBV/2 XRM | 2010s onwards | M-griffin's C++ remake for modern operating systems |

The "OBV/2 XRM" project preserves the menu model, the MPL
scripting concept, and the visual conventions in a contemporary
codebase that runs on Linux, Windows, and macOS over telnet.

## Cultural position

Oblivion/2 is the underground BBS package for the sysop who
wanted to write code. Renegade and Iniquity gave you menus and
configuration. Oblivion/2 gave you a scripting language and
expected you to use it. As a result, surviving Oblivion/2 boards
tend to be unusually heavily customised, with sysop-authored
features that did not exist on any other board.

## References

- Oblivion/2 XRM project, github.com/M-griffin/Oblivion2-XRM,
  and oblivion2-xrm.readthedocs.io.
- Oblivion/2 source discussion thread,
  gopherproxy.meulie.net/realitycheckbbs.org (Re: Oblivion/2
  Source code).
- BBS Software Directory entries for Illusion and Oblivion/2.
- See also [[telegard]], [[renegade]], [[vision-2]], [[iniquity]]
  for sibling underground BBS packages, and [[door-sys]] /
  [[dorinfo]] for the drop files Oblivion/2 emits.
