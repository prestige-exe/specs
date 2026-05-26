# PCBoard @-Codes – Display Macros for PCBoard BBS

by Clark Development Co. (Fred Clark)
Documented in PCBoard BBS reference manuals from version 14.5 onward
(1989–1996). PCBoard was one of the dominant DOS BBS packages of the
period and its display-file macro syntax was widely cloned by other
software.

@-codes are a single-line, in-text macro language embedded inside
PCBoard display files (`.PCB`, `.BBS`, `.DSP`, menu text, etc.). They
let the sysop place dynamic values – the caller's name, the current
node, today's date, the number of file uploads – into otherwise static
text, and let them paint colour without writing ANSI escape sequences.

## Syntax

```
@<token>@
```

Every @-code starts and ends with the at sign and contains an
uppercase token. Tokens are case-insensitive in PCBoard 15+, though
all documentation uses uppercase. Two consecutive at signs (`@@`) emit
a single literal `@`.

Unknown tokens are emitted verbatim, including the surrounding `@`.

## Categories

@-codes fall into four loose categories:

1. **User / session macros** – substitute information about the caller
   or the current session into the output.
2. **Colour macros** – `@X<bg><fg>@`-style colour control independent
   of the terminal's escape sequence dialect.
3. **Pause / paging macros** – control how text is sent to the screen.
4. **Conditional / file macros** – include a sub-file, switch on a
   conditional, etc.

## Colour: `@X<BG><FG>`

The most commonly seen @-code. Sets the current text colour using a
two-hex-digit attribute:

```
@X07         normal grey on black
@X1F         bright white on blue
@X74         red on light grey
@XFC         flashing light red on white  (or bright bg in iCE colors)
```

The two hex digits are *background then foreground*, each 0..F using
the IBM PC text-mode palette index order (see [[bin]]). Background `8`
through `F` requires the terminal to be in iCE-colors mode; on a
standard terminal these select blink + foreground colour.

`@X` does not need a closing `@` – it is exactly four characters
including the leading `@`. The colour stays in effect until the next
`@X` or until reset by an embedded ANSI sequence.

Internally, PCBoard converts each `@X` into the equivalent ANSI SGR
sequence on transmit if the caller's terminal advertises ANSI; on a
plain TTY the colour codes are silently dropped.

## User and session macros

| Token | Substitutes |
| --- | --- |
| `@FIRST@` | Caller's first name |
| `@LASTNAME@` | Caller's last name |
| `@FULL@` | Caller's full name |
| `@USER@` | Caller's full name (alias) |
| `@HANDLE@` | Caller's alias / handle, if alias support enabled |
| `@CITY@` | Caller's city |
| `@STATE@` | Caller's state |
| `@PHONE@` | Caller's voice phone number |
| `@DATAPHONE@` | Caller's data phone |
| `@SEC@` | Caller's security level |
| `@USERID@` | Caller's user number |
| `@CONF@` | Current conference number |
| `@CONFNAME@` | Current conference name |
| `@BPS@` | Connection speed |
| `@NODE@` | Current node number |
| `@TOTNODE@` | Total number of nodes configured |
| `@TIMELEFT@` | Time left in this session, hh:mm |
| `@TIMEUSED@` | Time used so far, hh:mm |
| `@MINLEFT@` | Time left, minutes only |
| `@DATE@` | Current date |
| `@TIME@` | Current time |
| `@BIRTHDATE@` | Caller's birthdate |
| `@FILES@` | Total files downloaded |
| `@FBYTES@` | Total bytes downloaded |
| `@UPFILES@` | Files uploaded |
| `@UPBYTES@` | Bytes uploaded |
| `@DLBYTES@` | Bytes downloaded today |
| `@DAYBYTES@` | Daily download limit |
| `@CREDLEFT@` | Download credit remaining |
| `@MSGLEFT@` | Messages left in this session |
| `@MSGREAD@` | Messages read |
| `@NUMCALLS@` | Caller's number of calls to the system |
| `@LASTON@` | Caller's previous login date |
| `@LOGDATE@` | Date the user joined the system |
| `@PROTOCOL@` | Caller's preferred transfer protocol |
| `@SYSDATE@` | System date (alias for DATE) |
| `@SYSOPIN@` | Hours sysop is available |
| `@SYSOPOUT@` | Hours sysop is unavailable |
| `@RATIO@` | Download/upload ratio |
| `@FREESPACE@` | Free disk space on download drive |
| `@VER@` | PCBoard version string |
| `@WHO@` | List of users on other nodes |

User macros expand at display time, so the same `.PCB` file shows the
right name and stats for every caller.

## Pause / paging macros

| Token | Meaning |
| --- | --- |
| `@MORE@` | "More? [Y/n]" prompt; caller hits a key to continue |
| `@PAUSE@` | Pauses until any key |
| `@WAIT@` | Like `@PAUSE@` but with a "Press [Enter] to continue" prompt |
| `@CLS@` | Clear screen |
| `@POS:<n>@` | Move cursor to column n on the current line |
| `@POFF@` | Pause off – disable any subsequent `@MORE@` |
| `@PON@` | Pause on – re-enable pausing |

## File / conditional macros

| Token | Meaning |
| --- | --- |
| `@<filename>@` | Inline a file by name (uncommon, version-dependent) |
| `@INCLUDE:<file>@` | Include another display file in place |
| `@IF "<expr>"@ ... @ELSE@ ... @ENDIF@` | Conditional inclusion |
| `@OPTEXT@` | Optional text passed by the calling subsystem (used in headers) |
| `@QOFF@`/`@QON@` | Turn caller's typeahead buffer off / on |
| `@SCROLL@`/`@NOSCROLL@` | Allow / disallow scrolling |
| `@AUTOMORE@` | Enable automatic "More?" pagination |

Conditional tokens are most often used in welcome screens to show
different text to first-time callers, expired callers, sysops, etc.

## Caret colour codes (PCBoard 15+)

In addition to `@X`, PCBoard 15 accepts a "caret" colour syntax for
foreground only:

```
^A   bright blue text
^B   bright green
^C   bright cyan
...
```

This shorthand is supported alongside `@X` and the two can be mixed.

## Compatibility with other BBS packages

The @-code idea was widely cloned. The token sets differ:

- **Wildcat!** uses `|` codes (e.g. `|07` for colour).
- **Renegade** uses `|`-prefixed codes plus its own MCI codes.
- **Spitfire** uses `@`-codes but with a smaller token set.
- **Synchronet** implements the PCBoard `@-codes` for compatibility
  with imported display files and adds its own.

The PCBoard set documented here is the one most files in the wild
target, because `.PCB` was the de-facto display-file format passed
around in archives of "BBS bulletins" and welcome screens.

## SAUCE

A `.PCB` display file may carry SAUCE; DataType = 1 (Character),
FileType = 4 (PCBoard). TInfo1 / TInfo2 carry width and height
exactly as for ANSI files. The colour @-codes do not move the cursor
in the same predictable way as ANSI does, so the height field is
advisory rather than authoritative.

## A real example

```
@CLS@@X1F                       Welcome to The Pit, @FIRST@!
@X07
You are caller number @X0E@NUMCALLS@@X07.
You have @X0A@TIMELEFT@@X07 left in this session.

@X08[ @X0FF@X08 ]iles  [ @X0FM@X08 ]essages  [ @X0FB@X08 ]ulletins
@X07
@WAIT@
```

That produces a cleared screen, a bright-white-on-blue banner, the
caller's first name, their visit count, their session time, a
three-button menu in mixed colours, and a "Press Enter" pause – all
without writing a single ANSI escape sequence.
