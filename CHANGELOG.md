# Changelog

All notable changes to ntp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
