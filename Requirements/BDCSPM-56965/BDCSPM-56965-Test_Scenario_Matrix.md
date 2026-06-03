# BDCSPM-56965 — Any Change Trigger Test Scenario Matrix

## Scope

This matrix targets the requirement intent for the BLSS `Any Change` trigger mode, with explicit coverage for:

- `delta` behavior for Numeric attributes
- `Min On Time` and `Min Off Time` behavior for change-of-state alarming
- `Enum` and `Numeric` attribute types
- `Unreachable` transition handling
- bounce / noisy-signal suppression

## Working Interpretation for Test Design

The following expectations are used to derive the scenarios below and should be treated as the baseline test oracle unless Product or Engineering explicitly refines the timing model:

1. `Enum / String / Datetime`: a change is detected when `new_value != old_value`.
2. `Numeric`: a change is detected only when `|new_value - old_value| > delta`. Equality to `delta` does not count as a change.
3. `Unreachable -> reachable` and `reachable -> Unreachable` are always treated as changes, regardless of `delta`.
4. First evaluation after startup or trigger enablement does not generate a false positive when no previous value exists.
5. `Min On Time` and `Min Off Time` remain in force for `Any Change`; the paired `alarm -> normal` behavior must not bypass debounce timing.

## Common Validation Points

Apply these checks to every scenario where they are relevant:

- Verify whether an `Alarm Active` record is generated or suppressed.
- Verify whether a paired `Alarm Return` record is generated or suppressed.
- Verify whether alarm history reflects debounce timing rather than an unconditional same-timestamp start/end pair when `Min On` or `Min Off` is configured.
- Verify whether `old value` and `new value` are captured correctly in alarm details.
- Verify whether `delta` is ignored only for `Unreachable` transitions, not for normal numeric changes.

## Suggested Test Data

| Attribute Type | Example Values | Notes |
|---|---|---|
| Enum | `OFF`, `ON`, `STANDBY`, `FAULT`, `Unreachable` | Useful for state-transition and noisy toggle scenarios |
| Numeric | `10.0`, `10.3`, `10.5`, `10.6`, `11.2` | Supports `< delta`, `= delta`, and `> delta` boundaries |
| Default Poll Interval | `1s` or other fixed known interval | Use a stable interval for timer verification |

## Scenario Matrix

