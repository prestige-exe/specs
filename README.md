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

[metadata/]

- [SAUCE 00.5](metadata/sauce-00.5.md). the 128-byte record at the
  tail of half the ANSI art ever made.
- [CP437](metadata/cp437.md). the IBM PC ROM character set. the
  codepage every other file in this repo silently assumes.
- [FILES.BBS](metadata/files-bbs.md). directory descriptions for a
  BBS file area.
- [FILE_ID.DIZ](metadata/file-id-diz.md). 10 lines by 45 columns
  of release description, inside the archive.
- [NFO](metadata/nfo.md). release notes, group greets, and a
  CP437 logo on top.

[art/]

- [ANSI escape codes](art/ansi-escape.md). the BBS-relevant subset
  of ECMA-48, as implemented by ANSI.SYS.
- [AVATAR / AVT-0+](art/avatar.md). FidoNet's denser alternative
  to ANSI.
- [PCBoard @-codes](art/pcboard-at-codes.md). display-file macros
  for PCBoard.
- [BIN](art/bin.md). raw IBM PC text-mode video buffer, written to
  disk unmodified.
- [XBin](art/xbin.md). BIN with a header, optional font, optional
  palette, optional RLE.
- [ADF](art/adf.md). Artworx editor save format.
- [IDF](art/idf.md). iCEDraw editor save format.
- [TUNDRA](art/tundra.md). 24-bit per-cell colour on a character
  grid.
- [TheDraw Font](art/tdf.md). .TDF bitmap font collection used by
  TheDraw for headers and group greets.
- [RIPscrip 1.53](art/ripscrip-1.53.md). TeleGrafix's vector
  graphics protocol.
- [RIPscrip 1.54](art/ripscrip-1.54.txt). and its
  [template](art/ripscrip-1.54-template.txt).

[transfer/]

- [XMODEM](transfer/xmodem.md). Ward Christensen, 1977. the
  ancestor. includes CRC and 1K extensions.
- [YMODEM](transfer/ymodem.md). Chuck Forsberg, 1985. batch
  transfers, file metadata, streaming.
- [ZMODEM](transfer/zmodem.md). Chuck Forsberg, 1986. full-duplex
  streaming with crash recovery.
- [Kermit](transfer/kermit.md). Frank da Cruz, Columbia. designed
  for the worst possible link. survives it.
- [HSLink](transfer/hslink.md). Samuel H. Smith. bidirectional
  alternative to ZMODEM.
- [SEAlink](transfer/sealink.md). System Enhancement Associates.
  sliding-window XMODEM variant, the SEAdog default.
- [Janus](transfer/janus.md). Rick Huebner, Opus-CBCS.
  bidirectional protocol for Fido mailers.

[fidonet/]

- [FidoNet FTS-0001](fidonet/fidonet-fts-0001.md). the type-2
  packet, the message format, the legacy session protocol.
- [FidoNet Echomail](fidonet/fidonet-echomail.md). FTS-0004 and
  FTS-0005. the conference layer that propagated by nightly modem
  call.
- [EMSI](fidonet/emsi.md). Electronic Mail Standard Identifier.
  one round-trip handshake instead of three.
- [Nodelist](fidonet/nodelist.md). FTS-5000-series. the FidoNet
  address database in ASCII, with weekly NodeDiff updates.
- [BinkP](fidonet/binkp.md). FTS-1026. the TCP/IP mailer protocol
  that replaced modem polling.
- [Hydra](fidonet/hydra.md). Arjen Lentz. bidirectional file and
  mail transfer over a single session.
- [TIC](fidonet/tic.md). FSC-0028. the file-echo companion to
  echomail. how artpacks propagated.

[msgbase/]

- [JAM](msgbase/jam.md). Joaquim-Andrew-Mats. the unifying message
  base.
- [Squish](msgbase/squish.md). Lanius Corporation. the message
  base behind Maximus.
- [QWK](msgbase/qwk.md). offline mail for everyone who could not
  afford to read online.

[doors/]

- [DOOR.SYS](doors/door-sys.md). the universal 51-line ASCII drop
  file.
- [DORINFOx.DEF](doors/dorinfo.md). RemoteAccess and QuickBBS
  drop file.
- [PCBOARD.SYS](doors/pcboard-sys.md). PCBoard's fixed binary
  record.
- [CHAIN.TXT](doors/chain-txt.md). WWIV's drop file.

[modem/]

- [Hayes AT command set](modem/hayes-at.md). every dial-up
  terminal program issued these.

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
