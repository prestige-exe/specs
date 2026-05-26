# NFO – Information File

Origin: The Humble Guys (THG), 1990. The first published NFO is
widely cited as the one accompanying their release of *Knights of
Legend*. From there the convention spread across the entire
shareware-cracking and ANSI-art scene and persists into the present.

An NFO is the "release notes / liner notes" file shipped inside a
software release archive. Unlike `FILE_ID.DIZ` (which is a short
machine-readable description for BBS file listings), an NFO is a
human-readable document with credits, instructions, group greets,
contact information, and almost always a piece of ASCII / ANSI art
forming the group's logo at the top.

NFO files are not standardised – every group has their own template
– but enough conventions have hardened to call NFO a format in its
own right.

## File naming

```
<release-name>.NFO
```

Where `<release-name>` is the release's short name, often the
group's tag plus the product name. On case-insensitive systems
`README.NFO`, `INFO.NFO`, or simply `<groupname>.NFO` also occur.

Exactly one `.NFO` per release archive is the convention; multiple
NFOs is unusual and suggests a "double-release" (merged release).

## Encoding

- **CP437** ([[cp437]]) is the canonical character set – every
  shaded block, double-line box, and high-ASCII glyph used in NFO
  art assumes it. Reading an NFO in a Latin-1 or UTF-8 editor
  without a CP437 conversion yields mojibake on the art.
- **Width**: traditional NFO width is **80 columns**. Block-art
  groups occasionally use 132 columns for wider banners. ASCII-
  oriented groups (`-CLASS-`, `-iCE-`) sometimes used 79 or even
  90.
- **Line terminator**: CR LF (DOS / OS/2 / Windows convention).
  Releases from Amiga and Atari ST scenes occasionally use LF only.
- **No ANSI escape sequences**: an NFO is a static, monochrome
  document. Coloured "ANSI NFOs" exist but are typically named
  `.ANS` and shipped alongside the `.NFO`.

## Conventional structure

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                          [group ASCII logo]                                  │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   <release title>                                                            │
│   <release date>                                                             │
│   <release type: e.g. CRACK, KEYGEN, ISO, SHAREWARE>                         │
│   <release size, e.g. 12 x 5.00MB>                                           │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   RELEASE NOTES                                                              │
│   Multi-paragraph description of what was done                               │
│   and how to install.                                                        │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   GREETS                                                                     │
│   Acknowledgements to other groups, friends, lovers, enemies                 │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   STAFF                                                                      │
│   Code:   ...                                                                │
│   Art:    ...                                                                │
│   Couriers:  ...                                                             │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   CONTACT                                                                    │
│   email / IRC channel / FTP / BBS list                                       │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

The boxes are drawn with CP437 0xC4 / 0xCD / 0xB3 / 0xBA / etc.;
divider lines are runs of 0xC4 or 0xCD; the group logo at the top
typically uses 0xB0–0xB2 (light/medium/dark shades) and 0xDB–0xDF
(blocks and half-blocks).

## Practical guidance for NFO authors

- Keep the document under one screen if possible (24 lines for an
  80×24 viewer). For larger releases two screens is acceptable;
  more than three is a sign the content belongs in a separate
  `README.TXT`.
- The group logo and divider art should fit on an 80-column
  display *and* render correctly in monospace. NFOs that look fine
  in a proportional font but garbage in monospace are broken.
- Do not include binary payloads. An NFO with embedded base64 or
  shellcode is no longer an NFO.
- The release date is conventionally listed once at the top, in
  `YYYY-MM-DD` or `Mon-DD-YYYY` form. Avoid 2-digit years.

## Viewing NFO files

- **Windows**: Notepad with a "Terminal" or "FixedSys" font set to
  CP437 (`chcp 437` is the OEM page; Notepad does not honour it
  for opens, so most users install a dedicated viewer).
- **Cross-platform**: Damn NFO Viewer, Maelstrom NFOpad, iNFekt
  NFO Viewer, online viewers like defacto2.net.
- **Terminal**: `iconv -f cp437 -t utf-8 file.nfo` produces a
  UTF-8 file that displays correctly in any modern monospace
  terminal.

## SAUCE in NFOs

Several modern NFO viewers honour SAUCE records appended to NFO
files. DataType = 1 (Character), FileType = 0 (ASCII). The SAUCE
record can carry the group / author, the release date, the font
name, and the comment block – this is how art-scene NFOs make sure
they render with the correct fixed-width font even on systems whose
default is something other than IBM VGA.

See [[sauce-00.5]] for the SAUCE specification.

## Cultural notes

The NFO is the most enduring artefact of the warez / cracker /
demoscene cultures of the 1990s. Long after the BBS systems that
distributed the underlying software disappeared, the NFOs survived
in archives and continue to be produced by modern scene groups for
new releases. The ASCII / ANSI logos at the top of the file are an
art form unto themselves and the subject of [Defacto2](https://defacto2.net),
[16colo.rs](https://16colo.rs), and the [ANSILove](http://ansilove.org)
ecosystem.
