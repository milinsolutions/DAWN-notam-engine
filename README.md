# DAWN — Deterministic Automation Workflow for NOTAM Processing

> *It is a hundred times better to skip or escalate than to generate a false rule with total confidence.*

*Automating a manual bottleneck in flight planning operations — with a safety model that refuses rather than guesses.*

---

## Overview

Flight planning systems require operational notices (NOTAMs) to be converted into machine-readable rules before route validation can account for them. This conversion is done by hand, one notice at a time, by trained analysts.

I spent five years doing this work. DAWN automates the part that can be automated safely, and — equally importantly — identifies precisely what cannot be, so human expertise is directed where it actually adds value.

**Validation to date:** 30 machine-generated rules were submitted through the normal quality-control process and reviewed by a QC analyst who was not told they were automated. All 30 were approved.

---

## Executive Summary

**The problem.** Converting NOTAMs into flight-planning rules is manual, repetitive, and slow — a complex notice with multiple coordinate sets can take several minutes each. It is also unforgiving: an incorrectly generated rule is syntactically valid, passes system validation, and enters production silently. Naive automation is worse than no automation.

**The solution.** A deterministic compiler that automates only the notices it can interpret with certainty, guarded by an explicit set of refusal conditions covering every category where correct interpretation is not deterministically achievable. Refused items are escalated with a stated reason rather than guessed at.

**Validation.** 30 compiler-generated rules were blind-reviewed by a QC analyst through the standard acceptance process, with no disclosure that they were machine-generated. **Approval rate: 30/30.**

**Indicative efficiency.** End-to-end processing runs at approximately 8–9 seconds per notice, against a manual equivalent of a minute or more for coordinate-heavy notices. A 30-item batch completes in 4–5 minutes versus 30+ minutes manually — an indicative reduction of around 85%.

**Status.** Hybrid supervised testing. Not yet released to the wider team.

> ### Disclaimer
>
> **Source code is not published.** This project integrates with a commercial flight-planning platform under a client NDA. The rule definition language, coordinate conventions, system interfaces, and client identity are proprietary and are not disclosed.
>
> This is a solution design and implementation case study. Input NOTAMs are shown in standard ICAO format, which is a public specification (ICAO Annex 15 / Doc 8126). Outputs are described semantically only.
>
> **Attribution:** The compiler, orchestration layer, and safety model are my own work. An earlier standalone HTML/JavaScript parser existed as prior art and informed my understanding of the output format — **I did not build it.** DAWN is a ground-up rewrite with a different safety model and a different execution target.

---

## The Problem

### Where this came from

I identified this bottleneck from inside the process. Five years of manually converting NOTAMs into flight-planning rules made three things obvious:

1. A significant proportion of the work is mechanical pattern-matching that does not require judgment.
2. A meaningful minority genuinely does require judgment — and those cases are not obviously distinguishable from the routine ones at a glance.
3. The two categories were being handled with identical effort, because nobody had ever formally separated them.

The opportunity was not "automate NOTAM processing." It was **define the boundary between the two categories precisely enough that the first can be automated without contaminating the second.**

### Why the stakes are asymmetric

- **An under-restrictive rule** may permit a route through active airspace.
- **An over-restrictive rule** blocks viable routes, causing unnecessary fuel burn, delays, and dispatcher overrides — which erodes trust in the automation and encourages the system to be bypassed entirely.
- **A semantically wrong but syntactically valid rule** enters the system silently. There is no error to catch, no exception raised, no validation failure. This is the failure mode that shaped the entire solution.

The design constraint is therefore not "maximise coverage." It is **never emit a rule that is confidently wrong.**

### Why not simply apply an LLM

Language models handle messy free text well and are poorly suited to a context where a plausible-sounding wrong answer is indistinguishable from a correct one at the point of use. A hallucinated coordinate is well-formed. A hallucinated altitude band is well-formed. Nothing downstream rejects either. In a regulated safety domain, confident wrongness is the specific risk to be engineered out.

---

## The Solution

A two-tier design that separates **certainty** from **coverage**.

