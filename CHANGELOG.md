# Changelog

All notable changes to ntp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## Unreleased

### Fixed

- Repeated `RATE` kisses back the poll interval off to at most 2^17 seconds instead of doubling it until the multiplication overflows.

## 0.1.0 — 2026-10-13

The **implementation**.  Every body under `src/` lands, the three
suites under `tests/` run green, and 92% of the instrumented lines
under `src/` are covered by them with the remaining nine regions
excused by a `// cov: skip` marker that names its reason.  0.0.3 is the
interface this release implements; no effect row changed, and one
frozen signature had to move — `ntppkt.encode_into`'s buffer gains
`var` — for the reason in the third bullet under `### Changed`.

### Added

- The bodies of all five modules.  `ntpfault`'s thirteen messages and
  the kiss-o'-death mappings; `ntppkt`'s codec, the two fixed-point
  scales and the five packet faults; `ntpcalc`'s offset, delay, era
  rule and the two civil conversions; `ntppool`'s policy arithmetic;
  and `ntpclient`'s socket, clock reads and two rounds.
- `tests/ntp_edge_tests.nv` — 18 tests over the boundaries the two
  oracle suites leave: a poll exponent past `2^30` and a negative one,
  a precision of minus thirty-two and of minus twenty, the byte the
  packet calls signed (`0xFFFFFFFF` is minus one, not four billion),
  a kiss code nobody registered, four zero bytes where a code should
  be, a stratum above fifteen, a buffer one byte too short and an
  offset past its end, an era-1 timestamp on both sides of 2036, the
  second before 1970, a civil time built field by field with a 25th
  hour, every fault's message verbatim, and the pool policy at its
  floor.  It also drives `ntpclient` against three addresses that no
  server can answer from — a name that does not resolve, the loopback
  port 123, and the broadcast address — which covers the transport,
  the timeout and the send-failure paths end to end.