| ID | Type | Initial Condition | Input Sequence / Setup | Delta | Min On | Min Off | Expected Result | Coverage Intent |
|---|---|---|---|---|---|---|---|---|
| AC-01 | Startup | No previous value exists | First sample received after service start or trigger enablement | N/A | 0 | 0 | No `Alarm Active`; no `Alarm Return` | Prevent false positive on first evaluation |
| AC-02 | Enum | Previous=`OFF` | Current=`OFF` | N/A | 0 | 0 | No alarm pair generated | Enum no-change negative case |
| AC-03 | Enum | Previous=`OFF` | Current=`ON` | N/A | 0 | 0 | One `Alarm Active` and one paired `Alarm Return` generated | Basic Enum positive path |
| AC-04 | Numeric | Previous=`10.0` | Current=`10.3` | `null` or `0` | 0 | 0 | Alarm pair generated because change is `> 0` | Numeric default-delta behavior |
| AC-05 | Numeric | Previous=`10.0` | Current=`10.5` | `0.5` | 0 | 0 | No alarm pair generated because change equals `delta` | Numeric exact-boundary negative case |
| AC-06 | Numeric | Previous=`10.0` | Current=`10.6` | `0.5` | 0 | 0 | Alarm pair generated because change is `> delta` | Numeric threshold-crossing positive case |
| AC-07 | Numeric | Previous=`10.0` | Sequence=`10.1 -> 10.2 -> 10.3 -> 10.4` | `0.5` | 0 | 0 | No alarm pair generated | Sub-delta noise suppression |
| AC-08 | Unreachable | Previous=`Unreachable` | Current=`10.0` | `999.0` | 0 | 0 | Alarm pair generated despite large `delta` | `Unreachable -> reachable` always counts as change |
| AC-09 | Unreachable | Previous=`10.0` | Current=`Unreachable` | `999.0` | 0 | 0 | Alarm pair generated despite large `delta` | `reachable -> Unreachable` always counts as change |
| AC-10 | Unreachable | Previous=`Unreachable` | Current=`Unreachable` | N/A | 0 | 0 | No alarm pair generated | No-change unreachable baseline |
| AC-11 | Enum + Min On | Previous=`OFF` | At `t0`, value changes to `ON` and remains stable for less than `Min On` | N/A | `10s` | 0 | No visible `Alarm Active`; no return pair should persist | `Min On` suppresses transient Enum change |
| AC-12 | Enum + Min On | Previous=`OFF` | At `t0`, value changes to `ON` and remains stable beyond `Min On` | N/A | `10s` | 0 | `Alarm Active` appears only after `Min On` expires; paired return must not bypass configured debounce semantics | Positive `Min On` gating for Enum |
| AC-13 | Enum + Min Off | Trigger already active from prior qualified change | Recovery toward normal lasts less than `Min Off` | N/A | 0 | `10s` | No final return-to-normal record yet; alarm remains effectively active | `Min Off` suppresses early recovery |
| AC-14 | Enum + Min Off | Trigger already active from prior qualified change | Recovery toward normal remains stable beyond `Min Off` | N/A | 0 | `10s` | Paired `Alarm Return` appears only after `Min Off` expires | Positive `Min Off` gating for Enum |
| AC-15 | Enum + Min On/Off | Previous=`OFF` | Sequence=`OFF -> ON` for `4s` then `ON -> OFF`; `Min On=10s` | N/A | `10s` | `10s` | No alarm history pair generated | Bounce suppression when change reverses before `Min On` |
| AC-16 | Numeric + Min On | Previous=`10.0` | Value jumps to `10.8` (`delta=0.5`) but remains there less than `Min On` | `0.5` | `10s` | 0 | No visible active alarm | Numeric over-delta spike suppressed by `Min On` |
| AC-17 | Numeric + Min On | Previous=`10.0` | Value jumps to `10.8` (`delta=0.5`) and stays stable beyond `Min On` | `0.5` | `10s` | 0 | `Alarm Active` generated after `Min On` expires | Numeric positive with both delta and debounce |
| AC-18 | Numeric + Min Off | Trigger already active from prior qualified change | Recovery path remains within normal / non-match zone for less than `Min Off` | `0.5` | 0 | `10s` | No return-to-normal record yet | Numeric recovery suppression |
| AC-19 | Numeric + Min On/Off | Previous=`10.0` | Over-delta change qualifies, active state is emitted, then normal state is restored and held past `Min Off` | `0.5` | `10s` | `10s` | Active is delayed by `Min On`; return is delayed by `Min Off`; history must not show zero-duration pair | End-to-end timer path for Numeric |
| AC-20 | Numeric Boundary + Timers | Previous=`10.0` | Value moves to `10.5` and stays beyond `Min On`; `delta=0.5` | `0.5` | `10s` | `10s` | No alarm pair generated because equality to `delta` is still not a change | Boundary must remain correct even with timers |
| AC-21 | Unreachable + Timers | Previous=`10.0` | Device becomes `Unreachable` and remains in that state beyond `Min On`, then later recovers and stays healthy beyond `Min Off` | `999.0` | `10s` | `10s` | `Unreachable` transition still generates alarm lifecycle, but active/return timing must honor `Min On/Off` | Combine unreachable semantics with debounce |
| AC-22 | Rapid Enum Bounce | Previous=`OFF` | Sequence=`OFF -> ON -> OFF -> ON -> OFF` with each state shorter than configured timers | N/A | `10s` | `10s` | No flapping alarm history; no duplicate short-lived pairs | High-risk noisy-signal regression |
| AC-23 | Rapid Numeric Bounce | Previous=`10.0` | Sequence crosses `delta` repeatedly in alternating directions, each state shorter than configured timers | `0.5` | `10s` | `10s` | No flapping alarm history; no duplicate short-lived pairs | Numeric noise regression |
| AC-24 | Mixed Semantics Regression | Previous=`Unreachable` | Sequence=`Unreachable -> 10.0 -> 10.4 -> 10.6 -> Unreachable` | `0.5` | `10s` | `10s` | Only transitions that satisfy the semantic rules should surface; `10.4` must not trigger, `10.6` may trigger if timer rules are met, unreachable transitions ignore `delta` but still honor timers | Combined regression across `delta`, timers, and unreachable |

## Coverage Map

| Dimension | Covered By |
|---|---|
| Enum positive / negative | `AC-02`, `AC-03`, `AC-11`, `AC-12`, `AC-13`, `AC-14`, `AC-15`, `AC-22` |
| Numeric positive / negative | `AC-04`, `AC-05`, `AC-06`, `AC-07`, `AC-16`, `AC-17`, `AC-18`, `AC-19`, `AC-20`, `AC-23`, `AC-24` |
| `delta` default behavior | `AC-04` |
| `delta` boundary `= delta` | `AC-05`, `AC-20` |
| `delta` strict positive `> delta` | `AC-06`, `AC-17`, `AC-19` |
| `Min On` | `AC-11`, `AC-12`, `AC-15`, `AC-16`, `AC-17`, `AC-21`, `AC-22`, `AC-23`, `AC-24` |
| `Min Off` | `AC-13`, `AC-14`, `AC-15`, `AC-18`, `AC-19`, `AC-21`, `AC-22`, `AC-23`, `AC-24` |
| `Unreachable` transitions | `AC-08`, `AC-09`, `AC-10`, `AC-21`, `AC-24` |
| Startup / no previous value | `AC-01` |
| Noisy / bouncing signal protection | `AC-07`, `AC-15`, `AC-22`, `AC-23` |

## Recommended Execution Order

1. Run `AC-01` through `AC-10` first to lock the change-detection oracle.
2. Run `AC-11` through `AC-21` next to validate debounce timing semantics.
3. Run `AC-22` through `AC-24` as regression and robustness coverage.

## Open Validation Items

- Confirm the exact expected history timestamps for `Any Change` when both `Min On` and `Min Off` are non-zero.
- Confirm whether timer validation should be asserted from raw change detection time, from effective active time, or from the next evaluation cycle boundary.
- Confirm whether `Any Change` with paired active/return is expected to materialize as one transient segment or as two explicit persisted transitions in every deployment mode.