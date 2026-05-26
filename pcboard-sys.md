# PCBOARD.SYS – PCBoard Door Drop File

Origin: Clark Development Co., 1986. Documented in the PCBoard SDK
and in PCBSDK.DOC distributed with PCBoard 14.5 through 15.4.

`PCBOARD.SYS` is the drop file PCBoard writes for doors. Unlike
`DOOR.SYS` and `DORINFOx.DEF`, it is a **fixed-size binary record**,
not a line-based ASCII file. A door reads it with a single
record-sized read and casts it onto a struct.

Two record sizes exist:

- **128 bytes** for PCBoard 14.x (the original).
- **256 bytes** for PCBoard 15.x (extended).

The first 128 bytes are identical between the two versions; the
second 128 bytes of the 15.x format add fields for new PCBoard 15
features (long node lists, conference passwords, multi-language
support).

## 14.x record (128 bytes)

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 2 | DisplayOn | "-1" if display is on, "0" off, ASCII string |
| 2 | 2 | PrinterOn | Printer toggle, ASCII |
| 4 | 2 | PageBellOn | Page bell toggle, ASCII |
| 6 | 2 | AlarmOn | Alarm toggle, ASCII |
| 8 | 1 | SysopFlag | ' ' (space) caller is not sysop, 'N' next, 'X' is sysop, etc. |
| 9 | 1 | SysopNext | 'N' if sysop will see caller next |
| 10 | 1 | ErrorCorrecting | 'Y' if MNP/V.42 connection |
| 11 | 1 | GraphicsMode | 'G' graphics, 'N' none, 'A' ANSI 7-bit |
| 12 | 1 | NodeChat | Chat-with-other-node flag |
| 13 | 5 | OpenBaud | Connection rate as ASCII, e.g. " 2400" |
| 18 | 5 | ComBaud | Computer-to-modem rate, ASCII |
| 23 | 2 | ComPort | "1" or "2", ASCII |
| 25 | 5 | ChannelType | "MODEM", "LOCAL", "PACKET", etc. |
| 30 | 4 | EventDate | mmdd, ASCII |
| 34 | 5 | EventTime | hh:mm, ASCII |
| 39 | 1 | EventActive | 'Y' if event is enabled |
| 40 | 1 | SlideEvent | 'Y' if event slides (delayed) |
| 41 | 2 | MemMsgInd | Memorized message indicator |
| 43 | 2 | NodeNumber | ASCII node number |
| 45 | 5 | NodeChatRequest | – |
| 50 | 5 | UserRecordNumber | ASCII decimal |
| 55 | 25 | UserName | Caller's name, space-padded |
| 80 | 12 | Password | Caller's password (security risk) |
| 92 | 4 | TimeOn | hhmm of login |
| 96 | 4 | TimeUsed | hhmm of session use |
| 100 | 5 | Conference | Current conference number, ASCII |
| 105 | 5 | ConferencesJoined | Bitmap (ASCII hex) of joined conferences |
| 110 | 5 | ConferencesScanned | Bitmap of scanned conferences |
| 115 | 13 | Reserved | Reserved |

Fields are *not* C-style or Pascal strings. They are ASCII text
right-padded with spaces; numeric fields are likewise text. A door
that wants the user's name as a string trims trailing spaces. A
door that wants the baud rate parses the ASCII digits.

This odd "binary file of ASCII strings" design is the worst of both
worlds and is the main reason PCBoard doors are harder to write than
RemoteAccess doors. The benefit, when it was new, was that a CP/M-
or DOS-era door could "see" the file as ASCII when type'd to the
screen yet read it as a fixed-record file for random access.

## 15.x extended record (additional 128 bytes)

PCBoard 15 added fields 128..255 for new BBS features:

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 128 | 14 | CallerID | Caller's caller-ID string if captured |
| 142 | 2 | UseANSI | "01" if ANSI is forced |
| 144 | 5 | UserSecurity | Security level, ASCII |
| 149 | 5 | LangNum | Language number for multilingual support |
| 154 | 9 | LangName | Name of the selected language |
| 163 | 12 | LangExt | Language file extension |
| 175 | 5 | CallerRecNum | Caller record number |
| 180 | 5 | PageLength | Lines per screen, ASCII |
| 185 | 5 | UserExpDate | User expiration date, MMDDYY |
| 190 | 5 | UserExpSecLvl | Expired security level |
| 195 | 5 | LastConfRA | Last conference for read-all |
| 200 | 5 | ProtocolFlag | One-letter protocol code |
| 205 | 8 | LogonDate | Date this session began, MMDDYY |
| 213 | 8 | LogonTime | Time this session began, HHMMSS |
| 221 | 35 | Reserved | Reserved |

15.x doors should check whether the file is 128 or 256 bytes before
indexing past offset 128; reading offsets 128..255 from a 14.x drop
file returns garbage.

## Identification

A door can distinguish PCBoard from other BBSes by:

- The presence of `PCBOARD.SYS` (vs `DOOR.SYS` etc.).
- The file size: 128 = PCBoard 14, 256 = PCBoard 15, anything else
  is a different package.
- The string "PCB" sometimes appears in the first few bytes of the
  user record area on 15.x.

PCBoard also conventionally sets the `PCB` environment variable to
the path of its data directory; a door can look there for
`PCBOARD.SYS` if no explicit path is passed.

## When PCBOARD.SYS matters

- A door distributed for PCBoard BBSes (most of the 1990–1996
  shareware doors with a `.PCB` distribution) reads `PCBOARD.SYS`
  natively.
- A "universal" door (TradeWars 2002, Legend of the Red Dragon)
  reads `DOOR.SYS` and ignores `PCBOARD.SYS`; the PCBoard sysop
  writes a one-line batch file to convert.
- PCBoard 15.21 and later can write a `DOOR.SYS` alongside
  `PCBOARD.SYS` for compatibility.

## Concurrency and shelling out

When a door is launched, PCBoard writes `PCBOARD.SYS` to the
caller's node directory (configurable in PCBSetup) and removes any
old file from the previous door call. The door reads the file
*once* on startup; PCBoard does not update it during the door
session. Time-remaining tracking inside the door is therefore the
door's responsibility – consult `TimeOn` + the system clock to
compute the user's remaining time.
