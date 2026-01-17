# Designing correct-by-default temporal types for Indent

**The ideal temporal type system separates machine time from human time, prevents timezone confusion through distinct types, and provides testable clock abstractions.** Based on analysis of java.time (the acknowledged gold standard), JavaScript's catastrophic Date API, and lessons from Python, Go, and Rust, Indent can achieve correct-by-default temporal handling through a carefully designed type hierarchy that makes invalid states unrepresentable. The core insight driving modern temporal design is that timestamps and calendar dates are fundamentally different concepts—conflating them causes bugs that cost millions of hours of developer time annually.

---

## The fundamental problem: two incompatible views of time

Time in computing exists on two distinct planes that must never be confused. **Machine time** represents points on a single, ever-increasing timeline measured in seconds since an epoch—it answers "when did this happen" without ambiguity. **Human time** represents what appears on a wall clock or calendar in a specific location—it contains gaps (DST spring forward), overlaps (DST fall back), and depends entirely on political decisions about timezone rules.

java.time's Stephen Colebourne identified this as Joda-Time's critical flaw: the `DateTime` class implemented `ReadableInstant`, confusing human-timeline representation with machine-timeline. JavaScript's `Date` commits the same sin—it's actually a UTC timestamp masquerading as a local datetime, causing the infamous bug where `new Date('2025-01-01')` produces December 31, 2024 in some timezones.

The solution is **type-level separation**. An `Instant` should never silently convert to a `LocalDateTime` or vice versa. Converting between them requires explicit timezone specification, making the programmer consciously acknowledge they're crossing conceptual boundaries.

---

## Recommended type hierarchy for Indent

### Core temporal types

| Type | Purpose | Example |
|------|---------|---------|
| `Instant` | Exact point on UTC timeline (nanoseconds since epoch) | Timestamps, logging, event ordering |
| `LocalDate` | Calendar date without timezone | Birthdays, holidays, "July 4th" |
| `LocalTime` | Wall-clock time without date | "9:00 AM", business hours |
| `LocalDateTime` | Calendar date + wall-clock time, no timezone | Recurring events "daily at 3pm" |
| `ZonedDateTime` | Full datetime with timezone (handles DST) | Meeting scheduling across locations |
| `OffsetDateTime` | Datetime with fixed UTC offset | Data parsed from "+02:00" without zone ID |

### Temporal amount types

The distinction between `Duration` and `Period` is **non-negotiable for correctness**. Adding "1 day" has two completely different meanings:

**Duration** measures elapsed time in seconds and nanoseconds. Adding `Duration.ofDays(1)` adds exactly **86,400 seconds**—which may result in a different wall-clock time the next day due to DST. When clocks spring forward, a 24-hour `Duration` lands at 7pm instead of 6pm.

**Period** measures calendar distance in years, months, and days. Adding `Period.ofDays(1)` means "same time tomorrow" regardless of DST—the wall-clock time remains **6pm**. A month is not a fixed duration; February has 28-29 days while March has 31.

```
// These produce DIFFERENT results across DST transitions!
zonedDateTime + Duration.ofDays(1)   // Exactly 24 hours later
zonedDateTime + Period.ofDays(1)     // Same local time tomorrow
```

### Timezone types

| Type | Purpose |
|------|---------|
| `TimeZone` | IANA timezone identifier ("America/New_York") with dynamic DST rules |
| `ZoneOffset` | Fixed offset from UTC ("+05:30") for storage/serialization |
| `ZoneRules` | Engine for computing transitions, offsets, gaps, and overlaps |

The critical distinction: `ZoneOffset` never changes, but `TimeZone` contains rules that vary with DST and political decisions. Storing a future event with only an offset loses the ability to recalculate when timezone rules update.

---

## Timezone handling strategy

### The "store UTC, display local" pattern—and when it fails

Conventional wisdom says to store everything as UTC and convert to local time for display. This works for **past events** where the conversion is deterministic. It fails catastrophically for **future events** because timezone rules change.