**Tier 1 — Deterministic compiler.** Processes every notice class where interpretation is provably unambiguous. Returns either a rule it is confident in, or a structured refusal explaining precisely why it declined. There is no partial or best-effort output.

**Tier 2 — Domain language model (in development).** Handles the refused remainder — a bounded, well-characterised set of hard cases, with human review retained in the loop.

The boundary between them is a requirements artifact: nine explicit refusal conditions, each derived from a real notice where automation would have produced an operationally unsafe result.

### The refusal taxonomy

This is the substantive deliverable of the project. Each condition represents a documented failure mode with a defined escalation path.

| Condition | Business risk if automated | Disposition |
|---|---|---|
| **Solar-referenced schedules** (sunrise/sunset operating hours) | Requires solar position for specific coordinates and dates. Any assumed window is active at incorrect times — under-restrictive at one end of the validity period, over-restrictive at the other. | Escalate |
| **Radial / azimuth sector geometry** | Sectors defined by bearing and distance require geodesic computation. Approximation produces a shape that looks valid and is not. | Escalate |
| **Partial activation or closure** | The affected sub-area is described in prose with no coordinate boundary. Options are to over-restrict the full published area or to invent a boundary. Neither is defensible. | Escalate |
| **Conditional flight exceptions** | Restrictions carrying carve-outs for specific traffic categories are conditional logic, not geometry. Encoding as unconditional inverts the originator's intent. | Escalate |
| **Deactivation and cancellation notices** | Structurally near-identical to activation notices — same subject code, area, altitude band, validity period. Only the plain-language verb differs. Naive parsing generates an *active* restriction from a notice whose purpose is to remove one. | Escalate |
| **Undeclared schedules embedded in narrative text** | Time windows appearing in free text rather than the formal schedule field are unvalidated. Parsing them risks a rule active at the wrong times. | Escalate |
| **Local-time references** | Requires resolving applicable timezone and daylight-saving offset per date. Timezone arithmetic on assumption is a silent-error generator. | Escalate |
| **Instrument procedures and route structures** | Correct handling requires the underlying navigation database, not text parsing. | Escalate |
| **Unresolvable geometry** | Terminal condition — no valid shape or fallback parameters extractable with confidence. | Escalate |

Every refusal carries a machine-readable reason code. This enables correct Tier 2 routing, produces a full audit trail of declined items, and makes the refusal set itself measurable.

---

## Architecture

```
Raw notice (standard text or native XML)
        │
        ▼
Format adapter — normalises native XML into standard ICAO structure
        │
        ▼
Text normalisation — repairs coordinates fragmented across line breaks,
                     including non-Latin hemisphere character variants
        │
        ▼
Field extraction — structured parsing of ICAO item fields
        │
        ▼
╔═══════════════════════════════════════════╗
║        REFUSAL GATE — 9 conditions        ║ ──► Structured refusal
║   Any trigger halts rule generation       ║      + reason code
╚═══════════════════════════════════════════╝      → Tier 2 / human review
        │  (all conditions clear)
        ▼
Rule synthesis — geometry · altitude band · schedule
        │
        ▼
Validated rule output
```

**Schedule engine.** Groups same-day time windows, expands weekday ranges including week-boundary wraparounds, auto-resolves overnight periods to the following day, deduplicates repeated windows, and chronologically orders all periods using a symbol-agnostic sort resilient to formatting noise in source text.

**Altitude engine.** Multi-format parsing across flight levels, imperial, and metric units, with directional rounding (floor for lower bounds, ceiling for upper bounds) and an override that enforces the officially published ceiling where a computed conversion would otherwise fall below it.

**Orchestration layer.** Browser automation drives the live workflow: queue scanning with pre-filtering, session persistence, structured extraction, compiler invocation, rule injection, validation polling, and full audit logging of every decision.

---

## Technical Stack

- **Core language:** Python — the compiler uses the standard library only, end to end, with no third-party dependencies
- **Parsing:** Layered deterministic processing — format normalisation, structured field extraction, and rule synthesis, built on native pattern-matching rather than an external parsing framework
- **Automation:** Playwright (orchestration layer only, outside the compiler boundary)
- **Computation:** Native arithmetic for unit conversion and chronological ordering
- **Configuration:** Environment-based, with externalised interface selectors
- **Persistence:** Session-state caching and CSV audit logging