- `tests/embedded_probe.nv` was the interface's device claim, and its
  eighteen checks are its own, with the one arithmetic correction
  `### Changed` records.  `score()` answers 18 when the checks are run
  (as a host program over `ntppkt`, `ntpcalc` and `ntpfault`, the
  file's device `main` replaced by a host one); the device build the
  probe exists for is blocked, and `### Known` says why.

### Changed

- `novo.toml`: `version = "0.1.0"`, `stability = "experimental"`,
  `novo = ">= 0.20.2"` (the version this release was verified against),
  the calendar-nv range to `^0.2.1` — the release the device-build
  finding below is measured against — and the description without its
  interface tail.
- README: the implementation-status table and the "red until the bodies
  land" paragraph are gone, the `Tests` section names all three suites,
  the `Running on a microcontroller` section is the build that runs,
  and the "what is not included" bullet names what is actually absent.
- **`ntppkt.encode_into`'s published buffer gains `var`.**  The
  interface froze `fn encode_into(out: [u8], at: Int, p: NtpPacket)`
  with the prose "Answers how many bytes were written, or zero when the
  buffer is too small — a device sizes its buffer once and this is the
  check that it did", which can only be written by filling `out`: a
  write through a plain (non-`var`) parameter is refused (`E2036`), and
  declaring the parameter `var` moves the refusal to every `let`-bound
  buffer (`E2038`), which is how the interface's own fixture binds it.
  One of the two had to give, so the parameter is `var out: [u8]` in
  this release and the one oracle fixture binds the buffer with `var`.
  Both arms are sighted on
  `language/a-frozen-into-signature-cannot-fill-its-buffer`.  A diff of
  every `pub` line against 0.0.3 shows it is the only signature in the
  package that differs.
- **Every published function now declares a tier.**  Five were left
  unannotated in the interface — `ntpcalc.exchange`, `ntpcalc.reading`,
  `ntpfault.kiss_text`, `ntpfault.kiss_stops_forever` and
  `ntpfault.try_another` — and an unannotated function is read one way
  by a device build (fail-closed, `@tier(embedded)`) and another by
  every other build (the project's tier, `app` by default), so a
  `@tier(rt)` caller refused one in a host build while a device build
  accepted the very same call.  All five are `@tier(embedded)` here:
  they are pure matching over `@value` enums and constant `Str`
  answers, the device build had already been checking them there, and
  the probe above is a `@tier(rt)` program.  `ntpcalc.check` is
  `@tier(rt)` (it constructs `?NtpFault` payloads) and the two civil
  conversions stay `@tier(app)`, as the 0.0.1 entry describes.
- **The interface release's claim that a `@value` struct may not be a
  `Result` payload is false.**  Measured on this toolchain: `Ok(Pt { x: 1 })`
  with `@value struct Pt` compiles and runs on the host — the struct
  crosses by boxing a copy, which is an allocation.  What the language
  refuses is a payload-carrying `Result` at `@tier(embedded)` (`E4000`),
  and THAT is why `ntppkt.decode` answers a packet with a fault field
  rather than a `Result` of one: the decoder is `@tier(embedded)`.
  `ntppkt`'s module comment says so now, and the 0.0.1 entry below is
  kept as the record of what that release said.
- **`ntpcalc.unix_seconds_of_civil` propagates the `CalError`.**  The
  body this release lands matches the documented refusal: a
  `CivilDateTime` whose time fields are out of range answers
  `Err(CalError.TimeOutOfRange(...))`.  The `!` that would have panicked
  on it is gone, and there is a test for the 25th hour and for a
  nanosecond past the second.
- Two oracle fixtures are corrected where the interface release
  contradicted its own arithmetic (both are class 1 in
  `../ORACLE-EDITS.md`).  `ntpwire_tests.nv`'s
  `test_the_offset_is_the_average_and_the_delay_is_the_difference`
  asserted `offset_micros == 2500000` over its own four timestamps,
  whose formula — written out two lines above the assertion — is
  `(2 + 1) / 2 = 1.5` seconds, so the assertion now reads `1500000` and
  the case name says "a second and a half"; the probe's `an_exchange()`
  check repeated the same value over the same fixture and is corrected
  the same way.  Every other assertion in both files is the release's,
  and the one `let`-bound buffer in `ntpclient_tests.nv` is the `var`
  the frozen signature needs (third bullet above).  The suites'
  comments are corrected where the bodies made them false — the nonce
  check's `@value`-as-payload sentence in `ntpclient_tests.nv`, which
  this release measured to be wrong — and both suites' "RED ON PURPOSE"
  release stamps are gone, which is what landing means
  (`../ORACLE-EDITS.md`, "Not in this file").
- Nine lines under `src/` carry a `// cov: skip` marker, all in
  `ntpclient`, each with its reason: six for the path a reply that
  arrived takes (an answer needs a server on port 123, which is below
  the unprivileged floor), two for the `server_at` guards in the round
  functions (`is_usable` was checked before the round, so `server_at`
  cannot answer `None` at an `i` the round reaches), and one for an
  unopenable datagram socket (an exhausted descriptor table).

### Known

- **A device build of this package fails inside `calendar-nv`.**  The
  five modules here are device-clean — zero tier errors — but
  `novo.toml` declares calendar-nv `^0.2.1`, no function in calendar-nv
  carries a `@tier` annotation, and every one of its functions is
  therefore checked at the consumer's severity.  Staging the probe into
  `src/` and building for `nrf52-qemu` reports **15 `E4000` errors, all
  of them in `calendar-nv` 0.2.1** (`calerror.nv:95` and fourteen in
  `civil.nv`), and zero in this package.  There is no knob in this
  package's manifest that reaches them: measured, `[tier]
  sibling_check = "warn"` here leaves all fifteen as hard errors (it
  downgrades a package's own sibling modules, not a registry
  dependency's) and would stop this package's own modules being checked
  at the tier it claims.  Filed in the tracker under
  `embedded-devex/`:
  `calendar-nv-s-unannotated-functions-make-every-consumer-s-device-build`.
- **`std.net`'s `udp_send_to` cannot send a `Bytes` payload.**  Measured:
  `udp_send_to(sock, host, port, data: Any)` sends a `Str` by the
  payload's own length, but a `Bytes` argument goes out as the buffer's
  length word, little-endian, cut at its first zero byte — a six-byte
  buffer arrives as `06`, a 48-byte frame as `30` — and the call still
  answers `true`.  `ntpclient.exchange_with` therefore crosses the wire
  as a `Str` (`bytes.to_str`), which is the right length but is not what
  the `Bytes` parameter is for; the day the byte-typed half lands this
  line simplifies.  Filed in the tracker under `stdlib/`:
  `net-udp-send-to-sends-a-bytes-payload-as-its-length-word`.
- **What is actually missing from `std.net`** is a byte-typed datagram
  API (`send_bytes` / `recv_bytes`, which TCP has) and an IPv6 socket.
  The 0.0.1 entry below named a NUL-terminated `udp_recv_from` and the
  absence of local-address binding and of a receive timeout; all three
  were wrong — `udp_recv_from` answers the kernel's length with a zero
  byte as a byte like any other, and `udp_bind_on` and
  `set_recv_timeout` both exist and are what this package uses.
- **`ntpclient.query` cannot answer `Ok` in a test.**  Port 123 is
  below the unprivileged floor, so no suite here can put a server where
  it would answer.  The six excused regions are that path; the
  arithmetic it wraps is covered through `ntpcalc` directly and the
  codec through `ntppkt` directly, so what is unverified is the wiring
  and not the arithmetic.

## 0.0.3 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- calendar-nv: `^0.0.2` to `^0.2.0`.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `ntpfault` — the faults and the kiss-o'-death codes, `[]` throughout:
  `DENY` and `RSTR` stop forever and `RATE` means back off, which is
  the one distinction that decides whether a client gets an address
  range banned from the public pool.
- `ntppkt` — the forty-eight bytes as a `@value`, `[]` throughout and
  `@tier(embedded)` on the arithmetic: the two fixed-point scales the
  packet mixes kept apart, the nonce as an argument to `request`, and
  `is_wellformed` / `decode` / `packet_fault` as three functions
  because a `@value` struct cannot be a `Result` payload.
- `ntpcalc` — the four timestamps and what may be concluded from them,
  `[]` throughout and `@tier(embedded)` on everything but the civil
  conversion: the offset and delay, the 2036 era rule as a rule rather
  than a subtraction, and a negative delay clamped rather than refused.
- `ntppool` — the server list and the poll policy as a value, `[]`
  throughout: the default list is EMPTY, the floor is a named constant
  with the reason beside it, and there is no loop in the package.
- `ntpclient` — the host half: `[net, time]` and nothing else, with
  `[time]` spent exactly twice per exchange.
- `tests/embedded_probe.nv` — eighteen checks over `ntppkt` and
  `ntpcalc`, built for `--target=nrf52-qemu` by hand, because the
  audit's `core-embedded` row belongs to `core` packages and this one
  is `host`.
- API tests in `tests/ntpwire_tests.nv` — a whole exchange with no
  socket and no clock — and `tests/ntpclient_tests.nv`.  Red until the
  bodies land.

### Named as missing

**A datagram API that can carry binary.**  `net.udp_recv_from` answers
a NUL-terminated `Str`, so a reply is truncated at its first zero byte
— and an NTP packet's second byte is the stratum, which is zero in
every kiss-o'-death reply.  The row is UDP in `std.net` answering
`Bytes`, beside the `send_bytes` / `recv_bytes` pair TCP already has.
Also named: binding a datagram socket to a specific address, and an
IPv6 datagram socket.

**`ntp-core-nv`**, which the README recommends taking: `ntpclient`
names `std.net` and that is enough to make the whole package unlinkable
in firmware, so the device this package was designed for cannot depend
on it.

### Design notes

Moved here from the 0.0.1 README, which argued them at length.

**`ntp-core-nv` should probably exist.**  `ntpclient` names `std.net`,
and that is enough to make the whole package unlinkable in firmware, so
the device this package was designed for cannot depend on it.  The
split is clean: `ntpclient` depends on the other four modules and
nothing depends on it, so `ntp-core-nv` would be `ntpfault`, `ntppkt`,
`ntpcalc` and `ntppool` unchanged, and `ntp-nv` would keep `ntpclient`
and take a dependency on it.  It would also mean the device claim is
built on every audit run instead of by hand.  What argues against it is
size: four small modules and about six hundred lines, against a second
name to maintain.  Whoever takes it should note that `ntpcalc.civil_of`
and `unix_seconds_of_civil` are the only two functions naming
calendar-nv, and calendar-nv makes no device claim, so an
`ntp-core-nv` claiming `@tier(embedded)` would either leave those two
behind or carry a dependency a probe cannot link.

**`[rand]` was kept out of the effect list deliberately.**  Drawing the
nonce inside `ntpclient.query` would have put `[rand]` in the row of
every program that links this package, and would have made a
reproducible test impossible.  The nonce is an argument instead, and
the cost is an obligation on the caller.

**Three types took their shape from the language rather than from the
protocol.**  `NtpExchange` and `NtpReading` are `@value` structs
because their fields are, and a boxed struct cannot hold an unboxed
one (the compiler reports `E2015`).  `ntpcalc.unix_seconds_of_civil`
answers an `Int` because a `@value` struct may not be a `Result`
payload.  `NtpAnswer` is flat because it holds a `Str`, so it cannot be
a `@value` and therefore cannot hold `NtpReading` or `NtpPacket`
either; `ntpclient.reading_of` rebuilds the `@value` from it.

**Three differences from the reference implementations.**  `ntplib`
answers an `NTPStats` with a `tx_time` field a caller reads as "the
time"; here the reading is the offset and the delay, with no such
field.  `ntplib` fills the transmit timestamp with `time.time()`, which
is predictable to within a second; here the nonce is a named parameter.
Both `ntplib` and the NTP Project's `ntp` hide the poll loop; here
there is no loop at all.
