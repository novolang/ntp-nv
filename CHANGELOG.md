# Changelog

All notable changes to ntp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