Consider scheduling "9am Paris, July 10, 2026" today. Converting to UTC using current rules (UTC+2 during summer) stores 7:00 UTC. If France abolishes DST before then (as nearly happened), the correct UTC becomes 8:00—but the system still shows 7:00 UTC, and the meeting appears at 8am local time.

**Recommended pattern** (from Jon Skeet): Store what the user said, not what you derived.

```
ScheduledEvent {
    localTime: LocalDateTime("2026-07-10T09:00"),  // User's intent
    timeZone: TimeZone("Europe/Paris"),            // Where
    cachedUtc: Instant,                            // For queries (recomputable)
    tzRulesVersion: "2025a"                        // For debugging
}
```

Periodically recompute `cachedUtc` when timezone database updates. The user's intent (`localTime` + `timeZone`) is the source of truth.

### Handling DST transitions explicitly

**Gap times** (spring forward) don't exist—2:30 AM vanishes when clocks jump from 2:00 to 3:00 AM. **Overlap times** (fall back) exist twice—1:30 AM occurs at both UTC-4 and UTC-5.

Indent should require **explicit disambiguation** when constructing times that might fall in transitions:

```indent
// Explicit resolution required for ambiguous times
ZonedDateTime.of(2024, 11, 3, 1, 30, 
    zone: "America/New_York",
    overlap: .earlier)  // Or .later, .error

// Explicit resolution for gap times  
ZonedDateTime.of(2024, 3, 10, 2, 30,
    zone: "America/New_York", 
    gap: .adjustForward)  // Or .adjustBackward, .error
```

java.time silently picks defaults; JavaScript Temporal offers `earlier`, `later`, `compatible`, and `reject` options. **Making ambiguity explicit prevents subtle bugs** where the system picks an offset the programmer didn't expect.

### Critical edge cases to handle

**Fractional-hour offsets** exist and must be supported: Nepal at +5:45, India at +5:30, Newfoundland at -3:30, and historically the Netherlands used +0:19:32. Offset representation must support at least minute precision.

**Days don't always have 24 hours.** DST transitions create 23-hour and 25-hour days. Code that calculates "tomorrow" by adding 86,400,000 milliseconds will eventually fail.

**Timezone rules change frequently.** The IANA database releases multiple updates yearly (2025a, 2025b, etc.) in response to government decisions, sometimes with only days of notice. Russia changed its base offset in 2014; Morocco pauses DST for Ramadan; Samoa skipped December 30, 2011 entirely when crossing the date line.

---

## API design patterns

### Consistent method naming (following java.time conventions)

| Prefix | Meaning | Example |
|--------|---------|---------|
| `of` | Factory from components | `LocalDate.of(2024, 1, 15)` |
| `from` | Factory with conversion | `LocalDate.from(temporal)` |
| `parse` | Factory from string | `LocalDate.parse("2024-01-15")` |
| `get` | Accessor | `date.getYear()` |
| `is` | Boolean check | `date.isLeapYear()` |
| `with` | Copy with changed field | `date.withMonth(3)` |
| `plus/minus` | Arithmetic | `date.plusDays(5)` |
| `to` | Convert to different type | `dateTime.toLocalDate()` |
| `at` | Combine with another | `date.atTime(LocalTime.NOON)` |

### Immutability as a core principle

All temporal types must be **immutable**. JavaScript's mutable `Date` causes bugs when objects are shared across functions—one caller modifies the date, affecting all other references. java.time, Temporal API, and Rust chrono all enforce immutability; every "modification" returns a new instance.

```indent
let date = LocalDate.of(2024, 1, 15)
let later = date.plusDays(7)  // Returns new instance
// date is unchanged: 2024-01-15
// later is: 2024-01-22
```

### Preventing accidental comparisons

