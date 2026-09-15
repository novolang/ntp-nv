# ntp-nv

The Network Time Protocol (NTP) is how a machine on a network finds out
what time it is. It is specified in
[RFC 5905](https://www.rfc-editor.org/rfc/rfc5905). This package
implements the Simple Network Time Protocol (SNTP),
[RFC 4330](https://www.rfc-editor.org/rfc/rfc4330), which is a
client-only subset of NTP that sends the same forty-eight-byte packet
and does none of the filtering and clock steering a full NTP
implementation does. It is built on
[calendar-nv](https://novo-lang.org/packages/calendar-nv), for the one
conversion from a timestamp to a civil date.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What SNTP is

A client sends a forty-eight-byte packet over UDP and a server sends
one back. RFC 5905 section 7.3 defines the packet, and both NTP and
SNTP use it unchanged. RFC 4330 defines SNTP as the protocol you get by
using that packet with a single server and no history: it is
interoperable with a full NTP server, and a client that needs better
accuracy than one exchange gives should run NTP instead.

Four timestamps decide the answer. **T1** is when the client sent its
request, as the client's own clock read it. **T2** is when the request
reached the server. **T3** is when the server sent the reply. **T4** is
when the reply arrived, again by the client's clock. T2 and T3 come out
of the reply; T1 and T4 the client stamps itself.

Two numbers come out of those four. The **offset** is
`((T2 - T1) + (T3 - T4)) / 2`, how far the client's clock is behind the
server's. The **round-trip delay** is `(T4 - T1) - (T3 - T2)`, how long
the exchange took on the network. The offset cannot be more accurate
than half the delay, which is why a client with three replies keeps the
one with the smallest delay rather than the most recent.

The reply carries the server's own quality as well. The **stratum** is
how many hops the server is from a reference clock: 1 is a radio clock
or a GPS receiver, 2 to 15 are servers that got their time from another
server, and 16 means unsynchronised. The **root delay** and the **root
dispersion** are how far the server may be from that reference. The
stratum says how many hops and not how wrong, so a client that reads
only the stratum will accept a stratum-15 server that is a minute out.

The **leap indicator** is two bits. 0 is no warning, 1 and 2 announce a
leap second at the end of the day, and 3 means the server's own clock
is not synchronised. A reply with leap indicator 3 must not be used.

**Stratum 0 is not a stratum.** It means the four bytes that would hold
a reference identifier are instead four ASCII characters of
instruction, called a **kiss-o'-death** code. RFC 5905 section 7.4
lists them. `DENY` and `RSTR` mean this client must never ask this
server again. `RATE` means the client is polling too fast and must slow
down.

| Quantity | Value |
| --- | --- |
| Packet size on the wire, without extensions | 48 bytes |
| Service port | 123 |
| Version this package writes | 4 |
| Versions it accepts inbound | 3 and 4 |
| Mode of a client request | 3 |
| Mode of a server reply | 4 |
| Seconds between the NTP epoch and the Unix epoch | 2 208 988 800 |
| Highest stratum that means anything | 15 |
| The 32-bit seconds field wraps at | 2036-02-07T06:28:16Z |
| Shortest interval RFC 4330 allows against one server | 15 seconds |
| Shortest interval the public pool asks for | 64 seconds |

The packet mixes two fixed-point scales, and confusing them is out by a
factor of 65 536.

| Field | Scale | One unit is |
| --- | --- | --- |
| The four timestamps, and the reference time | 32.32 | 2⁻³² seconds |
| Root delay, root dispersion | 16.16 | 2⁻¹⁶ seconds |
| Poll interval, precision | a signed power of two | seconds |

## Install

```
novo pkg add ntp-nv
```

## Example

```novo
use ntpclient
use ntppkt

fn main() [io, net, time]
    // The nonce goes in the request's transmit field and the server
    // copies it back. It must be unpredictable; draw it from a random
    // source rather than writing a constant as this line does.
    let nonce = ntppkt.timestamp(3900000000, 0x5A5A5A5A)

    // Ask one server once, waiting at most three seconds for a reply.
    match ntpclient.query("time.example.org", nonce, 3000)
        Ok(answer) =>
            // How far this machine's clock is behind the server's, and
            // how long the exchange took. Both in microseconds.
            println("${answer.offset_micros} behind over ${answer.delay_micros}")
        Err(fault) => println(fault.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: ntp-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `ntppkt` | The forty-eight bytes as a value: the timestamp type, the packet type, the constants, encoding and decoding, and the readers that convert each fixed-point field to microseconds. |
| `ntpcalc` | The four timestamps and what may be concluded from them: the offset, the delay and its clamp, the worst-case error, the comparison of two readings, the 2036 era rule, and the conversion to a civil date. |
| `ntpfault` | Every reason a reply cannot be used, and the kiss-o'-death codes, with the question a client actually asks of each: does this stop forever, and is it worth trying another server. |
| `ntppool` | A list of servers and the rules for asking them, as a value: the poll floor, what a kiss code does to the list, and which server is next. |
| `ntpclient` | The half that touches the machine: one datagram out, one datagram in, and the two clock reads around them. |

## How to choose an entry point

**`ntpclient.query` asks one server once.** You give it a hostname, a
nonce and a timeout, and it answers the offset or a fault. It performs
the whole exchange, including the checks. Use it when you have a server
to ask and a source of randomness.

**`ntpclient.query_pool` and `query_best` take a pool.** `query_pool`
tries servers in order until one answers. `query_best` asks several and
keeps the reading with the smallest delay. Each attempt needs its own
nonce, so both take a list of them.

**`ntpclient.exchange_with` takes a packet you built.** Use it when you
want to choose your own nonce, socket lifetime and timeout, but still
want this package to stamp T4 at the instant the datagram arrived.

**`ntppkt` and `ntpcalc` together are the whole protocol with no socket
at all.** Build a request with `ntppkt.request`, send it however you
like, decode the reply with `ntppkt.decode`, call `ntpcalc.check` with
the nonce you sent, then `ntpcalc.exchange` and `ntpcalc.reading`. This
is the path for a microcontroller, which has no `std.net`, and the path
a test takes, because none of it reads a clock or opens a socket.

## The rules a user needs

1. **The nonce is yours, and it must be unpredictable.** SNTP has no
   cryptography. A reply is forty-eight bytes of UDP from an address
   anybody can spoof, and the only thing in it an off-path attacker
   cannot know is the transmit timestamp the client sent, which the
   server copies into the reply's originate field. RFC 4330 section 3
   requires that field to be unpredictable. A caller that passes a
   counter, or the current time, has an SNTP client with no
   anti-spoofing at all.
2. **`ntpcalc.check` is the function that performs that comparison, and
   it is separate from `ntppkt.packet_fault`.** `packet_fault` makes
   the five checks that need only the reply: mode, version, leap
   indicator, stratum and a zero transmit timestamp. `check` makes
   those and the one that needs the request. A client that calls only
   `packet_fault` has skipped the check that matters. RFC 4330
   section 5 lists them.
3. **An exchange answers an offset, not a time.** `NtpReading` has no
   field saying what the time is. Stepping the clock breaks every
   monotonic assumption above it, slewing takes hours to converge, and
   a device with no clock at all just wants a number to add to its
   counter. Those are three right answers, and this package leaves the
   choice to you.
4. **A client reads its own clock exactly twice per exchange.** Once to
   stamp T1 and once to stamp T4. A client that reads it once has
   thrown away the round trip it was measuring.
   `ntpclient.now_timestamp` is the only function here that reads a
   clock.
5. **The seconds field wraps in 2036, and the conversion is not a
   subtraction.** RFC 4330 section 3 gives the rule: the high-order bit
   of the seconds field decides the era. Set means 1968 to 2036 and the
   1900 epoch. Clear means 2036 to 2104 and the next one.
   `ntpcalc.unix_seconds_of` applies it, `ntpcalc.era_of` reports which
   era a timestamp is in, and `NTP_ERA1_UNIX_START` is the instant of
   the wrap. Code that subtracts 2 208 988 800 unconditionally starts
   answering dates in 1900 on that morning, from packets that are
   perfectly valid.
6. **The two fixed-point scales are not the same scale.** See the table
   above. `ntppkt.root_delay_micros`, `root_dispersion_micros`,
   `poll_seconds`, `precision_micros` and `ntpcalc.fraction_micros`
   each do their own conversion once, so nothing else has to remember
   which field is in which scale. RFC 5905 section 7.3 defines both.
7. **A negative delay is clamped, not refused.** When the client's
   clock is coarser than the network is fast, which is every
   microcontroller, `(T4 - T1) - (T3 - T2)` comes out below zero.
   RFC 5905 section 8 says to clamp it to the system precision.
   `ntpcalc.delay_micros` takes your clock's precision in microseconds
   and clamps to it, and `NtpReading.clamped` records that it did.
8. **`DENY` and `RSTR` stop forever; `RATE` means back off.** That
   distinction is the whole of a client's obligation to a kiss-o'-death
   code. Treating `RATE` as permanent loses a working server. Treating
   `DENY` as temporary is what gets an address range banned from the
   public pool. `ntpfault.kiss_stops_forever` answers which it is, and
   `ntppool.after_kiss` applies both rules to the list in one call.
9. **A reply's leap indicator of 3 is a refusal.** The server is saying
   its own clock is not synchronised. `ntpcalc.check` reports it as
   `NtpFaultUnsynchronised`.
10. **Read the root delay and root dispersion, not just the stratum.**
    `ntppkt.server_error_micros` is RFC 5905's own expression for how
    far from the reference a server may be:
    `root_dispersion + root_delay / 2`.
    `ntpcalc.worst_error_micros` adds half the round trip to it, which
    is how wrong the whole reading may be.
11. **This package contains no loop and no default server.** `query`
    asks one server once. The poll schedule is yours and lives in your
    code. `NtpPool`'s server list starts empty, and `query_pool`
    refuses an empty pool rather than falling back to somebody else's
    volunteers. RFC 4330 section 10 sets the floor at 15 seconds
    against one server, `NTP_MIN_POLL_SECONDS` is that number, and
    `POOL_MIN_POLL_SECONDS` is the 64 seconds the public pool asks for.
12. **The server's advertised poll interval is an instruction.** Every
    reply carries one. A server that says "ask me every 1024 seconds"
    expects to be obeyed, and `ntppool.next_poll_seconds` takes the
    longer of that and yours.
13. **A fallible conversion answers a number, not a timestamp.**
    `ntpcalc.unix_seconds_of_civil` answers `Result<Int, CalError>`,
    because the language refuses a `@value` struct as a `Result`
    payload and `NtpTimestamp` is one. Pipe the number through
    `ntpcalc.timestamp_of_unix`, which cannot fail.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers `ntppkt` and `ntpcalc`: the timestamp and
packet types, the codec, the fixed-point readers, the offset and delay
arithmetic and the era rule. All of it is integer arithmetic over two
thirty-two-bit halves.

```bash
novo build --target=nrf52-qemu src/main.nv
```

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It builds a Cortex-M4 executable that runs eighteen checks
over the timestamp type, the two fixed-point scales, the era rule, the
offset, the delay and its clamp.

**A device cannot depend on this package as a whole.** `ntpclient`
names `std.net`, and one host-only function anywhere in a compilation
unit is an undefined symbol at link time on a device, whether or not
the firmware calls it. The probe above is built against `ntppkt` and
`ntpcalc` on their own. Until those two modules are published
separately, firmware that wants them copies them or waits.

`ntpcalc.civil_of` and `ntpcalc.unix_seconds_of_civil` are outside the
claim as well. They name calendar-nv's types, and a device adding an
offset to its own counter wants the integer rather than the calendar.

On a device the nonce comes from a hardware random number generator,
which is the firmware's business rather than this package's. A device
that receives a reply it cannot check should keep its own counter and
log nothing absolute, because a log with a wrong absolute time is worse
than one with none.

## What is not included

- **A UDP socket that can carry binary.** `std.net`'s
  `udp_recv_from` answers a string the runtime terminates at the first
  zero byte. An NTP packet's second byte is the stratum, which is zero
  in every kiss-o'-death reply, its fourth is a signed precision, and
  the root delay is usually zeroes, so almost no reply survives that
  API intact. `ntpclient`'s signatures are written against a datagram
  API that answers bytes, beside the `send_bytes` and `recv_bytes` pair
  the standard library's TCP already has. Until that exists,
  `ntppkt` and `ntpcalc` with your own transport are the working path.
  Binding a datagram socket to a specific local address, an IPv6
  datagram socket, and a documented receive timeout on a datagram
  socket are absent for the same reason.
- **Clock discipline.** This package answers an offset. It does not
  step the clock, slew it or write a drift file. See rule 3.
- **The peer filter, the clock filter and the intersection algorithm.**
  Those are NTP's, they need a history of readings per server, and
  RFC 4330 says a client that needs them should run NTP rather than
  SNTP. `ntpcalc.better` compares two readings from one round, which is
  as far as this goes.
- **Server mode, symmetric mode and broadcast mode.** A client only.
- **Network Time Security, autokey and symmetric-key authentication.**
  NTS (RFC 8915) is a TLS handshake and an authenticated encryption
  layer over the packet, and it is a package of its own. The MD5 and
  SHA-1 symmetric keys of RFC 5905 are a shared-secret scheme nobody
  should start with today.
- **Leap-second handling beyond reporting the indicator.** A leap
  second is an instruction to whatever disciplines the clock, and this
  package disciplines none.
- **Name resolution.** `ntpclient.query` takes a hostname and hands it
  to the standard library.
- **Printing.** Every failure is a value with a `message()`.

## Related packages

- `std.time` in the standard library is the machine's own clocks:
  `Mono` for measuring a duration, `Instant` and `Zoned` for the wall
  clock, and RFC 3339 formatting. It tells you what this machine thinks
  the time is. It has no way of telling you whether that is right. This
  package is the other half: it asks a server and answers how far this
  machine's clock is from it, in microseconds. A program that wants a
  correct wall-clock time reads `std.time` and adds this package's
  offset to it.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is civil
  dates, times and durations with no clock in them. This package
  depends on it for `ntpcalc.civil_of`, which is the only function here
  that names a calendar type.
- [mdns-nv](https://novo-lang.org/packages/mdns-nv) is multicast DNS
  and service discovery. It is the other UDP client on the registry,
  and it is how a program finds an NTP server advertised on a local
  network rather than being told one.
- [dns-codec-nv](https://novo-lang.org/packages/dns-codec-nv) is the
  RFC 1035 wire format, for resolving a server name without the
  standard library doing it.

## Tests

```bash
novo test tests/ntpwire_tests.nv      # 13 tests: the packet and the arithmetic
novo test tests/ntpclient_tests.nv    #  9 tests: the pool and the client surface
```

The reference implementations are `ntplib` for the shape of the client
and the NTP Project's own `ntp` for the packet and the rules. RFC 4330
is the specification the client implements, RFC 5905 section 7.3 is
where the packet format lives, and RFC 5905 section 8 is where the
delay clamp comes from.

No test opens a socket or reads a clock. The four timestamps, the nonce
and the client's own precision are all arguments, so a whole exchange
is a value the test writes out and the same bytes produce the same
offset on every run. The suite checks that a reply must echo the nonce,
that an unsynchronised server is refused, that a mode-3 reply is this
client's own request coming back, that a kiss-o'-death is read as an
instruction before its stratum is read, that the two fixed-point scales
are not the same scale, that the era rule is not a subtraction, that a
negative delay is clamped rather than refused, that the better of two
readings is the faster and not the newer, and that an empty pool
refuses rather than falling back.

The tests compile today and fail at run, each on the
`not implemented: ntp-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `ntppkt.NTP_PACKET_BYTES`, `.NTP_PORT`, `.NTP_VERSION`, `.NTP_MODE_CLIENT`, `.NTP_MODE_SERVER`, `.NTP_UNIX_EPOCH_DELTA`, `.NTP_MAX_STRATUM` | yes (they are constants) |
| `ntpcalc.NTP_ERA1_UNIX_START` | yes (it is a constant) |
| `ntppool.NTP_MIN_POLL_SECONDS`, `.POOL_MIN_POLL_SECONDS`, `.NTP_RATE_BACKOFF` | yes (they are constants) |
| `ntppkt.timestamp`, `.zero_timestamp`, `.timestamp_eq`, `.timestamp_is_zero`, `.request` | no |
| `ntppkt.encode_into`, `.is_wellformed`, `.decode`, `.packet_fault` | no |
| `ntppkt.reference_text`, `.kiss_of`, `.is_kiss` | no |
| `ntppkt.root_delay_micros`, `.root_dispersion_micros`, `.server_error_micros`, `.poll_seconds`, `.precision_micros` | no |
| `ntpcalc.exchange`, `.check`, `.offset_micros`, `.delay_micros`, `.delay_was_clamped`, `.reading` | no |
| `ntpcalc.worst_error_micros`, `.better` | no |
| `ntpcalc.unix_seconds_of`, `.timestamp_of_unix`, `.era_of`, `.fraction_micros` | no |
| `ntpcalc.civil_of`, `.unix_seconds_of_civil` | no |
| `ntpfault.kiss_text`, `.kiss_of_text`, `.kiss_stops_forever`, `.try_another`, `NtpFault.message` | no |
| `ntppool.pool`, `.empty_pool`, `.with_server`, `.without`, `.after_kiss`, `.with_poll` | no |
| `ntppool.server_at`, `.server_count`, `.is_usable`, `.next_poll_seconds` | no |
| `ntpclient.query`, `.query_pool`, `.query_best`, `.exchange_with` | no |
| `ntpclient.now_timestamp`, `.clock_precision_micros`, `.reading_of`, `.correction_micros` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
