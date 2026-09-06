<div align="center">

# 🛰️ netaudit

### What every program on this machine is talking to — one window, in real time

<p align="center">
  <a href="https://axle-lang.dev"><img alt="Powered by Axle" src="https://img.shields.io/badge/powered%20by-Axle-5B4BE1?style=for-the-badge&labelColor=1b1b2b"></a>
  <a href="https://github.com/Axle-lang/smalt"><img alt="Built on smalt" src="https://img.shields.io/badge/built%20on-smalt-1D6FB8?style=for-the-badge&labelColor=1b1b2b"></a>
</p>
<p align="center">
  <img alt="Platform: Windows" src="https://img.shields.io/badge/platform-Windows-1D6FB8?style=flat-square&labelColor=1b1b2b">
  <img alt="Dependencies: none" src="https://img.shields.io/badge/runtime%20dependencies-none-2E7D32?style=flat-square&labelColor=1b1b2b">
  <img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-555555?style=flat-square&labelColor=1b1b2b">
</p>

</div>

---

A network audit tool written entirely in Axle. One window, one tree: a row per
**program**, folding open onto the addresses it has been talking to, each one
carrying who owns it, where it is, how often it has come back, how regularly,
and what that adds up to.

```
▾ chrome.exe                     37 endpoints   12 live      2.1 MB/s   ●●○○○
    TCP  142.250.75.174:443      google.com                  FR  ●●●●●
         established             AS15169 Google LLC              ×214 visits
                                                                 every 30 s · clockwork
    TCP  104.18.32.7:443         api.stripe.com              US  ●●●●○
▸ Discord.exe                    12 endpoints    3 live    180 KB/s    ●●●●○
▸ svchost.exe  by name            4 endpoints    4 live                ●●●●●
```

