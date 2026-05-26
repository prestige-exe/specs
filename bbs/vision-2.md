# ViSiON/2 BBS

Origin: Crimson Blade, written in Turbo Pascal circa 1991 through
1993. ViSiON/2 is the second generation of the ViSiON code, which
descended from a leaked LSD 1.21 source, which descended from
Forum. The lineage commonly cited is Forum → LSD → ViSiON →
ViSiON/2; it is a separate family from the WWIV → Telegard →
Renegade tree, even though the visual conventions overlap.

ViSiON/2 was tightly associated with the warez and ANSi art scenes
of its years. Its defining feature was extreme configurability:
nearly every display string the BBS emitted lived in an editable
string file rather than being compiled into the executable, which
let sysops re-skin the BBS to mimic any other software in
circulation. Boards running ViSiON/2 are visually unmistakable
once you have seen one, and equally easy to disguise as something
else.

## File layout

A ViSiON/2 install centres on a single root directory containing
the executable and configuration. Data subdirectories hold the
user file, message bases, file bases, prompt strings, and display
files. The precise filenames and binary layouts of the system
data files are not publicly documented in a portable form; the
canonical reference is the Pascal source itself, which has
circulated in scene archives since the early 90s and is preserved
in modern resurrection projects.

The functional groups are:

| Group | Contents |
| --- | --- |
| Executable and config | The main `.EXE`, system configuration record |
| User base | Per-user fixed-length records (name, level, flags, statistics) |
| Message bases | Per-conference message header and text storage |
| File bases | Per-area file listings, descriptions, upload metadata |
| Prompts and strings | Editable string file with every interactive prompt |
| Display files | `.ANS` and `.ASC` art shown at login, in menus, between areas |
| Door section | Per-door configuration and launch templates |

## String file and prompts

ViSiON/2's prompts live in an external string file rather than
inside the executable. The sysop edits this file through the
configuration program to change any text the BBS displays:
welcome banners, login prompts, error messages, status lines.
This is what made ViSiON/2 the "BBS that can look like any other
BBS". A skilled sysop could and did configure ViSiON/2 to be
visually indistinguishable from Telegard, Renegade, Iniquity, or
even Oblivion/2.

Every prompt accepts pipe-code colour and MCI substitution. The
syntax follows the same family convention as Telegard/Renegade:
`|` followed by two digits is a colour, `|` followed by a letter
pair is a substitution token for a user or system field.

## Pipe codes

Colour codes are the standard IBM-CGA-attribute set, in the
familiar 16-foreground by 8-background pattern:

| Range | Meaning |
| --- | --- |
| `|00` through `|15` | Foreground colour, 0 = black, 15 = bright white |
| `|16` through `|23` | Background colour, 16 = black background, 23 = light grey background |

Substitution tokens cover the same conceptual ground as Renegade's
MCI codes: user name, alias, level, flags, time left, location,
BBS name, sysop name, current date and time, node number, last
caller. The exact token alphabet was customised in many community
builds; documenting a portable subset is more accurate than
claiming a single canonical list.

## Display files

ViSiON/2 display files are standard ANSi or ASCII (CP437), with
SAUCE metadata commonly attached for archived art (see
[[sauce-00.5]]). The BBS scans the first byte of each display file
to detect ANSi escape sequences and falls back to a stripped ASCII
rendering for callers without ANSi enabled.

iCE color support, where the high bit of the attribute byte
selects a bright background instead of a blinking foreground, was
expected by the time ViSiON/2 matured. Sysops typically enabled
iCE color globally because the warez and ANSi scenes had
standardised on it.

## Drop files

ViSiON/2 emits a drop file before launching a door. The native
support set centres on `DOOR.SYS`, the GAP-format universal drop
file (see [[door-sys]]), because that is what scene doors expected.
Sysops also commonly configured DORINFO1.DEF emission for the
QuickBBS family of doors (see [[dorinfo]]). Some builds added
CHAIN.TXT for WWIV-derived doors.

The drop file is regenerated per door launch in the node's working
directory; multi-node ViSiON/2 boards keep node directories
separate so concurrent door sessions do not overwrite each other.

## Networking and message bases

ViSiON/2 supported the FidoNet-style message networks of its era
and the warez-scene private networks (BLiTZNet, FelonyNet, and
similar) that emulated FidoNet-style point-to-point distribution
over a curated node list. The native message base format is a
header-plus-text pair per conference; tossing software for the
warez networks was typically distributed as separate utilities
that read and wrote the ViSiON/2 base directly.

## Cultural position

ViSiON/2 sat alongside Renegade, Iniquity, and Oblivion/2 as one
of the four BBS packages that defined what an underground board
looked like in the mid 90s. The string editor was the
differentiator: where Renegade tied you to a specific visual
template even after heavy customisation, ViSiON/2 let you rewrite
everything down to the error messages, which made it the
preferred package for sysops who wanted a board that did not
visibly belong to any one family.

## References

- ViSiON-2 Resurrection project on GitHub (stlalpha/vision-2-bbs),
  containing preserved source and documentation.
- Defacto2 archive of ViSiON-X and ViSiON/2 artifacts.
- scene.org BBS resource archive, /resources/bbs/pc_warez/.
- See also [[telegard]], [[renegade]], [[iniquity]], [[oblivion-2]]
  for the sibling underground BBS packages, and [[door-sys]] for
  the drop file format ViSiON/2 emits to doors.
