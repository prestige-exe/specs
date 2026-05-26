# specs

a collection of specifications for the systems that ran the
boards. ansi. sauce. zmodem. fidonet. hayes. ice colors. vga
text mode. the formats and protocols that made the scene
possible, written down the way they were actually used.


## scope

  cp437 and the high ascii range. sauce 00.5. ansi escape
  sequences as implemented, not as standardized. zmodem
  over 9600 half-duplex. fidonet 4d addressing. hayes at
  command set with the init strings that made a usr
  courier behave in 1994.

  not a history. a reference.


## why

  a 12,000-byte ansi scroller from 1994 will not render correctly
  in any tool shipped this decade. color, font, geometry,
  metadata: one of them goes every time. the files still
  exist. the specs that describe them are scattered, half
  archived, or written down only in the heads of people
  who logged off in 1998.


## what is here

[storage and metadata]

- [SAUCE 00.5](sauce-00.5.md). the 128-byte record at the tail of
  half the ANSI art ever made.
- [CP437](cp437.md). the IBM PC ROM character set. the codepage
  every other file in this repo silently assumes.
- [FILES.BBS](files-bbs.md). directory descriptions for a BBS file
  area.
- [FILE_ID.DIZ](file-id-diz.md). 10 lines by 45 columns of release
  description, inside the archive.
- [NFO](nfo.md). release notes, group greets, and a CP437 logo on
  top.

[text-mode art]

- [ANSI escape codes](ansi-escape.md). the BBS-relevant subset of
  ECMA-48, as implemented by ANSI.SYS.
- [AVATAR / AVT-0+](avatar.md). FidoNet's denser alternative to
  ANSI.
- [PCBoard @-codes](pcboard-at-codes.md). display-file macros for
  PCBoard.
- [BIN](bin.md). raw IBM PC text-mode video buffer, written to
  disk unmodified.
- [XBin](xbin.md). BIN with a header, optional font, optional
  palette, optional RLE.
- [ADF](adf.md). Artworx editor save format.
- [IDF](idf.md). iCEDraw editor save format.
- [TUNDRA](tundra.md). 24-bit per-cell colour on a character grid.
- [RIPscrip 1.53](ripscrip-1.53.md). TeleGrafix's vector graphics
  protocol.
- [RIPscrip 1.54](ripscrip-1.54.txt). and its
  [template](ripscrip-1.54-template.txt).

[transfer protocols]

- [XMODEM](xmodem.md). Ward Christensen, 1977. the ancestor.
  includes CRC and 1K extensions.
- [YMODEM](ymodem.md). Chuck Forsberg, 1985. batch transfers,
  file metadata, streaming.
- [ZMODEM](zmodem.md). Chuck Forsberg, 1986. full-duplex
  streaming with crash recovery.
- [Kermit](kermit.md). Frank da Cruz, Columbia. designed for the
  worst possible link. survives it.

[fidonet and message bases]

- [FidoNet FTS-0001](fidonet-fts-0001.md). the type-2 packet, the
  message format, the legacy session protocol.
- [FidoNet Echomail](fidonet-echomail.md). FTS-0004 and FTS-0005.
  the conference layer that propagated by nightly modem call.
- [EMSI](emsi.md). Electronic Mail Standard Identifier. one
  round-trip handshake instead of three.
- [JAM](jam.md). Joaquim-Andrew-Mats. the unifying message base.
- [Squish](squish.md). Lanius Corporation. the message base
  behind Maximus.
- [QWK](qwk.md). offline mail for everyone who could not afford
  to read online.

[bbs doors]

- [DOOR.SYS](door-sys.md). the universal 51-line ASCII drop file.
- [DORINFOx.DEF](dorinfo.md). RemoteAccess and QuickBBS drop
  file.
- [PCBOARD.SYS](pcboard-sys.md). PCBoard's fixed binary record.
- [CHAIN.TXT](chain-txt.md). WWIV's drop file.

[modem control]

- [Hayes AT command set](hayes-at.md). every dial-up terminal
  program issued these.

## provenance

where a document has a known author, the file names them. the
original author's prose is always preferable to any summary. this
collection cites them so the chain back to source stays intact.
the summaries here are scaffolding. the authoritative description
is the one a working renderer or tosser was built against.

## contribution

if a file is wrong, fix it. if a format is missing and you have
written a working decoder for it, add it. vague descriptions of
formats nobody has actually round-tripped are worse than nothing.

## license

the specifications themselves are works of their respective
authors. the original prose written for this collection is
released under the MIT License. see [LICENSE](LICENSE).
