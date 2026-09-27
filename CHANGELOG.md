# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.0 — 2026-09-27

humantime's grammar in both directions, the arithmetic over the length
it spells, and RFC 3339 in UTC over calendar-nv 0.2's civil types.

- `humantime.parse_duration` reads every spelling in humantime's unit
  table, including `nsec`, `nanos`, `usec`, `msec` and `millis`, which
  the 0.0.2 README left out.  A fraction is exact to the nanosecond,
  and digits past the ninth are dropped.  Every refusal names its
  offset.
- `format_duration` writes years, months and days as words with a
  plural and the smaller units as symbols, as humantime does:
  `1year 2months 3days 4h 5m 6s`.  A week is written as seven days.
- `format_approx` rounds to the nearest whole count, with a length
  exactly halfway rounded down, and says a count that rounds into the
  next unit in that unit.
- `htdur.mul_int` and every constructor saturate at `max_value`, and
  `div_int` is exact to the nanosecond for any divisor.
- A timestamp field out of range, a leap second included, is
  `HtTimestampOutOfRange` at that field's offset.  The 0.0.2 comments
  listed a month of 13 under `HtBadTimestamp`.
- The dependency is `calendar-nv ^0.2.0`, and the toolchain floor is
  0.13.0.

Breaking change against 0.0.2: `humantime.parse_duration`,
`htstamp.since` and `htstamp.from_time_delta` answer
`Result<HtDuration, HtError>` rather than `Result<(Int, Int), HtError>`.
A `@value` struct is accepted as the payload of a `Result` by toolchain
0.13.0, so the pair of integers and the `htdur.new` call after every
parse are gone.

## 0.0.2 — 2026-09-16

README rewritten to the package README style guide
(docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Three modules.  `htdur` is the length itself — a `@value` of seconds
  and nanoseconds, never negative, with the ten unit constructors and
  the arithmetic a timeout and a backoff need.  `humantime` is the
  grammar both ways.  `htstamp` is RFC 3339 over calendar-nv's civil
  types.
- **The grammar, whole**: terms are added, the number may carry a
  decimal point, the space before a unit is optional, and `m` is
  minutes while `M` is months — humantime's rule, kept, because a
  package that quietly differed from the grammar it is named after
  would be worse than one that carries an awkward rule and says so.
- **A year is 365.25 days and a month is a twelfth of one.**  Fixed
  lengths, not calendar arithmetic: adding `1month` gives 30.4375 days
  later and not the same day of the next month.  calendar-nv's
  `arith.add_months` is the other one, and the README has the table.
- **Two formatters, not one with a flag.**  `format_duration` is the
  shortest spelling that means exactly this length and round-trips;
  `format_approx` is `"about 2 hours"` and `"just now"` and
  deliberately does not, because those words are not in the grammar.
  `format_duration_from` drops the terms below a floor, so a measured
  elapsed time reads as `2h 30m 4s` rather than as seven terms.
- **A bare number is refused.**  `"30"` is `HtNumberWithoutUnit`, not
  thirty seconds: a bare number in a configuration file is ambiguous
  between seconds and milliseconds.
- **The length has no sign**, because the grammar has none.  `sub`
  traps when the right side is longer and `sat_sub` clamps to zero,
  which is what a deadline wants.
- **RFC 3339 with no clock in it**: a strict parser for a wire format
  and a lenient one for text a person typed, five formatters because
  the precision is a decision made once, and `since` taking BOTH
  instants — which is what keeps the package `core` and what lets a
  test pin the time.

**One dependency, calendar-nv**, for the civil types the timestamp half
reads and writes and the signed `TimeDelta` it crosses to.  A package
that parsed its own dates would have two definitions of a leap year in
one program.

**No device claim**, on purpose: the two useful halves of this package
are text, and `Str` does not link at `@tier(embedded)`.  `htdur` alone
would compile for a device, and a probe covering a tenth of the surface
would claim something the package does not mean.

**Every fallible direction answers integers**, not a length: a `@value`
struct may not be a `Result` payload, so `parse_duration` answers
`Result<(Int, Int), HtError>` and `htdur.new` is the line that follows.
