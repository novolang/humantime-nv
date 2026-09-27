# humantime-nv

`"2h 30m"`, `"1.5s"`, `"250ms"` and `"1y 2months"` are lengths of time
written the way a person writes them in a configuration file. The
grammar is [humantime](https://docs.rs/humantime)'s, a Rust crate that
parses those strings and prints them back, and that also carries a pair
of RFC 3339 timestamp parsers. This package brings both to novo-lang.
There is no clock in it: every function takes the instants it needs as
arguments.

## What it is

A **duration** here is a length of time, exact to the nanosecond and
never negative. It is written as a series of **terms**, and the terms
are added: `"2h 30m"` is two and a half hours, and `"1h 1h"` is two
hours. A term is a number and a unit. The number may carry a decimal
point, so `"1.5s"` is a second and a half. A space or a tab may stand
between the number and the unit, and between two terms, and neither is
required.

These are the units, with every spelling the grammar accepts.

| Unit | Spellings | Seconds |
| --- | --- | --- |
| nanosecond | `ns`, `nsec`, `nanos` | 0.000000001 |
| microsecond | `us`, `µs`, `usec` | 0.000001 |
| millisecond | `ms`, `msec`, `millis` | 0.001 |
| second | `s`, `sec`, `secs`, `second`, `seconds` | 1 |
| minute | `m`, `min`, `mins`, `minute`, `minutes` | 60 |
| hour | `h`, `hr`, `hrs`, `hour`, `hours` | 3600 |
| day | `d`, `day`, `days` | 86400 |
| week | `w`, `week`, `weeks` | 604800 |
| month | `M`, `month`, `months` | 2630016 |
| year | `y`, `year`, `years` | 31557600 |

A month and a year are **fixed lengths**, not calendar arithmetic. A
year is the Julian year of 365.25 days and a month is a twelfth of it,
which is 30.4375 days. Adding one month to the 31st of January
therefore lands 30.4375 days later and not on the 28th of February.

The other half of the package is **RFC 3339**, the timestamp format
`2026-09-11T14:30:00Z`, specified in
[RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) section 5.6. This
package reads and writes the UTC form of it, over the civil date and
time types of
[calendar-nv](https://novo-lang.org/packages/calendar-nv). A **civil**
date and time is a calendar date and a time of day with no zone
attached: the reading on a wall clock.

Nothing in the package reads a clock. `htstamp.since` takes both
instants; a caller who wants "now" gets it from `std.time` or
[chrono-nv](https://novo-lang.org/packages/chrono-nv) and passes it in.
No function declares an effect, and a test can pin the time by passing a
different value.

## Install

```
novo pkg add humantime-nv
```

humantime-nv needs a novo-lang toolchain of 0.13.0 or newer.

## Example

```novo
use std.str
use htdur
use humantime

fn main() [io]
    // A timeout as it was written in a configuration file.
    match humantime.parse_duration("2h 30m")
        Ok(d) =>
            // The length as a whole number of milliseconds, truncated.
            println(str.from_int(htdur.as_millis(d)))   // 9000000
            // The shortest spelling that means exactly this length.
            println(humantime.format_duration(d))       // 2h 30m
            // The same length as a person would say it.
            println(humantime.format_approx(d))         // about 2 hours
        // The refusal carries the byte offset it was found at.
        Err(e) => println(e.message())
```

Build with `novo pkg build` and run the suites with `novo test`.

## What the package contains

| Module | Contents |
| --- | --- |
| `htdur` | The length itself: a struct of seconds and nanoseconds, a constructor per unit of the grammar, the six unit lengths as numbers, the readers, and the arithmetic a timeout and a retry backoff need. |
| `humantime` | The grammar in both directions: three ways to read a string, four ways to write a length, and the refusals with the byte offset each was found at. |
| `htstamp` | RFC 3339 over calendar-nv's civil types: a strict parser, a lenient one, five formatters, the length between two instants, and the two conversions to calendar-nv's own signed length. |

## How to choose an entry point

**`humantime.parse_duration` is the ordinary read.** It answers an
`HtDuration`.

**`humantime.parse_duration_secs` refuses anything below a second.**
Take it when the consumer has second resolution — an HTTP `max-age`, a
cache lifetime, a schedule — so that a configuration saying `"500ms"` is
reported rather than silently truncated.

**`humantime.is_duration` answers yes or no and builds no error.** For a
form field, a linter or a completion candidate.

**`humantime.format_duration` round-trips.** It is the shortest spelling
that means exactly this length, largest unit first, and parsing it gives
the length back. Use it in a log line, a serialised value, anything
another program reads.

**`humantime.format_duration_from` drops the small terms.** It takes the
smallest unit worth printing and stops there, so a measured elapsed time
reads as `2h 30m 4s` rather than as six terms.

**`humantime.format_approx` is for a person to read.** One rounded term
with a word in front: `"about 2 hours"`, `"just now"`. It does not
round-trip. See rule 4.

**`htstamp.parse_rfc3339` is strict and `parse_rfc3339_weak` is
lenient.** Take the strict one for a wire format, a log line or a signed
token. Take the lenient one for text a person typed: a space where the
`T` goes, a missing `Z`, or a bare date meaning midnight.

**The five RFC 3339 formatters are one per precision.**
`format_rfc3339` prints as few fractional digits as say the instant
exactly; the other four fix the precision at seconds, milliseconds,
microseconds or nanoseconds. See rule 10.

## The rules a user needs

1. **`m` is minutes and `M` is months.** This is the one place the
   grammar is case-sensitive, and it is humantime's own rule. A
   configuration that says `"3M"` where `"3m"` was meant asks for three
   months of timeout.
2. **A bare number is refused.** `"30"` is `HtNumberWithoutUnit`, not
   thirty seconds. A bare number in a configuration file is ambiguous
   between seconds and milliseconds, and the two are three orders of
   magnitude apart.
3. **A length has no sign.** The grammar has none, so the type has none.
   `htdur.sub` traps when the right side is longer, and `htdur.sat_sub`
   clamps to zero, which is what a deadline wants: the time remaining is
   never less than none. A leading `-` in the input is
   `HtBadCharacter`, not a negative duration.
4. **`format_duration` round-trips and `format_approx` does not.**
   `parse_duration(format_duration(d))` gives `d` back for every `d`,
   including the lengths that are a whole number of years or months.
   `format_approx` produces `"about"` and `"just now"`, which are not in
   the grammar, so its output cannot be parsed. They are two functions
   rather than one with a flag, so that a call site says which it meant.
5. **A fraction is exact to the nanosecond and no finer.** `"1.5h"` is
   5400 seconds exactly, and `"0.5ns"` is zero. Fractional digits past
   the ninth are read and dropped.
6. **A month is 30.4375 days and a year is 365.25 days.** Exact lengths,
   for "expire this entry in a month". For "bill this customer next
   month", which is a calendar question, use calendar-nv's
   `arith.add_months`. `htstamp.add` and `.sub` are the exact-length
   arithmetic, not the calendar kind.
7. **Every conversion out of a length truncates.** `htdur.as_millis`,
   `.as_micros` and `.as_nanos` throw away what does not fit, because
   the consumers are timeouts and a timeout that rounded up would fire
   late. `format_duration_from` truncates too, so its answer never
   overstates the length.
8. **`htdur.as_nanos` saturates past about 292 years.** An `Int` is 64
   bits and nanoseconds run out there. A saturated answer is one a
   caller can notice; a wrapped one is not.
9. **`htdur.add` traps on overflow, and the rest of the arithmetic
   saturates.** `htdur.sat_add` and `htdur.mul_int` stop at
   `htdur.max_value()`, so a backoff that has run long stops growing.
   `htdur.div_int` by zero answers `htdur.max_value()`, so a caller
   dividing by a count that turned out to be empty gets a number it can
   see rather than a stopped program. A negative multiplier or divisor
   answers zero.
10. **The RFC 3339 precision is fixed per function, not per call.** A
    log whose timestamps are sometimes six fractional digits and
    sometimes nine does not sort as text, which is the one property a
    timestamp format is chosen for. `format_rfc3339_millis` always
    prints three digits, including `.000`.
11. **A leap second is refused.** `:60` has no place in calendar-nv's
    `CivilTime`, and a timestamp silently moved to the next minute would
    not round-trip. It is `HtTimestampOutOfRange` at the offset of the
    seconds, as a month of 13 or a 25th hour is at theirs.
12. **The timestamp half is UTC only.** Both parsers require, or assume,
    `Z`. A timestamp carrying an offset such as `+02:00` goes through
    calendar-nv's `iso8601.parse_rfc3339`, which answers the offset as a
    second value.
13. **`htstamp.since` refuses a `later` that is earlier.** The answer is
    a length, and a length has no sign; the refusal is
    `HtTimestampOutOfRange` at offset `-1`. A caller who wants a signed
    answer wants `htstamp.to_time_delta` and calendar-nv's arithmetic.
14. **`htstamp.from_time_delta` refuses a negative delta.** That is the
    only reason this direction can fail. A conversion that dropped the
    sign would turn "three hours ago" into "in three hours".
15. **Every refusal carries a byte offset, and `humantime.offset_of`
    reads it.** It answers `-1` for the refusals that have no position,
    so a caller printing a caret under the problem needs no `match`.
16. **`format_approx` rounds to the nearest whole count, and a length
    exactly halfway rounds down.** `2h 30m` is `"about 2 hours"`. A
    count that rounds up into the next unit is said in that unit, so
    3599 seconds is `"about 1 hour"`.
17. **RFC 3339 writes a year in four digits.** A `CivilDate` outside
    the years 0 to 9999 is formatted with more digits or a sign, and
    the parsers refuse that text.

## What is not included

- **A clock.** `htstamp.since` takes both instants. `std.time` and
  chrono-nv are where "now" comes from.
- **A word for "forever".** The grammar has none. A caller who wants one
  wants an optional around the length.
- **Time zones and offsets.** See rule 12.
- **Calendar arithmetic.** See rule 6.
- **A second grammar.** The units and their spellings are humantime's,
  and this package does not add to them. `format_approx` is the one
  addition, and it is a formatter rather than a grammar: its output is
  not meant to be read back.
- **Running on a microcontroller.** No such claim is made. Two of the
  three modules are about text, and `Str` does not link on the embedded
  target. `htdur` alone would build there, and a claim covering a third
  of the package would say less than it appeared to.

## Related packages

Four types in this project are a length of time, and they differ in
whether they carry a sign and in what they resolve to.

| You have | You want |
| --- | --- |
| a string a person wrote — `"30s"`, `"2h 30m"`, `"1y"` | **this package**. Nothing else parses the grammar |
| a length to print, for a log or for a person | **this package** — `format_duration` for a log, `format_approx` for a status line |
| a length measured from two clock readings | **`std.time`'s `Duration`** — signed, microsecond resolution, and what subtracting two monotonic readings answers |
| a length to add to or subtract from a date | **calendar-nv's `TimeDelta`** — signed, nanosecond, and what `arith.add_delta` takes |
| "the same day of next month" | **calendar-nv's `arith.add_months`** — calendar arithmetic, with the end-of-month rule written down |
| a timestamp with an offset in it | **calendar-nv's `iso8601.parse_rfc3339`**, which answers the offset separately |
| "now" | **`std.time`** or **chrono-nv** |

`htstamp.to_time_delta` and `htstamp.from_time_delta` are the two calls
that cross between this package's unsigned length and calendar-nv's
signed one.

- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is the civil
  date, the civil time and the arithmetic over them. This package's
  timestamp half reads and writes its types rather than defining its
  own, so a program has one definition of a leap year and not two.
- [chrono-nv](https://novo-lang.org/packages/chrono-nv) reads the clock
  and formats an instant in a zone. It is where "now" comes from.
- [cron-nv](https://novo-lang.org/packages/cron-nv) reads crontab
  expressions, which are the other kind of time a configuration file
  holds. It computes when a schedule fires next.
- [timer-nv](https://novo-lang.org/packages/timer-nv) holds many
  deadlines and answers which is due. A timeout parsed here is a
  deadline there.

## Tests

```bash
novo test tests/humantime_tests.nv   # 44 tests: every function, and the two properties
novo test tests/vectors_tests.nv     # 18 tests: humantime's own cases and RFC 3339 § 5.8
novo test tests/edges_tests.nv       # 10 tests: the arithmetic at the largest Int
bash tests/coverage.sh               # 468 of 468 lines of src/, merged over the three
```

The vectors are humantime's own: every unit spelling in the Rust
crate's unit test with the length it parses to, its combinations, its
overflows and refusals, and its exact formatter's output for each
unit. RFC 3339 section 5.8 supplies the timestamp examples, three of
which this profile refuses: two carry an offset other than `Z` and one
is a leap second.

The suite is built around the two properties the package claims. The
first is that `format_duration` round-trips: for each awkward length —
a whole year, a whole month, a second and a half, a single nanosecond —
the test formats and parses back and asserts it got the same length. The
second is that `format_approx` does not, and the test asserts that its
output is refused by `parse_duration`. The rest covers the grammar term
by term, each refusal with the offset it was found at, the truncation of
every conversion out of a length, the strict and lenient timestamp
parsers against RFC 3339 section 5.8's samples, and the two crossings to
calendar-nv's signed type in both directions.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
