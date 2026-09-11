# humantime-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

`"2h 30m"`, both ways.

- **Text into a length**: `"2h 30m"`, `"1.5s"`, `"3 days"`, `"250ms"`,
  `"1y 2months"` — terms are added, the number may carry a decimal
  point, and the space between a number and its unit is optional.
- **A length into text**, twice: `format_duration` gives the shortest
  spelling that means EXACTLY this length and round-trips, and
  `format_approx` gives the one a person reads — `"about 2 hours"`,
  `"just now"` — which deliberately does not.
- **RFC 3339 timestamps**, both ways, over calendar-nv's civil types,
  with **no clock anywhere in the package**.

## `m` is minutes and `M` is months

The one place the grammar is case-sensitive, and the one place a reader
gets it wrong.  It is humantime's rule and it is kept: a package that
quietly differed from the grammar it is named after would be worse than
one that carries an awkward rule and says so.

The units, in full: `ns`; `us` and `µs`; `ms`; `s`, `sec`, `secs`,
`second`, `seconds`; `m`, `min`, `mins`, `minute`, `minutes`; `h`, `hr`,
`hrs`, `hour`, `hours`; `d`, `day`, `days`; `w`, `week`, `weeks`;
**`M`**, `month`, `months`; `y`, `year`, `years`.

## A year here is 365.25 days, and a month is a twelfth of one

`1y` is 31557600 seconds — the Julian year — and `1M` is 2630016, which
is 30.4375 days.  They are **fixed lengths, not calendar arithmetic**:
adding `1month` to the 31st of January gives 30.4375 days later, not the
28th of February.

That is the right answer for "expire this cache entry in a month" and
the wrong answer for "bill this customer next month".  The table below
says which package to reach for.

## Three durations in this project, and which one you want

| you have | you want |
| --- | --- |
| a configuration string a person wrote — `"30s"`, `"2h 30m"`, `"1y"` | **this package**. Nothing else parses the grammar |
| a length to print for a person to read | **this package** — `format_duration` for a log, `format_approx` for a status line |
| a length measured from two clock readings | **`std.time`'s `Duration`** — signed, microsecond resolution, and what `Mono` subtraction already answers |
| a length that has to be added to or subtracted from a date | **calendar-nv's `TimeDelta`** — signed, nanosecond, and the type `arith.add_delta` takes |
| "the same day of next month", "this time next year" | **calendar-nv's `arith.add_months` / `add_years`** — calendar arithmetic, with the end-of-month rule written down |
| a timestamp with a zone offset in it | **calendar-nv's `iso8601.parse_rfc3339`**, which answers the offset as a second value; this package's RFC 3339 half is UTC only |
| "now" | **`std.time`** or **chrono-nv**. There is no clock in here |

**Why three and not one.** They differ in the two things a duration type
can differ in: whether it has a sign, and what its resolution is.
`std.time`'s is signed microseconds because it is what subtracting two
clock readings gives. calendar-nv's is signed nanoseconds because it is
what subtracting two civil instants gives. This package's is **unsigned**
nanoseconds, because the grammar it parses has no sign in it — `"2h 30m"`
is a length, and every consumer of one (a timeout, a retry backoff, a
cache lifetime, an uptime) is a length rather than a difference.

`htstamp.to_time_delta` and `htstamp.from_time_delta` are the two calls
that cross to calendar-nv's, and the second one can fail: a negative
`TimeDelta` has no length to become, and a conversion that dropped the
sign would turn "three hours ago" into "in three hours".

## The layer, and why

`core`. A scan over text the caller already holds and arithmetic over
two integers. No function declares an effect.

**There is no clock in it, and that is a design decision rather than an
omission.** `htstamp.since` takes BOTH instants; a caller who wants "now"
gets it from `std.time` or chrono-nv and passes it in. That is what keeps
the package `core` — and it is also what lets a test pin the time by
passing a different value, which a package that read the clock itself
could not offer.

**No device claim.** The two useful halves of this package are text, and
`Str` does not link at `@tier(embedded)`. A device that wants a length
of time wants the integer, and the host is where the configuration file
that spelled it was read. `htdur` on its own is `Int`-only and would
compile for a device, but a package whose probe covered a tenth of its
surface would be claiming something it does not mean.

## Adding it, and checking it

```bash
novo pkg add humantime-nv    # into your novo.toml
novo pkg build               # type- and effect-check the package
novo test --isolate tests/humantime_tests.nv
```

