# ntp-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

An SNTP client (RFC 4330) over the NTPv4 packet (RFC 5905 § 7.3),
written in novo-lang: the forty-eight bytes as a `@value`, the two
different fixed-point scales the packet mixes, the offset and
round-trip arithmetic, the 2036 era rule, the kiss-o'-death codes a
pool server answers, and a pool policy that is a value rather than a
loop somebody hid.

The codec and the arithmetic build for a microcontroller, and
`tests/embedded_probe.nv` is that claim as a program rather than a
sentence.

It is SNTP, not NTP.  There is no clock discipline loop, no peer
filter, no clock selection algorithm and no server — the section at the
bottom says why for each.

## Adding it, and checking it

```bash
novo pkg add ntp-nv            # into your novo.toml
novo pkg build                 # type- and effect-check the package
novo test --isolate tests/ntpwire_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: ntp-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use ntpcalc
use ntpclient
use ntppkt

// Ask one server what the time is, and answer how far this machine's
// clock is from it — in microseconds, with the worst case it could be
// wrong by.
//
// `nonce` is the caller's and it MUST be unpredictable: it is the only
// thing in SNTP an off-path attacker cannot forge.  See "The
// load-bearing interface".
fn how_far_out(server: Str, nonce: NtpTimestamp) -> Result<Int, NtpFault> [net, time]
    let answer = ntpclient.query(server, nonce, 3000)!
    Ok(answer.offset_micros)
```

## The layer, and why

`host`, and four of the five modules declare nothing.

| module | row | why |
| --- | --- | --- |
| `ntpfault` | `[]` throughout | the faults and the kiss codes |
| `ntppkt` | `[]` throughout | forty-eight bytes in, forty-eight out |
| `ntpcalc` | `[]` throughout | integer arithmetic over four timestamps |
| `ntppool` | `[]` throughout | a policy is a value |
| `ntpclient.query`, `.query_pool`, `.query_best`, `.exchange_with` | `[net, time]` | the datagram, and the two clock reads an exchange needs |
| `ntpclient.now_timestamp`, `.clock_precision_micros` | `[time]` | the only two functions in the package that read a clock |
| `ntpclient.reading_of`, `.correction_micros` | `[]` | arithmetic, published beside the calls that produce their input |

**`[time]` is called exactly twice per exchange**, and that is not an
optimisation: the offset is the difference between the client's two
readings and the server's two, so a client that read its clock once has
thrown away the round trip it was measuring.

**`[rand]` is NOT in this row, and it is the decision the next section
is about.**

`layer = "host"` and **not** `layer = "core"` with `host_modules`,
because the plan's row is `host` and the subject is a client that asks a
server for the time.  What the four `[]` modules would move to is
argued below.

## The load-bearing interface

**The nonce is an argument**, and everything else about this package
follows from that.

SNTP has no cryptography in it at all.  A reply is forty-eight bytes of
UDP from an address anybody can spoof, and the only thing in it that an
off-path attacker cannot know is the **transmit timestamp the client
sent**, which the server copies back into the reply's originate field.
RFC 4330 § 3 says to make that field unpredictable; comparing the two is
the whole of SNTP's authenticity story.

So:

