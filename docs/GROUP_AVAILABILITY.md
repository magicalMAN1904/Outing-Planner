# Engineering Case Study: Finding Time Together

[← Back to Outing Planner](../README.md)

## The problem

A useful group planner needs more than a list of calendar events. Each participant has recurring unavailable times, scheduled events, and potentially external calendar commitments. The system must determine *mutually free windows* without treating missing external calendar information as free time.

## The implemented approach

Outing Planner models availability as time intervals. The main group calculation gathers and combines participants' busy intervals, merges overlaps, and computes the free-time complement within the supported planning window. Combining everyone's busy time before inversion is equivalent to intersecting their individual free windows; an additional availability endpoint also performs explicit free-window intersection.

Illustrative example, independent of real user data:

| Participant | Free window |
| --- | --- |
| Person A | 09:00–15:00 |
| Person B | 11:00–17:00 |
| Shared result | **11:00–15:00** |

The availability calculation also accounts for existing personal events, scheduled outings, recurring unavailability, and cached Google Calendar FreeBusy intervals. Google busy intervals *remove* free time; they cannot create additional free time.

A shared day is classified as `full`, `partial`, or `unavailable` based on its free windows. The current backend uses a shared planning window of **08:00:00–23:59:59** and a hardcoded **240-minute** threshold in day classification. A day is full when a free window covers the entire planning window, partial when at least one continuous window reaches four hours, and otherwise unavailable—even if shorter free windows exist. The end-of-day sentinel is treated as midnight when measuring that threshold. These are current implementation constants, not user-configurable preferences.

## Why the design matters

**One interval-inversion path.** Both single-date and multi-day availability operations use a shared free-window inversion implementation. This avoids inconsistent handling of the planning boundary across endpoints.

**Explicit unknown state.** External availability is separately classified as `NOT_APPLICABLE`, `COVERED`, or `UNKNOWN`. When any contributing participant lacks trusted snapshot coverage for a date, the group result is `UNKNOWN`. In that case, the calculation returns the internal-only baseline windows rather than partially subtracting the known participants' Google intervals. The UI marks the result as not confirmed: those windows are not a guarantee that external calendars are free. The external-availability status is day-level rather than participant-attributed.

**No live Google requests during comparisons.** An explicit refresh populates cached FreeBusy data. Availability requests read that cache, reducing provider coupling during the calculation.

**Privacy by design.** The shared availability API does not expose another participant's Google event titles, descriptions, or event-level busy cards. Friends' external calendar data influences calculated availability only.

## Trade-offs and remaining work

The fixed planning window and classification threshold simplify consistent calculations but limit customization. External FreeBusy refresh is explicit, and a known error-recovery issue may leave the cache in an unknown state when a selected calendar cannot be queried. A documented sub-minute edge case remains unresolved, with a disabled regression test recording the desired behavior. These are tracked limitations, not solved features.

## Engineering takeaway

This subsystem illustrates how an apparently simple UI question—“When can everyone meet?”—becomes a problem in interval processing, boundary consistency, external-data uncertainty, and privacy-preserving API design.

*Verified against the development main branch on October 20, 2026. This public case study is a conceptual explanation, not a source-code mirror.*