Built on [**smalt**](https://github.com/Axle-lang/smalt) — the window, the
surface, the event queue, the clipped 2-D primitives, the two anti-aliased
faces and the BMP writer are Axle calling Win32 directly, so there is nothing
to ship beside the binary and nothing on the link line the source does not
already name.

---

## What it can and cannot see

There is **no URL**. Windows does not record the path of an HTTP request
anywhere a tool can read, and HTTPS would encrypt it even if it did. Getting
one needs a proxy in the middle or a packet capture, and this program is
neither. What it answers instead:

| Question | Where the answer comes from | Needs admin |
|---|---|---|
| which program, which protocol, which address and port | the per-process TCP/UDP tables (`iphlpapi`) | no |
| which **name** is behind the address | the machine's DNS cache, and reverse DNS | no |
| who **owns** the address, and where it is | `ip-api.com`, with Team Cymru's DNS whois behind it | no |
| how many **bytes** moved | a kernel ETW trace | yes |
| the short connections polling misses | the same trace | yes |

Three readings, three badges in the title bar — `POLL`, `DNS`, `ETW` — each lit
when it is running and each explaining itself when hovered. An empty byte column
must never be readable as "this program sent nothing" when it means "nothing
measured bytes".

---

## What it is actually for

A connection list tells you a machine is talking to two hundred addresses. That
is not an audit; it is a haystack. What separates one address from another is
not volume but **shape**:

- a browser tab opens a burst of connections and stops;
- an update service reconnects every few hours, raggedly, because its timer drifts;
- a beacon reconnects every thirty seconds, to the millisecond, forever.

So every endpoint carries a ring of the instants it came back, and three
readings taken from it — the median gap, the wander around it, and the
steadiness that falls out. The default ranking weighs repetition, that
steadiness, the address's own reputation, traffic and recency together, and the
row says its reading in words: `×214 visits · every 30 s · clockwork`.

The trust score is out of five and **always explainable**. Enter opens a card
that lists every signal that applied with the points it cost or earned. A score
nobody can interrogate is one that will be believed blindly or ignored
entirely, and both are worse than no score.

---

## It refreshes every second and still sits still

The machine underneath changes constantly; the window does not move under your
hand. Six rules, applied everywhere:

1. **An endpoint is a journal entry, not a snapshot.** A closed connection goes
   grey and stays for five minutes with its counts intact. A table showing only
   what is `ESTABLISHED` right now would flicker continuously and would answer
   the wrong question.
2. **Identity is never an index.** The selection is an endpoint key, the folds
   are group keys, the scroll is pixels. A rebuild cannot move either.
3. **The sort is damped, and freezes on hover.** A row only overtakes its
   neighbour by a margin, and while the pointer is over the tree nothing
   reorders at all.
4. **Figures are smoothed.** Rates carry three quarters of the previous reading.
5. **It repaints on change, not on a clock.** Idle, it sleeps.
6. **Enrichment lands quietly.** A row completes in place; it does not jump.

---

## Keys

| | |
|---|---|
| `↑` `↓` | move · `→` `←` open and fold · `8` fold everything |
| `Enter` | the full card for an endpoint |
| `O` | open the folder holding the program, binary selected |
| `S` | next ranking · `/` filter · `L` local traffic · `B` listening and UDP sockets |
| `Space` | pause · `R` re-read now · `C` clear the history |
| `F12` | write the window to `netaudit.bmp`, beside the binary |
| `F1` | the key list · `Esc` close, clear the filter, or quit |

The column headers rank by what they name, and the one the rows are ordered by
is underlined. A header covering two readings takes both: `REPETITION / RHYTHM`
ranks by repeat count, and again by steadiness. The scrollbar is draggable and
clicking its track jumps there. A program's triangle folds it; the rest of its
row selects it, so you can read a program's totals without closing what you
were looking at.

Hovering a program shows its full image path; hovering an endpoint shows
everything the two lines had to elide.

---

## Where the names and the networks come from

Two sources, neither of which needs a key or an account.

**`ip-api.com/batch`** — up to a hundred addresses per request: AS number and
name, country, city, operator, and the `proxy` / `hosting` / `mobile` flags the
score reads. One batch in flight, three seconds apart, exponential back-off on
failure, and every address asked about exactly once per session.

**Team Cymru's DNS whois** — `x.y.z.w.origin.asn.cymru.com` and
`AS<n>.asn.cymru.com`, both `TXT`, for the addresses the batch could not name.

The second one is over DNS rather than HTTP on purpose. `std::net`'s HTTP
client speaks HTTP/1.1 and **not TLS**, so an `https://` request from an Axle
program cannot succeed — a registry fallback over HTTPS would have been one
that never once answered, and would have looked exactly like an address nobody
could name. DNS needs no TLS, no key, and no rate limit worth the name.

For the same reason there is **no blocklist signal** in the score. Every
keyless blocklist service is HTTPS-only, and a trust signal that can never
fire is worse than an absent one: it reads as evidence of innocence.

## Building

smalt is a submodule, so the clone has to bring it:

```bash
git clone --recursive https://github.com/Axle-lang/netaudit
cd netaudit
axle build
./target/netaudit.exe
```

An existing clone that predates it: `git submodule update --init`.

The only prerequisite is the Axle compiler, **v0.12.1 or newer**. There is no
SDK to install, no DLL to copy beside the binary and no `[link]` section to
fill in: every OS library this program uses — `iphlpapi`, `dnsapi`,
`advapi32`, `shell32`, and `gdi32` through smalt — is named by the
`extern "C" from "…"` block that imports from it, so the link line learns of
each from the declaration that needed it.

Run it as administrator to light the `ETW` badge and add the byte columns.
Everything else works unelevated.

`--snap <ms>` waits that long, writes `netaudit.bmp`, and quits — a capture
for a report, or for a script, without anyone standing over the machine at
the right moment. `F12` does the same thing on demand.

**Two things this program hides by default**, both with their count on the
toggle that reveals them: traffic that never leaves the building (this
machine and this network), and sockets with no peer (listening and UDP).
Between them they are three quarters of the rows on a working machine, and
none of them is what "what is this machine sending" means.

---

## Layout

```
src/
  main.axle          the window, the loop, the tiers, the two lookups in flight
  theme fmt          the palette and the grid; the figures a library cannot format
  app input          what the reader is doing, and the keys that do it

  sys/raw            the four pointer views the OS imports need — all the `unsafe`
  sys/tiers          which readings are running, and why the others are not
  sys/win32/         the Windows port, and the only files that name an OS
    conn             GetExtendedTcp/UdpTable, v4 and v6, one row shape
    procs            NtQuerySystemInformation names + QueryFullProcessImageNameW paths
    dnscache         the resolver's cache, inverted to address → name
    rdns             the PTR record, for what nothing else could name
    shell            explorer.exe /select,"…"

  model/pool         the readings taken of interned text — nothing here is a `string`
  model/key          the one hash every identity is folded with
  model/addr         loopback / private / public, and how an address is keyed
  model/hosts        one row per address: name, network, place, flags
  model/endpoints    the journal, and the salience the tree ranks by
  model/rhythm       the arrival ring, and the period and steadiness from it
  model/groups       one row per program, keyed on the image path
  model/score        the trust reading, and the reasons behind it
  model/view         the flattened tree: which rows, in what order, at what height
  model/pulse        the four headline cards, sampled once a tick

  enrich/queue       who gets looked up, how often, and the back-off
  enrich/worker      the one blocking call, on its own thread
  enrich/scan        reading values out of a JSON response, byte by byte
  enrich/ipapi       the batch response, folded into rows
  enrich/cymru       the registry's answer over DNS, for what the batch could not name
  enrich/ripestat    the announcing network, when the registry has one

  ui/parts card      the pieces every surface is assembled from
  ui/chrome tree     the title bar and cards; the list itself
  ui/tooltip detail  the hover card and the full endpoint card

vendor/smalt         the library, as a submodule
```

**One directory names an operating system, and `axle.toml` says which.**
`[port.win32]` binds `when = { os = "windows" }` to `dirs = ["win32"]`, so
`use crate::sys::conn::Conns` resolves to `sys/win32/conn.axle` on Windows and
would resolve to a sibling directory's file on another target. Everything
above `sys/` — the model, the enrichment, the whole UI — names no OS at all,
which is what makes a second port five files and no edit anywhere else.
`axle ports` prints the table with a tick per seam.

---

## What it stands on

Everything below the audit is [smalt](https://github.com/Axle-lang/smalt): the
window, the surface, the event queue, the clipped 2-D primitives with their
rounded corners and anti-aliased text, the two baked faces, the byte pool, the
slot index, the formatter and the BMP writer. This program carries none of
them, and the four that matter most are worth naming:

- **The frame is a view, not an owner.** `Surface::frame()` hands back an
  address and a clip, never the colour buffer as an array — so the shape that
  double-frees is not writable. An `i32[]` field over a borrowed plane gives
  one allocation two owners, and the second release is a fault on exit, after
  everything has been drawn and flushed.
- **The loop sleeps in the OS.** `Events::wait` blocks until an event arrives
  or the next reading is due. Idle and paused, this program uses no measurable
  CPU at all — which a poll-and-sleep loop cannot say, whatever the sleep, and
  which a tool that measures the machine owes it.
- **Text is drawn from bytes.** `Scratch` writes a figure into a block and
  `BitmapFont::drawBytes` renders straight from it, so a repaint that draws a
  few hundred numbers allocates nothing at all.
- **Wide names go through a real UTF-8 decode.** `Mem::wideFrom` emits
  surrogate pairs and degrades a malformed sequence to U+FFFD, so a path with
  an em dash in it survives the round trip back to `ShellExecuteW`.

What stays here is what a library cannot know: an address in its canonical
text form, a rate whose empty case means "nothing measured bytes" rather than
"nothing was sent", the fold the identity indexes are keyed on, and the band a
reading's trace is drawn against.

---

## License

MIT — see [LICENSE](LICENSE).