**On dependency choice:** the compiler deliberately carries no third-party dependencies. Every library in a safety-critical path is a component whose correctness must be trusted without being verified, and whose behaviour can change under version drift. Where the entire value of a system rests on never producing a confidently wrong output, minimising the trust surface is worth the cost of writing more logic by hand. Browser automation is the sole exception, and it sits outside the compiler boundary — it moves data in and out, but never interprets it.

---

## AI-Assisted Development

This project was developed using AI-assisted engineering practices to accelerate implementation, documentation, debugging, and iterative refinement.

AI served as an engineering accelerator throughout development, while architectural decisions, workflow design, operational validation, testing, and final implementation decisions remained under human review and responsibility.

This reflects my approach to modern engineering: combining domain expertise with AI to rapidly deliver practical, reliable operational solutions.

---

## Business Impact

**Validated**

| Metric | Result |
|---|---|
| Blind QC validation | 30/30 approved |
| Silent errors | None observed in supervised testing — architecture refuses rather than guesses |
| Documented refusal conditions | 9, empirically derived |

**Indicative** *(single-operator estimates from supervised testing — see caveat)*

| Metric | Result |
|---|---|
| Processing time | ~8–9 seconds per notice |
| Manual equivalent | 1 minute+ for coordinate-heavy notices |
| Batch reduction | ~85% (30 items: 4–5 min vs 30+ min) |

*Caveat, stated deliberately:* the indicative figures are single-operator estimates from supervised testing, not validated production metrics. NOTAM complexity varies substantially, and the figures are most representative for polygon and corridor-buffer notices carrying multiple coordinate sets — which are the slowest to process manually and therefore where automation returns most. Establishing a properly measured baseline across a representative notice mix is on the roadmap ahead of team rollout.

### Operational value beyond throughput

**Directed expertise.** The refusal taxonomy converts "review everything" into "review these specific cases, for these stated reasons." Analyst effort concentrates on genuinely ambiguous notices instead of spreading evenly across routine ones.

**Preserved trust.** Automation that occasionally emits wrong rules gets switched off permanently. A system with a hard, auditable boundary can be relied on within that boundary — which is what makes incremental scope expansion possible later.

**Auditability.** Every processed and refused item carries a timestamped decision record with cause. In a regulated environment this is not a nice-to-have.

**Transferable requirements artifact.** The refusal taxonomy documents nine operational risk categories in NOTAM interpretation. It has value independent of this implementation — as a training reference, a QC checklist, or a specification input for any future automation effort.

---

## Engineering Principles

> **The value of an automation system in a safety-critical domain is defined less by what it handles than by how reliably it identifies what it should not touch.**

**Accountability is binary.** In NOTAM processing, 99.99% accuracy is not a passing grade. A rule is either verifiably correct or it does not get generated. There is no acceptable rate of confident error — one silent wrong rule costs more than every skipped notice combined.

**Refuse over guess.** 95% coverage with silent errors in the remainder is unusable. Lower coverage with a hard boundary is deployable — because the boundary is where human attention gets directed.

**Fail loudly or not at all.** Every failure produces a structured, logged, machine-readable reason. Silent degradation is a defect, not a fallback.

**Widen under uncertainty.** Where ambiguity exists in altitude or extent, err toward the more restrictive interpretation. Over-restriction is a cost; under-restriction is a hazard.

**Trust intent over structure.** Where coded fields and plain language disagree, the language generally wins — it reflects what the originator meant, not how the notice was tagged.

**Minimise the trust surface.** No third-party dependencies in the safety-critical path.

**Encode experience, not just rules.** The refusal taxonomy is operational knowledge made machine-readable. It is the part of this project that could not have been built without the domain background.

---

## Current Status

**Stage:** Hybrid supervised testing. Every run is human-observed. Not yet released to the wider team.

