# Measurement plan

How Manager Recognition Coach would be evaluated in a real deployment, and what has to be instrumented to do it. The prototype exercises part of this. See [What's not built](README.md#whats-not-built).

---

## Scope

**In scope.** The manager-side funnel, from opting into a trigger through a recognition being published. Every metric below derives from two append-only logs the feature already writes: the recognition history log and the nudge interaction history log.

**Out of scope for the first measurement period.** Recipient-side outcomes, whether the person recognized felt recognized, and any movement in engagement survey scores. Both need a separate instrument and a longer window than this feature's first read.

**Measurement window.** Twelve months, with cohorts assigned once at launch by recognition-frequency percentile and held fixed for the period. Reassignment is a later decision, since a cohort that moves mid-period can't be compared against itself.

---

## North star

**Manager Activation.** Average recognitions sent by opted-in managers from the bottom quartile, divided by a top-quartile benchmark frozen at launch, measured against a matched control group of bottom-quartile managers who didn't opt in.

The control group carries the weight here. A bottom-quartile cohort drifts upward on its own through regression to the mean, and that drift reads as lift if nothing is held against it.

The feature is built for every manager, and this metric deliberately tests the hardest cohort. Middle and top quartile trend is tracked alongside it to confirm the feature doesn't fatigue managers who were already active.

---

## Supporting metrics

| Metric | Definition | Why it's here |
|---|---|---|
| **Opt-in rate** | Eligible managers who enable the feature and turn on at least one trigger | The ceiling on everything else |
| **Opt-out rate** | Managers who disable the feature after enabling it | Guardrail. Volume can climb through fatigue, and this is how that shows up |
| **Funnel conversion** | shown → accepted → drafted → submitted, with drop-off by step | Localizes where the flow fails |
| **Repeat usage** | Managers with a send in two consecutive periods | Separates a habit from a novelty |
| **Draft edit distance** | Character distance between generated draft and sent text | Whether the writing is any good |
| **Opportunity acceptance** | Accepted over dismissed, split by signal type | Whether the agent's gaps are real gaps |
| **Coverage** | Direct reports recognized at least once in the period | The outcome a buyer actually cares about |
| **Budget utilization** | Allocated points spent | Recognition budget sitting unspent is the problem restated in dollars |

Opt-in and opt-out rates both depend on separating a manager who completed setup and later turned everything off from one who never set up at all. Both have zero triggers enabled, so counting managers with no triggers measures neither. Setup completion has to be recorded as its own state, which is what the prototype does.

**Downstream activation**, phase two. Recognition sent by the reports and peers of an activated manager, against the same for the control group's reports and peers. This tests the claim that a manager's recognition pulls others in behind it. It needs a longer window than the north star and a cleanly separated cohort, so it isn't part of the first read.

---

## Calibration

**Top-band rejection rate.** The share of nudges scored in the top decile that managers still dismiss.

Every nudge shown has already cleared the score threshold, so this is fully observable without deliberate exploration. A sustained rise means something the three scoring inputs can't see is driving rejections.

The response is staged, and the order matters:

1. Retune the weights and impact values, then re-measure. Most misses are fixable here.
2. Read the actual miss cases.
3. Only if retuning failed *and* the misses share a pattern none of the structured inputs represent, consider escalating that component to a real agent.

Reaching for a model because a number looks bad is the failure mode this architecture was built against.

---

## Instrumentation

Every metric above resolves to these events. Nothing else needs to be captured.

| Event | Fires when | Fields |
|---|---|---|
| `trigger_enabled` / `trigger_disabled` | Manager toggles a trigger type | `manager_id`, `trigger_type`, `timestamp` |
| `setup_completed` | Manager saves the setup screen, including a save with nothing selected | `manager_id`, `trigger_count`, `timestamp` |
| `feature_disabled` | Manager saves with every trigger off, having completed setup before | `manager_id`, `timestamp` |
| `nudge_shown` | A candidate clears Prioritization and reaches the manager | `manager_id`, `trigger_type`, `score`, `score_band`, `timestamp` |
| `nudge_accepted` / `nudge_dismissed` | Manager acts on the card | `manager_id`, `trigger_type`, `timestamp` |
| `draft_generated` | Draft generation returns | `manager_id`, `mode`, `recipient_count`, `clarifying_question_shown`, `timestamp` |
| `draft_abandoned` | Manager leaves the draft screen without submitting | `manager_id`, `trigger_type`, `timestamp` |
| `submission_blocked` | A gate check fails | `manager_id`, `reason` (policy, budget, auth, recipient), `timestamp` |
| `recognition_sent` | Publish succeeds, one row per recipient | `sender`, `recipient`, `core_value`, `monetary_flag`, `amount`, `relationship_type`, `edit_distance`, `timestamp` |

`trigger_type` carries one of eight distinct values, including three separate Opportunity signals (`opportunity_benchmark_gap`, `opportunity_recipient_neglect`, `opportunity_corevalue_narrowness`). Lumping them into one `opportunity` category would make per-signal acceptance invisible, and per-signal acceptance is how a weak signal type loses trust on its own.

`edit_distance` is computed at submit time and stored as a number. The generated draft itself is not retained.

### Constraints on what gets logged

These are product commitments, and they bound the instrumentation rather than the other way around.

- **No message content, from any source.** The only text stored is a published recognition, which is already public on the platform by design.
- **Abandoned drafts log the abandon, never the text.** A manager's half-written thought is not data.
- **Candidates Prioritization filtered out are not logged at all.** Only things a manager actually saw produce an outcome.
- **No per-manager reporting to anyone but that manager.** Admins, HR, and leadership see aggregates. Nobody sees an individual manager's nudge activity, which rules out a "least active managers" dashboard even though the north star is computed from exactly that population.

---

## What the prototype can and can't show

No prototype proves Manager Activation. Behavior change takes months of real use with real managers.

What it can demonstrate: the opt-in flow works end to end, the draft sounds like the manager who supposedly wrote it, and the calibration check reacts correctly to a deliberately bad recommendation fed through the eval runner.

---

## Deferred

Cut from the first measurement period to keep the set readable, and worth revisiting once the core metrics have a baseline: trigger configuration depth, time to first recognition, prioritization ranking accuracy as a standalone metric, personalization improvement over time, and per-trigger disable rate.
