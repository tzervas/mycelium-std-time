# Spec — `std.time` (clocks, durations, instants — three typed clock sources)

| Field | Value |
|---|---|
| **Status** | **Accepted** (2026-06-20, maintainer-ratified per DN-07 — guarantee matrix asserted in tests; open §7/§8 questions are design/scope calls, not contract violations; was *Implemented (Rust-first) — pending ratification* 2026-06-18, Draft/needs-design 2026-06-17) — the Rust-first code landed as `mycelium-std-time` (M-529, #170, Batch P5-B; guarantee matrix asserted in tests). The Mycelium-lang migration (M-502-gated) remains. |
| **Module / Ring** | `std.time` · Ring 2 (RFC-0016 §4.2) · Tier B (RFC-0016 §4.4) |
| **Tracks** | `M-529` (#170) — the Phase-5 task this spec delivers |
| **Scope** | Value-semantic **durations** and **instants**, and the **typed reading surface** for three clock sources: **MONOTONIC** (elapsed-time, never-backward), **WALL-CLOCK** (civil/UTC time, an entropy source), and the RFC-0008 **in-runtime LOGICAL clock** (a deterministic monotonic tick the runtime advances — the M-356 reclaim-windowing basis). Every clock *read* is a **declared effect** (C6). |
| **Boundary** | Out of scope: the LOGICAL clock's *advancement semantics* — owned by `std.runtime`/`colony` (M-521) over **RFC-0008** §4.7 / M-356; `time` exposes only the typed *reading* surface and FLAGs (never invents) the M-356 API. The general nondeterminism/entropy discipline a WALL-CLOCK read shares with RNG is `std.rand` (M-531) — both are entropy sources declared the same way (cross-referenced, not duplicated). Calendar/timezone arithmetic, formatting of instants, and parsing are `text`/`fmt` (M-524/M-533) consumers of these values. The OS facility a WALL-CLOCK/MONOTONIC read bottoms out in (`wild`/FFI, ADR-014) is the `std-sys` floor (RFC-0016 §8-Q6). |
| **Depends on** | RFC-0016 §4.1 (the contract C1–C6) / §4.4 (the `time` row: monotonic vs wall a typed distinction; logical clocks tie to RFC-0008); RFC-0008 (RT2/RT3 determinism rules; §4.7 / M-356 the in-runtime logical clock); RFC-0014 (declared, bounded effects — a clock read is a declared `time` effect; I4 budgets); RFC-0001 (the value model — `Value`/`Repr`/`Meta`, the guarantee lattice §4.3, ADR-003 metadata ≠ identity); RFC-0013 (the structured diagnostic record a failed read carries). |
| **Grounds on** | `std.core` (M-515) `Option`/`Result`/error values and the guarantee-lattice tags; the RFC-0008 R1 runtime primitives (`mycelium_interp::supervise`, M-356 — the logical clock as a deterministic monotonic counter) which `runtime` (M-521) will surface and `time` *reads against*. KC-3: above the kernel — no new trusted code; the typed surface is the honesty mechanism, not a new bound. |

---

## 1. Summary

`std.time` is the value-semantic time surface — immutable `Duration` and `Instant` values, duration arithmetic,
instant differences, and the typed *reading* of three clock sources. Its **honesty crux** is a **typed clock
distinction that is structurally never a silent swap**: MONOTONIC, WALL-CLOCK, and the RFC-0008 LOGICAL clock
are three *distinct, non-interconvertible* clock types, so a program cannot read wall-clock entropy where it
asked for elapsed time, and — load-bearing — **a deterministic-fragment program (RT2) cannot read WALL-CLOCK
entropy silently: a wall read is a `time`+`entropy` declared effect, and asking for it from the deterministic
fragment is a *type error surfaced explicitly* (RT2/RT3), never a silent allow.** Every clock read is a
**declared effect** (C6/RFC-0014): a WALL-CLOCK read is `Declared` and effectful exactly like an RNG draw —
never dressed up as exact or pure (VR-5). The LOGICAL clock is the *only* time source legible to the
deterministic fragment, because its advancement is deterministic and runtime-owned (M-356). Ring 2, Tier B; it
adds no trusted code (KC-3) — it consumes the RFC-0008 runtime's clock and `std.core`.

## 2. Scope & module boundary

- **In scope:**
  - **Value types:** `Duration` (a span — immutable, signed-or-unsigned per §7-Q4) and three *typed* instant
    kinds — `MonoInstant`, `WallInstant`, `LogicalInstant` — each tagged by its source clock so they do not
    silently interconvert.
  - **Duration arithmetic:** `add`/`sub`/`scale`, comparison, and conversion between explicit units — all
    fallible on overflow (`Err(Overflow)`, never a wrap/clamp — C1/G2).
  - **Instant differences:** `MonoInstant − MonoInstant → Duration` (within one clock source only); the
    *cross-source* difference is a **type error**, not a silent reinterpretation.
  - **Typed clock reads (effectful):** `mono_now() ! time`, `wall_now() ! {time, entropy}`,
    `logical_now() ! time` — each declares its effect on the signature (C6) and returns the *typed* instant of
    its own clock.
- **Out of scope (and who owns it):**
  - **The LOGICAL clock's advancement** (what a tick *means*, when it increments, the reclaim-window
    semantics) — `std.runtime`/`colony` (M-521) over **RFC-0008 §4.7 / M-356**. `time` reads the counter; it
    does not advance it and does not define the M-356 API (FLAGGED §7-Q1).
  - **The shared entropy/nondeterminism discipline** — `std.rand` (M-531). A WALL-CLOCK read *is* an entropy
    source; it is declared and named under the *same* RT3 reified-nondeterminism rule as an RNG seed. `time`
    cross-references `rand`'s discipline rather than restating it (§6).
  - **Calendar/timezone math, instant formatting, parsing** — `text`/`fmt` (M-524/M-533). `time` owns the
    monotonic/elapsed/logical primitives, not civil-calendar interpretation.
  - **The OS facility** a wall/mono read bottoms out in (`clock_gettime`-equivalent via `wild`/FFI, ADR-014) —
    the `std-sys` floor, RFC-0016 §8-Q6 (FLAGGED §7-Q3).
- **Ring & layering:** Ring 2 (RFC-0016 §4.2). `time` is **new library code written to the contract over Ring
  0/1**: the value types (`Duration`/`Instant`) are pure total functions over the value model; the clock reads
  *wrap* the RFC-0008 runtime's clock surface and the (FLAGGED) `wild` OS floor. It is a **consumer**, never a
  producer of new trusted clock code (KC-3).

## 3. Exported-op surface (design sketch)

A design sketch — enough to fix the surface and feed the guarantee matrix, not a committed grammar.
Value-semantic, immutable-by-default. Pure ops are total functions of their inputs; **fallible** ops return
`Result`; **effectful** ops declare their effect on the signature (`! time`, `! {time, entropy}` — the
RFC-0014 effect-row notation, FLAGGED §7-Q5). A read of the deterministic fragment that names a WALL clock is
rejected at type-check, not at run time (RT2/RT3).

```
// illustrative signatures (not a committed surface)

// --- value types: immutable, value-semantic (C4) ---
type Duration                       // a span; unit-explicit; arithmetic is checked
type MonoInstant                    // a point on the MONOTONIC clock     (never-backward, no civil meaning)
type WallInstant                    // a point on the WALL-CLOCK          (civil/UTC; an entropy source)
type LogicalInstant                 // a point on the RFC-0008 LOGICAL clock (deterministic runtime tick)

// --- duration arithmetic: pure, total-or-explicitly-fallible (C1/G2) ---
add(a: Duration, b: Duration) -> Result<Duration, TimeErr>   // Err(Overflow) — never wrap/clamp
sub(a: Duration, b: Duration) -> Result<Duration, TimeErr>   // Err(Overflow)
scale(d: Duration, k: Int)    -> Result<Duration, TimeErr>   // Err(Overflow)
cmp(a: Duration, b: Duration) -> Ordering                    // Exact, total
as_unit(d: Duration, u: Unit) -> Result<Duration, TimeErr>   // Err(Overflow) on a narrowing unit change

// --- instant difference: WITHIN one clock source only (cross-source is a TYPE error) ---
diff(a: MonoInstant, b: MonoInstant) -> Duration             // Exact, total (same-source)
diff(a: WallInstant, b: WallInstant) -> Result<Duration, TimeErr>  // Err(NonMonotonic) on a backward jump
//   diff(a: MonoInstant, b: WallInstant)  -> ⊥  COMPILE ERROR: cross-clock, never a silent reinterpret

// --- typed clock reads: DECLARED effects (C6). The effect is on the signature. ---
mono_now()    -> Result<MonoInstant,    TimeErr>  ! time              // monotonic; Err(ClockUnavailable)
wall_now()    -> Result<WallInstant,    TimeErr>  ! { time, entropy } // WALL: an entropy source (≡ rand)
logical_now() -> LogicalInstant                   ! time              // RFC-0008 logical tick (M-356-owned advance)

//   A read of a WALL clock from the deterministic fragment (RT2) does not type-check:
//   the `entropy` effect is not in the deterministic fragment's effect row (RT2/RT3) — surfaced explicitly.

enum TimeErr { Overflow, ClockUnavailable, NonMonotonic }   // every read/arith failure is explicit (I1/G2)
```

> **Note (design choice, FLAGGED §7-Q5):** the `! time` / `! {time, entropy}` effect-row syntax is illustrative
> — the concrete effect-declaration surface is RFC-0014's (its T3.4 effect-row growth path, RFC-0008 R8-Q2), not
> settled here. `time` commits only to *declaring the effect and naming entropy*, never to a syntax.

## 4. Guarantee matrix (the load-bearing deliverable — RFC-0016 §4.5)

Rows = exported ops. To be encoded as a checked table (the RFC-0003 §4 template) and asserted in tests once
code lands — never prose only. **Columns:** guarantee tag · fallibility (explicit error set) · declared effects
(time/entropy) · EXPLAIN-able?. **The clock-read rows are the crux:** a wall read is `Declared`+effectful, a
mono read is `Declared`+effectful but *not* entropy, and the logical read is `Declared`+`time`-only and is the
*sole* fragment-legible source.

| Op | Guarantee tag | Fallibility (explicit error set) | Declared effects (time / entropy) | EXPLAIN-able? |
|---|---|---|---|---|
| `Duration::add` | `Exact` | `Err(Overflow)` | none | yes (the refusal record) |
| `Duration::sub` | `Exact` | `Err(Overflow)` | none | yes (the refusal record) |
| `Duration::scale` | `Exact` | `Err(Overflow)` | none | yes (the refusal record) |
| `Duration::cmp` | `Exact` | total | none | n/a |
| `Duration::as_unit` | `Exact` | `Err(Overflow)` (narrowing) | none | yes (the refusal record) |
| `diff` (mono − mono) | `Exact` | total (same-source) | none | n/a |
| `diff` (wall − wall) | `Exact` | `Err(NonMonotonic)` on backward jump | none | yes (the diagnostic record) |
| `diff` (cross-source) | — (does not exist) | **compile-time type error** | none | n/a |
| **`mono_now`** (MONOTONIC read) | `Declared` | `Err(ClockUnavailable)` | **`time`** (not entropy) | yes (effect + read record) |
| **`wall_now`** (WALL-CLOCK read) | `Declared` | `Err(ClockUnavailable)` | **`{ time, entropy }`** | yes (effect + entropy-source record) |
| **`logical_now`** (LOGICAL read) | `Declared` | total (the counter is always readable) | **`time`** (deterministic, runtime-owned) | yes (effect + tick record) |

**Tag justification (VR-5 — downgrade rather than overclaim):**

- **`Exact` rows** are the **pure value operations**: duration arithmetic and same-source instant differences
  are exact integer-span computations over the value model — no accuracy semantics, no effect. They tag
  `Exact` because the *computation* is exact; their **fallibility is the never-silent guarantee** — an
  arithmetic that cannot be represented is `Err(Overflow)`, **never** a wrap-around or a saturating clamp
  (C1/G2). The `wall − wall` difference is exact arithmetic but **fallible on a non-monotonic backward jump**
  (a wall clock can step backward — NTP, leap, DST): that backward jump is `Err(NonMonotonic)`, an explicit,
  traceable outcome, **never a silent negative-or-zero span**.
- **The three clock-read rows tag `Declared`, never `Exact`/`Proven`/`Empirical` (VR-5, the crux of this
  module).** A clock read introduces a value the program did not compute — it is an *input from outside the
  pure fragment*. It is therefore `Declared` (asserted, surfaced) and **effectful**, exactly as RFC-0001's
  lattice and the §4.1 contract require; it is **never** dressed up as a pure exact value. The three differ
  only in *which* effects they declare:
  - **`wall_now` declares `{ time, entropy }`.** A wall-clock value is an **entropy source** — it carries
    real-world nondeterminism (it differs every read, it is unpredictable, it seeds). It is declared and named
    under the *same* RT3 reified-nondeterminism rule as an RNG draw (`std.rand`, M-531; §6). This is the
    load-bearing column: **a deterministic-fragment (RT2) program cannot read it silently** — the `entropy`
    effect is absent from the deterministic fragment's effect row, so the read is a **type error surfaced
    explicitly** (RT2/RT3), not a silent allow.
  - **`mono_now` declares `time` but *not* `entropy`.** The monotonic clock is still an effectful, ambient,
    nondeterministic-across-runs *input* (`time`), so it is `Declared`/effectful and out of the deterministic
    fragment — but it is not a civil-time entropy *source* in the RNG-seeding sense; it measures elapsed time.
    The typed distinction is what keeps "I want elapsed time" from silently becoming "I read wall entropy".
  - **`logical_now` declares `time` only, deterministically.** The RFC-0008 LOGICAL clock is a deterministic
    monotonic counter the runtime advances (M-356; RFC-0008 §4.7). Its read is the **only** time read legible
    to the deterministic fragment, *because* the value is reproducible under RT2 sequentialization (it is not
    real-world entropy). It still **declares the `time` effect** (it is a read of runtime state, not a pure
    constant) — declared, not silent. **Its advancement is `runtime`-owned, not `time`-owned (§7-Q1).**
- **EXPLAIN-able everywhere (C3).** Each clock read reifies *which* clock was read and *what effect* it
  declared (a read record); a failed read or a `NonMonotonic` backward jump carries an RFC-0013 structured
  diagnostic. No opaque ambient time source feeds a user-visible value.

## 5. §4.1 contract conformance (C1–C6)

- **C1 — never-silent (G2).** Every failure is an explicit, propagating outcome: duration overflow →
  `Err(Overflow)` (never wrap/clamp); a clock that cannot be read → `Err(ClockUnavailable)`; a wall-clock
  **backward jump** → `Err(NonMonotonic)` (never a silent zero/negative span). The **failure-semantics
  directive** holds: a clock-unavailable, a duration overflow, and a non-monotonic backward jump are each a
  robust, legible, traceable outcome (RFC-0013 diagnostic; recovered or re-propagated per I1), **never a
  silent zero/clamp/wrap**.
- **C2 — honest per-op tag (VR-5).** The crux. Pure arithmetic tags `Exact`; **every clock read tags
  `Declared` and is effectful** — a wall-clock read is *never* upgraded to a pure/exact value. The three clock
  sources are a **typed distinction**, so a wall read can never silently substitute for a monotonic or logical
  read; the tag is honest about *which* source and *what* effect.
- **C3 — no black boxes / EXPLAIN (SC-3/G11).** Each read reifies its clock identity + declared effect as an
  inspectable record; a refusal (`Overflow`/`ClockUnavailable`/`NonMonotonic`) carries a diagnostic naming the
  cause. The clock source is never an ambient hidden global — it is named on the call.
- **C4 — content-addressed, value-semantic (ADR-003 / RFC-0001).** `Duration` and the three `Instant` kinds
  are **immutable values**; arithmetic is a pure function of inputs (the *effect* lives only in the read, not
  in the value). Two equal-span durations are the same value; the source-clock tag and any provenance ride in
  `Meta` and are **not** identity (ADR-003).
- **C5 — above the small kernel (KC-3).** `time` adds no trusted code: the value types are total functions
  over the value model; the clock reads consume the *already-existing* RFC-0008 runtime clock (M-356, the
  logical counter) and a (FLAGGED §7-Q3) `wild` OS floor for wall/mono. No bound, no new trusted clock lives
  here.
- **C6 — declared, bounded effects (RFC-0014).** **A clock read is a `Declared` effect** on the signature
  (`mono_now ! time`, `wall_now ! {time, entropy}`, `logical_now ! time`) — never an undeclared side effect.
  Per RT2/RT3, the deterministic fragment's effect row excludes `entropy` (and ambient `time`), so a wall read
  from that fragment is a **type error**, not a runtime surprise. A blocking/deadline-bearing future timer, if
  any, would carry an explicit fuel-style budget (I4) — out of this read-only v0 (FLAGGED §7-Q2).

## 6. Grounding

- **The monotonic/wall/logical taxonomy + the "typed distinction, not a silent swap" crux:** RFC-0016 §4.4
  (the `time` row — "monotonic vs wall-clock is a *typed distinction*, not a silent swap; logical clocks tie
  to RFC-0008") and §4.1 (the C1–C6 contract).
- **The determinism rules — a deterministic-fragment program cannot read wall-clock entropy silently:** RFC-0008
  **RT2** (the deterministic fragment is the default; a program in it has one meaning) and **RT3**
  (nondeterminism is reified, named, and explained — every departure is an explicit construct). A wall read is
  exactly such a departure, so it is named (effect `entropy`) and excluded from the fragment.
- **The in-runtime LOGICAL clock (the M-356 reclaim-windowing basis):** RFC-0008 §4.7 / the Meta-changelog M-356
  entry — `reclaim` bounds restart intensity over a **logical clock** ("a deterministic monotonic counter the
  supervisor advances"), *not* wall-clock time (wall-clock is RFC-0008 **R8-Q3**, deferred). `time` exposes the
  typed *reading* surface of that counter; its **advancement** is `runtime`/`colony`-owned (M-521 / M-356) —
  **not invented here** (FLAGGED §7-Q1).
- **Reads are declared, bounded effects:** RFC-0014 §4 (effects on the signature; the never-undeclared rule),
  **I4** (a `time`-bearing effect carries a fuel-style clock/budget), **I1** (failures propagate, never
  swallowed). RFC-0016 **C6** names `time` explicitly as a declared effect.
- **The shared entropy discipline with `rand`:** RFC-0016 §4.4 (`rand` row — "nondeterminism is reified/named
  (RT3); a deterministic-fragment program cannot pull entropy silently"), RFC-0008 RT3. A WALL-CLOCK read is an
  entropy source and is declared/named the **same** way as an RNG draw — `std.rand` (M-531) owns the shared
  discipline; `time` cross-references it (§2 boundary).
- **The value model + honesty lattice + identity rule:** RFC-0001 (`Value`/`Repr`/`Meta`, the
  `Exact ⊐ Proven ⊐ Empirical ⊐ Declared` lattice §4.3), **VR-5** (honest tags — downgrade, never upgrade),
  **G2** (never-silent), **KC-3** (small kernel), **ADR-003** (metadata is not identity). The failure
  diagnostic shape: **RFC-0013** (structured, additive, traceable — I1).

## 7. Open questions (FLAGGED — resolve before ratification)

- **(Q1) The RFC-0008 LOGICAL-clock reading API — owned by M-356/M-521, not invented here.** `time` commits to
  exposing a *typed read* of the deterministic logical counter (`logical_now -> LogicalInstant`), but the
  concrete API — how a tick is read, whether it is per-colony/per-scope, the value shape, and the read's exact
  effect classification — is **`std.runtime`/`colony`'s (M-521) over RFC-0008 §4.7 / M-356**. This spec does
  **not** fabricate that surface. — *Disposition: defer to M-521/M-356; `time` re-exports/reads whatever the
  runtime publishes. Cross-phase, ties to RFC-0016 §8-Q4 (`runtime` sequencing, Phase-7-gated).*
- **(Q2) Timers / deadlines / sleeping — in scope, and budgeted how?** This v0 is the **read-only** clock +
  duration/instant surface. A blocking sleep, a deadline, or a timeout is a *future-bearing, time-budgeted*
  effect (RFC-0014 I4, RFC-0008 R8-Q3 wall-clock deadlines). — *Disposition: FLAGGED out of v0; if added, each
  is a declared `time` effect with an explicit fuel-style budget, never an unbounded block. Ties RFC-0008
  R8-Q3.*
- **(Q3) The `wild`/FFI floor for WALL/MONOTONIC reads.** A real wall/monotonic read bottoms out in an OS
  facility (`clock_gettime`-equivalent) via `wild`/FFI (ADR-014). If so, that block is inventoried (LR-9) and
  the C5 "no trusted code" claim narrows to "no *new* trusted code beyond the audited `std-sys` clock floor".
  The LOGICAL read needs **no** `wild` (it is pure runtime state). — *Disposition: FLAGGED; ties RFC-0016
  §8-Q6 (the `io`/`fs`/`time`/`rand` `wild` floor / `std-sys` split).*
- **(Q4) `Duration` representation + signedness + resolution.** Is `Duration` signed (so `sub` can yield a
  negative span) or unsigned-with-explicit-direction? What is the unit/resolution (nanoseconds? a rational
  tick?) and the overflow envelope that makes `Err(Overflow)` precise? — *Disposition: FLAGGED; the value-model
  representation choice (RFC-0001 `Repr`) is settled at design, but overflow must be explicit regardless (C1
  holds either way).*
- **(Q5) The effect-declaration surface for `time`/`entropy`.** The `! time` / `! {time, entropy}` notation is
  illustrative; the concrete effect-row surface is RFC-0014's (its T3.4 effect-row growth path, RFC-0008
  R8-Q2). The shared `entropy` effect tag must be the **same** token `rand` (M-531) declares — a cross-module
  naming agreement. — *Disposition: FLAGGED; defer the syntax to RFC-0014; co-name `entropy` with `rand`. Ties
  RFC-0016 §8-Q3 (ergonomics vs the contract: always-explicit effect at the call site).*

## Meta — changelog

- **2026-06-17 — Draft (needs-design).** Stands up the `std.time` (M-529, #170) module spec under RFC-0016
  (Draft): Ring 2 / Tier B clocks, value-semantic durations and instants. Fixes the scope + boundary (owns the
  typed *reading* surface and the `Duration`/`Instant` value types; defers the LOGICAL clock's *advancement* to
  `runtime`/M-521 over RFC-0008 §4.7 / M-356, and the shared entropy discipline to `rand`/M-531), the
  exported-op surface sketch (checked duration arithmetic; same-source-only instant differences; three typed,
  effectful clock reads), and — the load-bearing deliverable — the per-op **guarantee matrix** with the
  monotonic/wall/logical distinction made **typed and explicit**: pure arithmetic tags `Exact` (overflow and a
  wall-clock backward jump are explicit `TimeErr`, never wrap/clamp/silent-zero — C1/G2); **every clock read
  tags `Declared` and is effectful** — `wall_now` declares `{time, entropy}` (an entropy source, ≡ an RNG draw),
  `mono_now` declares `time`, `logical_now` declares `time` deterministically and is the sole fragment-legible
  source — so a deterministic-fragment program **cannot read wall-clock entropy silently** (a type error
  surfaced explicitly, RT2/RT3), never a value dressed up as pure (VR-5). §4.1 conformance (C1–C6) stated
  concretely (a clock read is a declared effect, C6; failures are robust/legible/traceable per RFC-0013, never
  silently swallowed). Grounding traces to RFC-0016 §4.1/§4.4, RFC-0008 (RT2/RT3, §4.7 / M-356 logical clock),
  RFC-0014 (declared/bounded effects, I1/I4), RFC-0001 (lattice/VR-5/G2/KC-3/ADR-003), RFC-0013. Five questions
  FLAGGED (the M-356 logical-clock read API owned by M-521/M-356 and **not invented here**; timers/deadlines
  out of v0; the `wild` OS floor for wall/mono; `Duration` representation/signedness; the effect-declaration
  surface + co-naming `entropy` with `rand`). No code; no kernel change (KC-3). Append-only.

- **2026-06-20 — Accepted (maintainer ratification, DN-07).** The maintainer ratified this Rust-first spec: the §4.5 guarantee matrix is asserted in tests, never-silent fallibility and honest per-op tags hold, and the open §7/§8 questions are design/scope calls, not contract violations. No guarantee tag was upgraded without a checked basis (VR-5). Status moves *Implemented (Rust-first) — pending ratification → Accepted*. Append-only; no kernel change (KC-3).