| Component | Status |
|---|---|
| Deterministic compiler | Functional, in active refinement |
| Browser orchestrator | Functional, supervised runs only |
| Refusal taxonomy | 9 conditions, empirically derived, still expanding |
| Blind QC validation | 30/30 approved |
| Tier 2 domain SLM | In development |
| Automated regression suite | Planned |

### Known limitations

Stated deliberately — current boundaries, not oversights:

- **Refusal visibility is log-only.** Declined notices are logged with reasons, but there is no alerting or dedicated review queue. Acceptable under supervision; a prerequisite for unattended operation.
- **Test-scoped processing constraints remain active** in the orchestrator, on the pre-production removal checklist.
- **No systematic regression harness.** Validation is by observed behaviour on live traffic plus the blind QC round, not a fixed labelled corpus.
- **Text normalisation under independent review.** The coordinate repair layer applies sequential transformations to damaged input — the one area where a defect could produce a valid-looking but incorrect result that passes every refusal check.
- **The refusal taxonomy is retrospective by construction.** Each condition was added after encountering a real failure. Cases not yet seen are not yet covered. The rate of novel refusal categories per batch is tracked as the readiness signal for unattended operation.
- **Efficiency figures are indicative, not measured.** See Business Impact caveat.

---

## Future Roadmap

1. **Measured baseline** across a representative notice mix, replacing the current indicative estimate
2. **Labelled regression corpus** covering every refusal category, with negative controls that must *not* trigger a refusal
3. **Independent validation of the normalisation layer** against adversarial coordinate inputs
4. **Alerting and dedicated review queue** for refused items
5. **Expanded blind QC round** at larger sample size before team rollout
6. **Tier 2 SLM integration**
7. **Unattended operation** — gated on the above and on the novel-refusal rate having plateaued

---

## Lessons Learned

**The dangerous output is the well-formed one.** Early development optimised for parse coverage. The turning point was recognising that a notice *withdrawing* a restriction is structurally near-identical to one imposing it — same subject code, same area, same altitude band, same validity period, with only the plain-language verb distinguishing them. A structure-driven parser generates an active restriction from a notice whose entire purpose is to remove one. It passes every downstream validation. That single case reframed the project from a parsing problem into a refusal problem.

**Safety boundaries are discovered, not designed.** The refusal taxonomy could not have been written up front. Design-time reasoning produced perhaps half of the conditions; the rest required running against real operational traffic and investigating every divergence. Any requirements process for automating expert judgment should assume the same — the initial specification will be substantially incomplete, and the plan needs to accommodate that rather than treat it as failure.

**Blind validation is worth more than self-reported accuracy.** I could have measured the compiler against my own expectations and reported a number. Submitting output into the existing QC process without flagging its origin produced something far more useful: independent confirmation that the work is indistinguishable from manual output at the point of review. Where an acceptance process already exists, routing through it unannounced is the cheapest credible validation available.

**Domain experience is the differentiator, not the implementation.** Knowing that solar-referenced schedules need an ephemeris, that partial closures have no derivable sub-geometry, that subject codes are applied loosely in practice — none of that is available from the specification. It comes from having done the work by hand and knowing where the traps are. The code is the easy half.

**Measure the rate of new failure categories, not the failure count.** For an empirically derived safety model, readiness is not "how many cases pass." It is whether genuinely novel refusal categories are still appearing. If they are, the boundary is not yet characterised, regardless of how good the pass rate looks.

**Repair is safer than rejection — but only up to a point.** Normalising damaged input increased coverage substantially. It also created the project's subtlest risk: a chain of text transformations can produce output that is valid-looking and wrong, passing every refusal check. Coverage gained through transformation needs a higher standard of validation than coverage gained through parsing.

**Adoption is a trust problem before it is a technical one.** The reason the refusal model matters commercially is not correctness in the abstract — it is that one visible bad output would end the project. Designing for the failure mode that destroys confidence, rather than the one that is statistically most frequent, is what makes the system deployable at all.

---

*Source code withheld under client NDA. Solution design and reasoning documented here in full — available to discuss the approach in more depth.*

*Part of the **[Milin Solutions](https://milinsolutions.com)** portfolio.*