`novo test` is red today and that is the point of the release: all
forty-four assertions fail with `not implemented: <module>.<fn>`. They
turn green one at a time as bodies land.

## The one example that will work

```novo
use htdur
use humantime

fn main() [io]
    // A timeout out of a configuration file.
    match humantime.parse_duration("2h 30m")
        Ok((secs, nanos)) =>
            let d = htdur.new(secs, nanos)
            println("${htdur.as_millis(d)}")              // 9000000
            println(humantime.format_duration(d))         // 2h 30m
            println(humantime.format_approx(d))           // about 2 hours
        Err(e) => println(e.message())
```

## The load-bearing interface

`HtDuration`, in `htdur`:

```novo
pub @value
struct HtDuration
    secs: Int
    nanos: Int
```

Two integers in the caller's frame, no header and no allocation — a
retry policy that keeps four of these keeps sixty-four bytes. `nanos` is
`0 ..= 999_999_999` and `secs` is never negative, always: every
constructor and every operation normalises, so a caller may read the two
fields directly.

Everything else follows from two decisions:

- **The length has no sign.** That is the grammar, not a
  simplification — `"2h 30m"` has no sign in it, and humantime parses
  into Rust's unsigned `std::time::Duration`. `sub` therefore traps when
  the right side is longer, and `sat_sub` clamps to zero: a retry budget
  that went negative is a budget that was already spent, and a signed
  type would have carried the negative forward into the next sleep.
- **The two formatters are two functions, not one with a flag.**
  `format_duration` round-trips and `format_approx` does not
  (`"about"` and `"just now"` are not in the grammar). A caller who
  passed a variable would not be able to say which it meant, and a log
  line that sometimes round-trips is a log line nothing can parse.

Three consequences a reviewer should push on:

- **A parse answers a pair of integers, not a duration.** A `@value`
  struct may not be the payload of a `Result` — it is unboxed, and the
  position has no unboxed lowering — so `parse_duration` answers
  `Result<(Int, Int), HtError>` and `htdur.new(secs, nanos)` is the line
  that follows. Every fallible direction in the package has that shape.
- **A bare number is refused.** `"30"` is `HtNumberWithoutUnit`, not
  thirty seconds. A bare number in a configuration file is ambiguous
  between seconds and milliseconds and the two are three orders of
  magnitude apart, so the grammar makes the author say which.
- **Five RFC 3339 formatters, not one with a precision argument.** A log
  whose timestamps are sometimes six fractional digits and sometimes
  nine does not sort as text, which is the one property a timestamp
  format is chosen for. The precision is a decision made once.

## The reference implementations

`humantime` (Rust, MIT/Apache-2.0) for the grammar, the unit table, the
`m`/`M` rule, the two RFC 3339 parsers (strict and weak) and the five
formatters. `std::time::Duration`'s unsigned semantics are why this
package's length is unsigned.

RFC 3339 § 5.8 for the timestamp vectors, and calendar-nv for the civil
types the timestamp half reads and writes — which is a dependency rather
than a re-implementation, because a package that parsed its own dates
would have two definitions of a leap year in one program.

`format_approx` is this package's addition rather than a port:
humantime formats exactly, and the rounded phrasing a status line wants
is a separate job with a separate contract.

## Status

| function | implemented |
| --- | --- |
| `htdur.new`, `.zero`, `.max_value` | no |
| `htdur.from_nanos` … `.from_years` (ten units) | no |
| `htdur.minute_secs` … `.year_secs` (six lengths) | no |
| `htdur.as_secs`, `.subsec_nanos`, `.as_millis`, `.as_micros`, `.as_nanos`, `.is_zero` | no |
| `htdur.add`, `.sub`, `.sat_add`, `.sat_sub`, `.mul_int`, `.div_int` | no |
| `htdur.compare`, `.min`, `.max` | no |
| `humantime.parse_duration`, `.parse_duration_secs`, `.is_duration` | no |
| `humantime.format_duration`, `.format_duration_from`, `.format_approx`, `.largest_unit_name` | no |
| `humantime.offset_of`, `.HtError.message` | no |
| `htstamp.parse_rfc3339`, `.parse_rfc3339_weak`, `.is_rfc3339` | no |
| `htstamp.format_rfc3339` and the four fixed precisions | no |
| `htstamp.since`, `.add`, `.sub` | no |
| `htstamp.to_time_delta`, `.from_time_delta` | no |