JavaScript Temporal deliberately throws on `>` and `<` operators to prevent bugs from implicit string conversion ("2025-01-15" > "2025-03-20" evaluates incorrectly because it's string comparison). Indent should require explicit comparison:

```indent
// WRONG: Implicit comparison undefined
date1 > date2  // Compile error

// CORRECT: Explicit method
date1.isAfter(date2)
LocalDate.compare(date1, date2)  // Returns -1, 0, or 1
```

---

## Clock abstraction and testing

### The two clocks every system needs

**Wall clock** reports calendar time—it can jump forward (NTP sync), jump backward (manual correction), or skip hours (DST). Use it for timestamps and displaying time to users.

**Monotonic clock** counts time since system boot—it only moves forward, never adjusts, and is immune to NTP corrections. Use it for measuring elapsed time, timeouts, and performance profiling.

Go's `time.Time` elegantly embeds both readings in a single struct—`time.Now()` captures wall clock and monotonic values simultaneously. When calculating `time.Sub()`, Go automatically uses the monotonic component for accuracy. Cloudflare's 2017 DNS outage stemmed from using wall clock for duration measurement, which produced negative values after NTP adjustment.

### Testable clock injection

java.time's `Clock` abstraction enables deterministic testing by injecting controllable time sources:

```indent
trait Clock {
    fn now() -> Instant
    fn zone() -> TimeZone
}

// Built-in implementations
SystemClock             // Production: real system time
FixedClock(instant)     // Testing: frozen time
OffsetClock(base, +2h)  // Testing: shifted time  
SimulatedClock          // Testing: controllable advancement
```

**All `now()` methods should accept an optional clock parameter.** This allows business logic to be tested without time-dependent flakiness:

```indent
fn isOverdue(deadline: Instant, clock: Clock = SystemClock) -> Bool {
    Instant.now(clock).isAfter(deadline)
}

// Test with fixed clock
let testClock = FixedClock(Instant.parse("2024-06-01T00:00:00Z"))
assert(isOverdue(Instant.parse("2024-05-01T00:00:00Z"), testClock))
```

### Simulated time for async testing

Testing timeouts and scheduled tasks requires advancing fake time. Go's new `testing/synctest` package (Go 1.25) and libraries like `jonboulle/clockwork` provide patterns where tests can:

1. Create a fake clock at a specific instant
2. Block until goroutines/tasks are waiting on timers
3. Advance the clock to trigger those timers
4. Assert expected behavior occurred

This eliminates the "slow or flaky" dilemma—tests run in microseconds while simulating hours or days of elapsed time.

---

## Parsing and formatting

### ISO 8601 as the default standard

Indent should use **ISO 8601** for all default serialization:

- Date: `2024-01-15`
- Time: `14:30:00.123456789`
- DateTime: `2024-01-15T14:30:00`
- With offset: `2024-01-15T14:30:00+05:30`
- With timezone: `2024-01-15T14:30:00+05:30[Asia/Kolkata]`
- Duration: `PT1H30M5S`
- Period: `P1Y2M3D`

ISO 8601 is lexicographically sortable, unambiguous (no MM/DD vs DD/MM confusion), and machine-readable. The extended format with bracketed timezone ID (from RFC 9557) preserves full context for round-trip serialization.

### Strict parsing by default

**Default to strict parsing** that rejects invalid dates like `2024-02-30`. JavaScript's Date happily overflows invalid dates (February 30 becomes March 2), silently accepting garbage input. Lenient parsing should be opt-in with explicit intent:

```indent
LocalDate.parse("2024-02-30")                    // Error: invalid day
LocalDate.parse("2024-02-30", lenient: true)    // 2024-03-01 (overflow)
```

---

## Scheduling and calendar arithmetic

### Cron expressions and recurring events

For scheduling, support both machine-readable cron expressions and human-readable recurrence rules:

```indent
// Standard cron (minute hour day-of-month month day-of-week)
Schedule.cron("0 9 * * MON-FRI")  // 9am weekdays

// Structured recurrence
Schedule.weekly(DayOfWeek.Tuesday, LocalTime.of(15, 0))
Schedule.monthly(day: -1, time: LocalTime.of(9, 0))  // Last day of month
Schedule.nthWeekday(2, DayOfWeek.Monday)  // Second Monday
```

**Critical cron pitfall**: Jobs scheduled between 1-2am may skip or run twice during DST transitions. Scheduling systems should either document this behavior explicitly or schedule relative to a timezone-aware anchor.

### Business days calculation

Working day arithmetic requires a calendar of holidays:

```indent
trait BusinessCalendar {
    fn isBusinessDay(date: LocalDate) -> Bool
    fn addBusinessDays(date: LocalDate, n: Int) -> LocalDate
    fn businessDaysBetween(start: LocalDate, end: LocalDate) -> Int
}

// Built-in calendars
BusinessCalendar.USSettlement  // US federal holidays
BusinessCalendar.TARGET2       // EU bank holidays
BusinessCalendar.custom([...]) // User-provided holidays
```

---

## Comparison with existing approaches

### What java.time gets right

java.time established the gold standard through **complete type separation** (Instant vs LocalDateTime vs ZonedDateTime), **immutability**, **explicit DST handling** with `withEarlierOffsetAtOverlap()`/`withLaterOffsetAtOverlap()`, **testable Clock abstraction**, and **consistent API naming**. Its design philosophy—"clear, explicit, and expected"—should guide Indent.

### What JavaScript Date gets wrong (anti-patterns to avoid)

JavaScript's `Date` demonstrates every pitfall: **mutability** causing action-at-a-distance bugs, **zero-indexed months** (January = 0) causing off-by-one errors, **no timezone support** beyond local/UTC, **implementation-dependent parsing** producing different results across browsers, **no Duration/Period concepts** forcing manual millisecond arithmetic, and **single type for all use cases** conflating timestamps with calendar dates.

### Go's elegant simplicity—and its limitations

Go's single `time.Time` type eliminates naive/aware confusion by making **all times timezone-aware**. Its embedded monotonic clock reading is brilliant for duration measurement. However, it lacks separate `Date` type (forcing midnight timestamps for date-only values), lacks calendar-based `Period` (intentionally—"no definition for Day or larger to avoid confusion"), and loses timezone ID during serialization (only offset preserved).

### Rust chrono's type-level safety

Rust's chrono crate uses the type system to **prevent naive/aware mixing at compile time**: `DateTime<Utc>` and `DateTime<Local>` are different types that cannot be accidentally compared. Invalid operations return `Option` or `MappedLocalTime` enum, forcing explicit handling. This compile-time safety is ideal for a new language.

---

## Summary of recommendations for Indent

### Core design principles

1. **Separate machine time (Instant) from human time (LocalDateTime, ZonedDateTime)**
2. **Separate Duration (fixed seconds) from Period (calendar-based years/months/days)**
3. **Require explicit timezone for local-to-instant conversion**
4. **Require explicit disambiguation for DST gaps and overlaps**
5. **Make all temporal types immutable**
6. **Use 1-indexed months (January = 1)**
7. **Support nanosecond precision throughout**

### Type system checklist

- `Instant`: UTC nanoseconds since epoch
- `LocalDate`, `LocalTime`, `LocalDateTime`: timezone-unaware calendar types
- `ZonedDateTime`: full datetime with timezone rules
- `OffsetDateTime`: datetime with fixed offset (for interchange)
- `Duration`: seconds + nanoseconds
- `Period`: years + months + days
- `TimeZone`: IANA timezone with rules
- `ZoneOffset`: fixed UTC offset
- `Clock`: injectable time source abstraction

### Default behaviors

- **Strict parsing** (reject invalid dates)
- **ISO 8601 serialization**
- **Error on ambiguous DST times** (require explicit resolution)
- **System clock only via injection** (never implicit global access)

By following these patterns—learned from java.time's successes and JavaScript Date's failures—Indent can provide temporal types that make correct code easy to write and incorrect code difficult to compile.