- **`ntppkt.request(nonce)` takes it**, rather than reading a clock.
  A `core`-shaped module has no randomness, and this is the shape that
  keeps the package's row at `[net, time]` instead of `[net, rand,
  time]`.  The README's "Where a row wanted to widen" section is about
  exactly this trade.
- **`ntpcalc.check(sent, reply)` takes the nonce back**, and it is a
  *separate function* from `ntppkt.packet_fault`, which makes the other
  five checks.  Two functions rather than one, so that a client calling
  only the packet-shaped one can be seen to have skipped the check that
  matters.
- **A caller that passes a counter, or the current time, has an SNTP
  client with no anti-spoofing.**  That is the cost of the argument
  being the caller's, it is said here, in the module comment and in the
  parameter's name, and it is the one thing to get right when
  implementing against this interface.
- **And a whole exchange becomes reproducible.**  The nonce, the four
  timestamps and the client's own precision are all arguments, so
  `tests/ntpwire_tests.nv` drives a complete exchange with no socket, no
  clock and the same answer every run.

The second decision is that **an exchange answers an offset and not a
time**.  `NtpReading` has no "the time is" field, because what to do
about the offset is a policy a library cannot take: a program that steps
its clock backwards breaks every monotonic assumption above it, one that
slews takes hours to converge, and one on a device with no clock at all
just wants the number to add to its counter.  Three callers, three right
answers, and a library that answered "the time is X" chose for all
three.

## What sensorhub would call

`orbit/sensorhub` logs a temperature reading every two seconds to a
block device that survives a reset, and its log has a cycle counter
where a timestamp should be.  A cycle counter is fine until the board is
power-cycled, at which point every reading after the reset claims to be
cycle 1 again and the log cannot be ordered against the one before it.

What closes that is three calls and about forty bytes of state:

| when | what it calls |
| --- | --- |
| once, after the radio is up | `ntppkt.request(nonce)` and the board's own datagram send |
| when the reply arrives | `ntpcalc.check(sent, reply)`, then `ntpcalc.exchange` and `ntpcalc.offset_micros` |
| on every log line after that | the board's counter plus the offset — `ntpcalc.unix_seconds_of` is not even needed on the device |

The device never calls `ntpclient`: it has no `std.net`, and one
host-only function anywhere in a compilation unit is an undefined symbol
at embedded link time whether or not the firmware calls it.  It calls
`ntppkt` and `ntpcalc`, which is exactly what `tests/embedded_probe.nv`
builds.

Two things the port would have to decide.  Where the nonce comes from on
a board with no entropy source — the nRF52's `RNG` peripheral is the
answer and it is `[hw]` in the firmware rather than anything this
package does.  And what the device does with a reply it cannot check:
the honest answer is "nothing, and keep the cycle counter", because a
log with a wrong absolute time is worse than one with none.

## The device claim, and how it was checked

`tests/embedded_probe.nv` is a firmware `main` that runs eighteen
checks over `ntppkt` and `ntpcalc` — the timestamp type, the two
fixed-point scales, the era rule, the offset, the delay and its clamp —
and parks.  It builds:

```
novo build --target=nrf52-qemu src/probe.nv
novo: built probe.elf
```

**The shard audit does not build it**, and that is worth writing down:
the `core-embedded` row is a `core` package's row, and this package is
`host`, so the audit passes it with "`host` makes no device claim — the
embedded probe is `core`'s".  The claim above was built by hand, in a
scratch package holding `ntppkt`, `ntpcalc` and the probe.  That gap is
the strongest argument in the next section.

## `ntp-core-nv` should probably be a row

Unlike s3-nv, where this lane recommends against the split, here there
is a real consumer and a real check that is not being run.

- **A device wants the four `[]` modules and cannot take this
  package.**  `ntpclient` names `std.net`, and that is enough to make
  the whole package unlinkable in firmware.  The probe above proves the
  modules build; it does not prove anybody can *depend* on them.
- **The audit would then build the probe on every run.**  Today the
  claim is checked when somebody remembers to check it, which is the
  definition of a claim that will go stale.
- **The split is clean.**  `ntpclient` depends on the other four and
  nothing depends on it.  `ntp-core-nv` is `ntpfault`, `ntppkt`,
  `ntpcalc` and `ntppool` unchanged, and `ntp-nv` keeps `ntpclient` and
  takes a dependency on it.

What argues the other way is size: four small modules and about six
hundred lines, against a second name on the grid.  The decision is the
grid's rather than this package's, so it is argued here and the modules
are arranged for the day it is taken.

One note for whoever takes it: `ntpcalc.civil_of` and
`unix_seconds_of_civil` are the only two functions that name
calendar-nv, and calendar-nv makes no device claim, so a `ntp-core-nv`
that claimed `@tier(embedded)` would either leave those two behind or
carry a dependency a probe cannot link into.

## What is missing, by name

**A datagram API that can carry binary.**  `net.udp_recv_from` answers
a `Str` that the runtime NUL-terminates, so a received datagram is
truncated at its first zero byte.  An NTP packet's second byte is the
stratum — which is **zero in every kiss-o'-death reply**, the one kind
of reply a client is obliged to obey — its fourth is a signed precision,
and the root delay and root dispersion fields are usually zeroes.  In
practice almost no NTP reply survives that API intact.

The row is **UDP in `std.net` answering `Bytes`**, alongside the
`send_bytes` / `recv_bytes` pair TCP already has.  `ntpclient`'s
signatures here are written against the API that row would provide,
because writing them against the one that exists would be designing
around a defect rather than reporting it.

**Also absent from `std.net`'s datagram surface**, and named because
mdns-nv needs all of them and this package needs two: binding to a
specific local address (`udp_bind` takes a port and binds
`INADDR_ANY`), an IPv6 datagram socket (`udp_socket` is `AF_INET`
only), and a receive timeout that is documented to apply to a datagram
socket.

**Not missing, and worth saying so**: everything else this package
needs is integer arithmetic and calendar-nv, both of which are here
today.

## Where a row wanted to widen

**`[rand]`, and the design answered by not taking it.**  The nonce that
makes a reply unforgeable has to be unpredictable, and the obvious
shape — `ntpclient.query(server, timeout)` drawing its own — would have
put `[rand]` in the package's row and in every program that links it,
and would have made a reproducible test impossible.  So the nonce is an
argument, the row stays `[net, time]` exactly as the plan wrote it, and
the cost is that the caller has an obligation this README states three
times.  tls-nv made the same trade for its handshake randomness; this
is the cohort's second instance and it is worth naming as a pattern.

**Two `@value` rules shaped two types, and both are visible in the
published signatures.**

`NtpExchange` and `NtpReading` are `@value` structs *because their
fields are*: a boxed struct stores pointer-shaped slots and cannot hold
an unboxed one, so an exchange holding four `NtpTimestamp`s is a
`@value` or it is eight loose integers.  The compiler said so at
`E2015`, and the answer it pushed the design towards is the right one
for a device anyway.

`ntpcalc.unix_seconds_of_civil` answers an **`Int`** rather than an
`NtpTimestamp`, because a `@value` struct may not be a `Result` payload.
The caller pipes the number through `timestamp_of_unix`, which cannot
fail.  Same rule, and the same shape the language asks for: the fallible
step answers a raw, and the construction that follows it is total.

`NtpAnswer` is **flat** for the same reason from the other side: it
holds a `Str` — the server's name, which is what a caller needs to act
on a kiss code — so it cannot be a `@value`, and therefore it cannot
hold `NtpReading` or `NtpPacket` either.  The numbers are flattened into
it and `ntpclient.reading_of` rebuilds the `@value` for the one function
that compares two of them.

## What this does not do, on purpose

- **No clock discipline.**  This answers an offset; it does not step,
  slew, or write a drift file.  See "The load-bearing interface" for
  why that is three different right answers.
- **No peer filter, no clock filter, no intersection algorithm.**
  Those are NTP's, they need a history of readings per server, and
  RFC 4330 is explicit that a client which needs them should run NTP
  rather than SNTP.  `ntpcalc.better` is the one comparison this
  package offers and it is one round, not a history.
- **No server mode, no symmetric mode, no broadcast.**  A client only,
  which is the row.
- **No NTS, no autokey and no symmetric-key authentication.**  NTS
  (RFC 8915) is a TLS handshake and an AEAD over the packet, and it is
  a row of its own once tls-nv is implemented; the MD5 and SHA-1
  symmetric keys of RFC 5905 § 7.3 are a shared-secret scheme that
  nobody should be starting with in 2026.
- **No leap-second handling beyond reporting the indicator.**  A leap
  second is an instruction to whatever disciplines the clock, and this
  package does not discipline one.
- **No name resolution.**  `ntpclient.query` takes a host and hands it
  to `std.net`.
- **It does not print.**  Every failure is a value with a `message()`.

## The reference implementation

`ntplib` for the shape of the client and `ntp` (the reference
implementation from the NTP Project) for the packet and the rules.
RFC 4330 is the specification this package implements; RFC 5905 § 7.3
is where the packet format actually lives, and § 8 is where the delay
clamp comes from.

Three things change in the port.

`ntplib` answers `NTPStats` with a `tx_time` field a caller reads as
"the time", and its own documentation then explains that the offset is
what you want.  Here there is no such field: the reading is the offset
and the delay, and a caller that wants a wall-clock time adds the offset
to its own.

`ntplib` fills the transmit timestamp with `time.time()`, which is
predictable to within a second and makes its anti-spoofing check worth
very little.  Here the nonce is a parameter with a name and three
paragraphs about it.

And both references hide the poll loop.  Here there is no loop at all:
`ntppool`'s constants say how slow a client has to be, `after_kiss` says
what a refusal costs the list, and the schedule is in the caller's code
where its author can see it.

## Status

| item | implemented |
| --- | --- |
| `ntpfault` — `NtpKiss`, `NtpFault` | types only |
| `ntpfault.kiss_text`, `.kiss_of_text`, `.kiss_stops_forever`, `.try_another`, the `message` impl | no |
| `ntppkt` — `NtpTimestamp`, `NtpPacket` | types only |
| `ntppkt.NTP_PACKET_BYTES`, `.NTP_PORT`, `.NTP_VERSION`, `.NTP_MODE_CLIENT`, `.NTP_MODE_SERVER`, `.NTP_UNIX_EPOCH_DELTA`, `.NTP_MAX_STRATUM` | yes — they are constants |
| `ntppkt.timestamp`, `.zero_timestamp`, `.timestamp_eq`, `.timestamp_is_zero`, `.request` | no |
| `ntppkt.encode_into`, `.is_wellformed`, `.decode`, `.packet_fault` | no |
| `ntppkt.reference_text`, `.kiss_of`, `.is_kiss` | no |
| `ntppkt.root_delay_micros`, `.root_dispersion_micros`, `.server_error_micros`, `.poll_seconds`, `.precision_micros` | no |
| `ntpcalc` — `NtpExchange`, `NtpReading` | types only |
| `ntpcalc.NTP_ERA1_UNIX_START` | yes — it is a constant |
| `ntpcalc.exchange`, `.check`, `.offset_micros`, `.delay_micros`, `.delay_was_clamped`, `.reading` | no |
| `ntpcalc.worst_error_micros`, `.better` | no |
| `ntpcalc.unix_seconds_of`, `.timestamp_of_unix`, `.era_of`, `.fraction_micros` | no |
| `ntpcalc.civil_of`, `.unix_seconds_of_civil` | no |
| `ntppool` — `NtpPool` | types only |
| `ntppool.NTP_MIN_POLL_SECONDS`, `.POOL_MIN_POLL_SECONDS`, `.NTP_RATE_BACKOFF` | yes — they are constants |
| `ntppool.pool`, `.empty_pool`, `.with_server`, `.without`, `.after_kiss`, `.with_poll` | no |
| `ntppool.server_at`, `.server_count`, `.is_usable`, `.next_poll_seconds` | no |
| `ntpclient` — `NtpAnswer` | types only |
| `ntpclient.query`, `.query_pool`, `.query_best`, `.exchange_with` | no |
| `ntpclient.now_timestamp`, `.clock_precision_micros`, `.reading_of`, `.correction_micros` | no |
