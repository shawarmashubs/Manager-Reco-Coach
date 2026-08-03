# Manager Recognition Coach

An agentic prototype that catches the moments a manager should recognize someone, and drafts the recognition in the chat app they already live in.

Built as a capstone for an Agentic AI product course. Everything here runs on synthetic data against a fictional company. See [Disclaimer](#disclaimer).

---

## The problem

Employee recognition platforms have a distribution problem that has nothing to do with the product itself.

Recognition from a manager carries more weight than recognition from a peer. Managers know this, and most of them genuinely want to do it. But the recognition platform is not where they work. It's a separate tab they open a handful of times a year, usually after a program admin sends a reminder.

Here is what actually has to happen for one recognition to get sent today:

1. Someone on the team does something notable.
2. The manager independently notices it was significant.
3. The manager remembers the recognition platform exists.
4. The manager leaves whatever they were doing and logs in.
5. The manager re-orients to an interface they rarely use.
6. The manager writes something from scratch, with no starting point.
7. The manager submits.

The process usually breaks at step 2 or 3. Most recognition-worthy moments never reach step 7.

### Why it breaks

Four distinct failure points, and none of them is "the manager doesn't care":

- **No trigger in the workflow.** Nothing in the manager's actual day signals *this is a moment*. The moment passes silently.
- **Context-switching cost.** Recognizing someone means leaving the tool you're working in for one you're not.
- **Relearning friction.** Infrequent visits mean the interface is never familiar, and a blank text box is a real tax.
- **No self-sustaining loop.** Once the moment passes, the only thing that brings the manager back is someone nagging them. Recognition frequency is externally enforced rather than habitual, which caps how often it can happen even for managers who want to do it.

## Why I picked this problem

Recognition tools are usually measured on total volume sent, which flatters the wrong population. Volume climbs when already-engaged managers send more. That's a real number and a weak signal.

The interesting question is whether you can move the managers who *aren't* recognizing anyone. They are the hardest cohort to shift, and they're also where the actual business outcome sits, since an employee whose manager never recognizes them is the retention risk.

I also wanted a problem where the agentic framing had to earn itself. The lazy version of this feature is a notification panel: fire on a schedule, show a static template, done. That version doesn't need AI. The parts that do need judgment are narrow and identifiable, which made this a good test of where an agent belongs and where it's overhead. More on that below.

## Who it's for

> A manager at a company that uses a recognition platform, who lives in Teams or Slack all day and wants to recognize their team but keeps forgetting to.

Both halves are load-bearing. The chat surface is why the intervention point is a chat app. The platform relationship is why recognition has somewhere to land.

---

## How it works

The feature lives inside the manager's chat client as a bot. It watches structured platform data, decides when there's a real reason to interrupt, and hands the manager a draft they approve, edit, or throw away.

```
signal crosses threshold
        ↓
  Opportunity agent   ── is this a real gap, or normal variation?
        ↓  candidate
  Prioritization      ── deterministic: score, pace, rank, bundle
        ↓  top 1–3
  Nudge card          ── manager accepts or dismisses
        ↓  accept
  Input screen        ── who, and what do you want to say
        ↓
  Style/Tone agent    ── which of this manager's past recognitions
        ↓                fit this recipient's relationship
  Draft generation    ── static AI: write it
        ↓
  Draft screen        ── manager edits, sets core value, confirms
        ↓  submit
  Submission gate     ── policy check (AI) + auth, eligibility,
        ↓                budget (all deterministic)
  Published
```

### Two agents, two static AI calls, one deterministic layer

It's tempting to call every component an agent. Most of them aren't, and the distinction changes what gets built.

| Component | Type | Why |
|---|---|---|
| **Opportunity** | Agent | Judges whether a pattern in recognition history is a real gap or normal variation. Genuine judgment under ambiguity. |
| **Style/Tone** | Agent | Picks which of a manager's past recognitions best represent their voice for *this* recipient, blending across relationship types when a draft has mixed recipients. Interpretive, not a formula. |
| **Draft generation** | Static AI | Bounded to a fixed rubric. Doesn't learn between calls. |
| **Policy check** | Static AI | Reads the final text for hostility, protected-class references, confidential figures. Needs comprehension, not a blocklist, but adapts to nothing. |
| **Prioritization** | Deterministic | Every input is already a number. |

The Prioritization call is the one worth explaining. It ranks candidates on urgency, the manager's historical acceptance rate for that trigger type, and impact. All three are quantities, and the architecture deliberately never gives it unstructured content to reason over. An LLM handed those same fields would do the same arithmetic slower, at higher cost, less reproducibly, with no interpretive advantage. It also has to be explainable, because it arbitrates scarce attention between competing candidates, which is a fairness question.

The scoring layer does adapt, without being an agent:

- **Acceptance weight** is an exponential moving average per manager per trigger type, updated after every logged outcome. Below three real interactions for a given pair, the term is skipped entirely rather than seeded with an invented population average.
- **Cooldown is a decaying score penalty, not a hard time block.** A recently-fired trigger type is disadvantaged, never impossible. This fixes the failure where a mediocre candidate arriving first locks out a much better one that shows up two days later.
- **A single high override threshold** bypasses the penalty and the volume cap entirely, so a genuinely exceptional candidate always gets through.

### The design constraints that shaped it

**The agent never reads message content.** Not from chat, not from email, not from anywhere. This was a deliberate cut after an early review flagged that a company-wide comms-monitoring feature would never clear infosec or legal, regardless of how useful it was. Every signal comes from structured platform data: recognition history, HRIS-derived reporting lines, budget records. An earlier version of the nudge-timing logic proposed inferring manager bandwidth from calendar density. That's the closest thing here to real regulatory precedent against productivity monitoring, and it was cut for the same reason.

**Nothing sends itself.** A manager approves or edits every recognition. Nothing fires unless they opted into that trigger type. Coaching mode can sharpen phrasing but is hard-blocked from inventing a project name, number, or outcome the manager didn't supply.

**No escalation, ever.** A policy violation returns a reason and a suggested rewrite to the manager. It does not notify an admin, HR, or leadership. Nobody but the manager ever sees their nudge activity. This was a firm requirement, not a default.

**Never a dead end.** Budget empty auto-downgrades to non-monetary and offers a reminder for when budget refreshes. Auth expiry silently retries once, then hands over a link to send manually. A blocked policy check highlights exactly what to change. Every branch returns control to the manager with something they can do.

---

## Running the prototype

The whole thing is one static `index.html`. No build step, no backend, no dependencies.

**Stubbed mode (default).** Open `index.html` in a browser. Every AI call returns a canned response, explicitly labeled `[STUBBED]` in the console rather than silently faked. The full flow is walkable.

**Live mode.** The prototype calls Vertex AI directly from the browser. In the *Connect a Vertex AI project* panel at the top, supply:

| Field | Value |
|---|---|
| Project ID | Your `$GOOGLE_CLOUD_PROJECT` |
| Location | e.g. `global` |
| Model | e.g. `gemini-3.1-flash-lite` |
| Access token | Output of `gcloud auth print-access-token` |

Credentials are held in your own browser's `localStorage` and posted only to Google. Nothing is written into this file and nothing reaches a server. Access tokens expire roughly hourly; regenerate and re-save if calls start failing.

No API key is committed to this repository, and none should be.

### What to look at

The right-hand panel exposes the machinery rather than hiding it: the live score breakdown behind each ranking, the actual voice-profile examples feeding the draft, the run log, and an eval-case runner. The point of the prototype is showing *why* a nudge surfaced, not just that it did.

---

## Repository contents

```
index.html                        the entire prototype
data/
  roster.csv                      synthetic HRIS stand-in
  client_config.json              company settings, recognition rules, award amount
  core_values.csv                 core values catalog
  scoring_config.json             weights, caps, EMA decay, thresholds
  benchmark_figures.json          hardcoded aggregate benchmarks
  recognition_history.csv         append-only log; source for voice profiles
  nudge_interaction_history.csv   append-only log; source for acceptance weights
  budget_records.csv              per-manager balances
  custom_reminders.csv            manager-set and system-generated reminders
  manager_preferences.json        enabled trigger types
  eval_cases.csv                  test scenarios
policies/
  recognition_guidelines.md       writing guidance + compliance rules
```

`recognition_guidelines.md` is read by two callers: draft generation reads the writing sections, the submission-time policy check reads the compliance sections.

---

## How this would be measured

**North star: Manager Activation.** Average recognitions sent by opted-in bottom-quartile managers, divided by a top-quartile benchmark frozen at launch, measured against a matched control group of bottom-quartile managers who didn't opt in. The control group is the part that matters, since a bottom-quartile cohort will regress toward the mean on its own and look like lift.

**Required companions.** Opt-in rate, which is the ceiling on everything else. Middle and top quartile trend, to confirm the feature doesn't fatigue managers who were already engaged.

**The guardrail.** Opt-out rate. It's the check on the north star rising for the wrong reason.

**What the AI is doing.** Edit distance between the generated draft and what actually got sent. Percent of opportunity suggestions accepted versus dismissed.

**Top-band rejection rate**, the calibration metric: the share of nudges scored in the top decile that managers still dismiss. A sustained rise means something the formula's three inputs cannot see is driving rejections. The response is staged deliberately: retune weights first and re-measure, and only consider escalating a component to a real agent if retuning fails *and* reading the actual miss cases shows a shared pattern none of the structured inputs represent. Adding an LLM in response to a bad number, without reading the cases, is the over-engineering this architecture was built to avoid.

A prototype cannot prove Manager Activation. Moving real manager behavior takes months of real usage. What the prototype can demonstrate are the leading indicators: that the opt-in flow works, that the draft sounds like the manager wrote it, and that the calibration check reacts correctly to a deliberately bad recommendation.

---

## Known limitations

Stated plainly, because a prototype that hides its edges isn't useful to review.

- **Coverage.** Two entry paths are built end to end (a cold-start manager on a custom reminder, an established manager on an Opportunity-detected gap) plus one auth-failure case. Five eval cases from the design plan are out of scope for this build: decaying-penalty override, adaptive frequency-cap band, multi-recipient partial failure, relationship ineligibility, and thin-input branching.
- **Authentication is a mocked stub.** A deterministic pass/fail flag standing in for a real SSO call. It only models both-attempts-failed, not partial or intermittent failure.
- **Budget is an in-memory ledger**, reset by the *Reset synthetic data* button. It doesn't persist across a real page reload.
- **Opportunity is constrained to three known signal types** in this build (benchmark gap, recipient neglect, core-value narrowness). The open-ended novel-pattern reasoning was intentionally removed, since a small hand-authored synthetic dataset can't contain patterns nobody deliberately planted.
- **The connector is Vertex-native**, not the provider-agnostic endpoint originally specced. Running live requires your own Vertex project and a manually pasted, hourly-expiring token.
- **Nothing persists across sessions.** "A manager who has used this for months" is pre-seeded fixture data. Anything that happens during a run is session-scoped JavaScript state.
- **The scoring constants have no data behind them.** The weight split, the volume baseline, the penalty decay rate are starting points someone chose, not values derived from usage. They are inspectable and adjustable config rather than an opaque model output, and top-band rejection rate is the mechanism for finding out they're wrong.

---

## Disclaimer

A personal learning project, built for an Agentic AI product course. Not affiliated with, endorsed by, or representative of any employer's product, roadmap, or internal work.

All data in this repository is synthetic. The company, the employees, the recognition history, and the benchmark figures are fabricated for demonstration. No real customer, employee, or company data appears anywhere in this repository.
