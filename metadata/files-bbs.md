# FILES.BBS – BBS File-Area Directory Listing

Convention rather than a single owner; in widespread use from c. 1985
onward. The format originates with the SEAdog mailer and was adopted
by every major BBS package (PCBoard, Wildcat!, RemoteAccess, Maximus,
WWIV, ...) as the common interchange representation of a file-area's
contents.

A `FILES.BBS` is a plain text file that lives in each file-area
directory. It lists every downloadable file in the directory, along
with its size, upload date, and a short description. BBSes display it
to users browsing the file area; sysops edit it by hand or rebuild it
from the BBS database.

## Format

Plain ASCII (CP437), one logical record per file. CR LF line endings.
Multi-line descriptions are supported via continuation lines.

Each record begins on a line whose first non-space character is the
filename:

```
FILENAME.EXT [size] [date] Description text starts here
                                continuation line 1
                                continuation line 2
```

| Column range | Field | Description |
| --- | --- | --- |
| 1–12 | Filename | DOS 8.3 name, left-justified, padded with spaces |
| 13–22 | Size | Decimal size in bytes, right-justified, space-padded, optional |
| 24–31 | Date | MM-DD-YY of upload, optional |
| 33–end | Description | Free text, max line width 79 or 80 |

Continuation lines start with one or more spaces in column 1 (no
filename); the description text begins indented to roughly column 33
to align with the first line.

## Common variants

The "column 1 is filename" rule is universal; the rest is more
flexible:

- **PCBoard style** uses fixed columns 1–12 for filename and
  13–22 for size. Date in columns 24–31. Description from 33.
- **Wildcat! style** uses tab separators instead of fixed columns.
- **Maximus style** allows the file size and date to be absent
  (they're recomputed from the file on disk).
- **Universal / minimum** style: filename, then whitespace, then
  description; no size, no date.

A tool reading FILES.BBS in the wild needs to handle all four.

## Example

```
README.TXT      1024 04-15-95 Release notes for v2.1
                                Includes installation instructions
                                and a list of known bugs.
SETUP.EXE     105678 04-15-95 Install program
PWFM21A.ZIP  1245789 04-15-95 PrestigeWare File Manager v2.1
                                Drop-in BBS file-area replacement
                                with SAUCE support.
```

Note the indentation alignment: continuation lines indent to column 33
to line up under the description column of the line above.

## Special markers

Some BBSes add leading sigils to the filename column to flag
particular states:

| Marker | Meaning |
| --- | --- |
| `*FILENAME.EXT` | New file (uploaded today) |
| `-FILENAME.EXT` | Failed integrity check |
| `+FILENAME.EXT` | Recently downloaded |
| `=FILENAME.EXT` | Offline / not on disk |

These markers are removed before the filename is parsed; readers that
do not understand them simply treat them as part of the name (and
fail to find the file).

## Comment / blank lines

Lines whose first character is `;`, `#`, or whitespace through to
end-of-line are conventionally treated as comments and not displayed
to the user. A completely blank line ends the current file's
description block – the next non-blank line must start a new file.

Some BBSes use lines beginning with `--` or `===` as visual separators
between sub-groups of files.

## File size in lines vs bytes

Older PCBoard `FILES.BBS` listings used **kilobytes** in the size
column ("104K" instead of "105678"). Modern listings universally use
bytes. A robust reader checks for a trailing 'K' or 'k' and converts
if present.

## Date format

`MM-DD-YY` is the only widespread format. Y2K rules vary; the
conventional split was "00–79 = 2000–2079, 80–99 = 1980–1999". Some
late-1990s BBSes switched to `MM-DD-YYYY`, breaking column alignment
for any reader that assumed 8 characters in the date field.

## Multi-area FILES.BBS

A sysop with many file areas often maintains a single master
`ALLFILES.LST` (or `ALLFILES.ZIP`) that concatenates every area's
`FILES.BBS` with separator lines naming each area. There is no
standard for this; the format is whatever the sysop's batch file
produces.

## When to use FILES.BBS

- For BBS file-area distribution: every modern BBS package reads
  FILES.BBS at startup or on demand to build its file database.
- For shareware authors: shipping a one-line `FILES.BBS` inside the
  release archive lets the BBS auto-populate the description without
  needing `FILE_ID.DIZ` extraction.
- For interchange between BBSes: FILES.BBS is the lingua franca for
  moving file lists between PCBoard and Maximus, between Wildcat
  and Renegade, etc.

For inside-archive descriptions, prefer `FILE_ID.DIZ`
([[file-id-diz]]); FILES.BBS is for the *directory*, not for
individual archives.
