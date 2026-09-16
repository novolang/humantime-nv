# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
