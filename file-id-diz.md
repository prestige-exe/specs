# FILE_ID.DIZ – Description in ZIP

by Clark Development Co. (Fred Clark) / The Forbin Project (Richard
Holler), c. 1992. Originally a convention published in PCBoard
Magazine and quickly adopted across every major BBS package and
upload tester.

`FILE_ID.DIZ` is a small plain-text file embedded inside a software
release archive (`.ZIP`, `.LZH`, `.ARJ`, `.RAR`, ...) that describes
the release. When a user uploads the archive, the BBS extracts
`FILE_ID.DIZ` and uses its contents as the file description. Without
it, the BBS prompts the uploader to type a description by hand;
with it, the description is consistent across every BBS that hosts
the release.

## Rules

The original spec is short:

1. The file is named exactly `FILE_ID.DIZ` (case-insensitive on the
   filesystems it was designed for, always uppercase on the wire).
2. Plain ASCII / CP437. No formatting codes, no ANSI sequences, no
   `^G` (BEL), no high-bit characters above 0x7F that the BBS's
   description field cannot display.
3. Maximum **10 lines**.
4. Maximum **45 characters** per line.
5. Lines terminated with CR LF (or LF on Unix releases, tolerated by
   most readers).
6. No trailing whitespace; no embedded tabs.
7. The file lives in the *root* of the archive, not in a sub-folder.

The 45×10 grid is the width and height of PCBoard's "long
description" field. Other BBSes (Wildcat, Renegade, RemoteAccess,
Synchronet) display longer descriptions but truncate to 45×10 for
compatibility with FILE_ID.DIZ; modern BBSes accept the file
verbatim.

## Recommended structure

Convention (not part of the spec) has solidified around a three-block
layout:

```
+---------------------------------------------+
| Group / product name + version              |  Lines 1..2
+---------------------------------------------+
| One- to four-line description               |  Lines 3..6
+---------------------------------------------+
| Crack / release credits, requirements       |  Lines 7..10
+---------------------------------------------+
```

The first line typically opens with the artist/group's "tag" in CP437
block characters; many releases use the line as a four- or eight-
character wide block-art header.

## Example FILE_ID.DIZ

```
PrestigeWare File Manager v2.1 [PWFM]
-------------------------------------
A drop-in replacement for the BBS file
section, including SAUCE display, ZIP
comment extraction, and FidoNet ECHO
support.

(c) 1995 PrestigeWare. All rights
reserved. Requires DOS 5.0+ and 384K
free conventional memory.
```

(8 lines, longest line 42 characters – fits comfortably.)

## DESC.SDI – the predecessor

Before `FILE_ID.DIZ`, the convention was `DESC.SDI` ("Standard
Description Information"), used by the Aquila BBS package and a
handful of others. The format is identical, but the filename did
not catch on – `FILE_ID.DIZ` is the version every shareware author
and warez group ended up using.

## Reading FILE_ID.DIZ

A BBS upload tester:

1. Looks inside the uploaded archive for any file whose name matches
   `FILE_ID.DIZ` (case-insensitive) at the archive root.
2. If found, extracts up to 10 lines, truncating any line longer
   than 45 characters and any file longer than 10 lines.
3. Optionally strips any 0x1A (DOS EOF) and trailing whitespace.
4. Sets the file description in its file database to the extracted
   text.

If the file is missing or contains only whitespace, the BBS prompts
the uploader for a description.

## Wider context

`FILE_ID.DIZ` became the standard "shipping label" of the warez
scene. By the mid-1990s, any release that did not include a
properly formatted FILE_ID.DIZ would be re-packed by the next site
in the chain – not because of any technical requirement, but as a
quality-control signal. The combination of a FILE_ID.DIZ, a SAUCE-
sauced NFO ([[nfo]]), and a proper directory layout was the visible
mark of a "polished" release.

## SAUCE in FILE_ID.DIZ

A FILE_ID.DIZ may carry a SAUCE record. DataType = 1 (Character),
FileType = 0 (ASCII). TInfo1 = 45 (width), TInfo2 = lines used.
Practical adoption is rare – the file is so small that adding a
128-byte SAUCE more than doubles its size.

## Variants

- `BBS_ID.DIZ` – per-BBS variant carrying BBS-specific info, never
  caught on.
- `FILE_ID.ANS` – ANSI-formatted description, only readable by
  graphics-capable BBSes; some art groups shipped both.
- `WHATSNEW.TXT` – longer release notes, complementary to
  FILE_ID.DIZ rather than a replacement.
