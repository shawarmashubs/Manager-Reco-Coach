# Manager Recognition Coach

A recognition tool that comes to the manager, instead of waiting for the manager to come to it.

Recognition platforms let people send public thank-yous at work, usually with points attached. Companies buy them for engagement and retention. Recognition from a manager lands hardest. They see the work up close, and they have the most say in what comes next.

---

## TLDR; video intro
<div>
<a href="https://www.loom.com/share/d240c6260d10400cbcd3695acfdecc6a">
<p>Beacon: How to use - Watch Video</p>
</a>
<a href="https://www.loom.com/share/d240c6260d10400cbcd3695acfdecc6a">
<img width="300" alt="Beacon demo video" src="https://cdn.loom.com/sessions/thumbnails/d240c6260d10400cbcd3695acfdecc6a-9ebf21a1056e9a49-full-play.gif">
</a>
</div>

---

## Why this?

> A manager who lives in Teams or Slack all day, wants to recognize their team, but keeps forgetting to.

The platform isn't where they work. It's a separate tab, opened a few times a year, usually after an admin reminder. Sending a recognition currently requires noticing the moment, remembering the tool exists, leaving what you're doing, and writing from an empty box. It breaks at every step.

This prototype moves recognition into Teams, where the manager already is, and replaces an empty box with a guided draft. Every manager hits this friction, so removing it lifts all of them. 

It matters most for managers who've fallen behind. They don't show up in total volume, the metric these platforms report, and they're the hardest group to move. An employee whose manager never recognizes them is the retention risk the software was bought to prevent. Recognition from a manager also spreads, often prompting their reports and peers to start recognizing each other.

---

## What it does

A bot in the manager's chat client watches structured platform data, decides when there's a real reason to interrupt, and hands over a draft.

**Setup, once.** The manager picks which triggers to turn on: HRIS milestones, budget expiry, custom reminders, and gaps the Opportunity agent finds. Nothing fires that they didn't choose, and there's no cadence to configure, since pacing is handled downstream.

The same screen is the Settings tab afterwards, pre-filled with what's live. Turning everything off is a supported end state, kept distinct from never having set up: the first case says so plainly and leaves a route back, the second gets the welcome. A saved change takes effect on the current day rather than the next one, since the candidates were already detected and the opt-in filter was the only thing holding them.

Then three surfaces, each reviewable in under a minute:

**Nudge card.** One concrete line on why this surfaced. *"Alex hasn't been recognized in three months."* Accept or dismiss, nothing else.

**Input.** Who it's for, and what you want to say. Rough notes are enough.

**Draft.** Generated text, editable, core value and award pre-filled. Submit runs the checks.

```
signal crosses threshold
        ↓
  Opportunity agent   ── real gap, or normal variation?
        ↓  candidate
  Prioritization      ── deterministic: score, pace, rank, bundle
        ↓  top 1–3
  Nudge card          ── accept or dismiss
        ↓
  Input screen        ── who, and what you want to say
        ↓
  Style/Tone agent    ── which past recognitions fit this relationship
        ↓
  Draft generation    ── static AI: write it
        ↓
  Draft screen        ── edit, set core value, confirm
        ↓
  Submission gate     ── policy check (AI); auth, recipient
        ↓                validity, budget (deterministic)
  Published
```

---

## Running it

One static `index.html`. No build, no backend, no dependencies.

**Bring your own model.** This is the intended way to run it. Open the panel at the top, choose a provider, paste your key, and name the model you want. Four provider types are wired in:

| Provider | What you enter |
|---|---|
| Google Gemini | API key, model name |
| Anthropic (Claude) | API key, model name |
| OpenAI-compatible | Base URL, API key, model name |
| Google Vertex AI | Project ID, location, model name, access token |

**Use the OpenAI-compatible option for anything not listed.** That name refers to a request format, not a company — it is the de-facto standard most providers implement, so it covers hosted services and locally-run open-source models alike. Point it at the provider's base URL and it works. If you run a model on your own machine, download this project and open the file locally: a browser will not let a page served over HTTPS call an address on `localhost`.

**Model names are examples, not requirements.** Every model box is free text and ships with a name that worked at the time of writing. Model names change constantly, so if a call fails with "model not found," replace it with whatever your own account has access to. Nothing else needs changing.

**No key needed to look around.** With nothing connected, the whole flow still runs on a handful of pre-written answers so you can click through end to end. Each one is labelled `DEMO MODE` in the panel on the left rather than passed off as real output. This is a fallback for browsing, not the way to evaluate the agents — connect a model for that.

Keys live in your browser's local storage and post directly to whichever provider you picked. Nothing routes through a server, and no key is committed to this repo. Each provider's settings are stored separately, so filling in a second one does not erase the first.

**A run has a clock.** Nothing in the thread is placed by hand. Each simulated day runs detect, score, deliver, and the panel steps one day at a time so a cycle is watchable. State survives a reload in `localStorage`, so opt-ins, budget, delivered nudges and outcomes accumulate over a multi-day run. Reset returns to day one.

**Two manager profiles**, switched from the Teams avatar: one who's never opened setup, one four months in with everything on. Same engine, different history, which is what makes the acceptance-weight cold-start gate and the adaptive frequency cap visible.

The right panel exposes the machinery: the score breakdown behind each ranking, the voice examples feeding each draft, a run log, and the test harness. Showing *why* a nudge surfaced is the point of the prototype.

**The AI is graded, not assumed.** Two sets of tests run against whatever model you connect. One sends fixed inputs to each AI step on its own, with a known right answer, so a failure points at a component. The other drives the whole chain — accept, type, generate, edit, send, submission gate — because four steps can each be correct and still be wired together wrong.

Half the policy-check cases are messages it must *not* block. Without those, a component that refuses everything scores perfectly, and over-blocking is the failure a manager actually feels. Results are two numbers side by side, never one: what it correctly caught, and what it wrongly caught. Cases are run three times, since one pass from a model means it worked once, and a case that answers differently across runs is flagged as unreliable rather than counted either way. With no model connected the tests refuse to run instead of grading canned text.

Every model call, check and interaction is written to a permanent record, including you just clicking around. It survives a reload and survives Reset, and exports to a file.

`data/` holds the synthetic roster, recognition and nudge logs, scoring config, benchmarks, and eval cases. `policies/` holds the guidelines that draft generation and the policy check both read.

---

## What needed AI, and what didn't

Five components. Two are agents.

| Component | Type | Why |
|---|---|---|
| **Opportunity** | Agent | Judges whether a pattern in recognition history is a real gap or normal variation. Judgment under ambiguity. |
| **Style/Tone** | Agent | Picks which of a manager's past recognitions represent their voice for *this* recipient, blending across relationships on mixed drafts. Done by ear. |
| **Draft generation** | Static AI | Fixed rubric. Doesn't learn between calls. |
| **Policy check** | Static AI | Reads final text for hostility, protected-class references, confidential figures. "I won't miss working with you" clears every keyword filter. |
| **Prioritization** | Deterministic | Every input is already a number. |

Prioritization is the one worth defending. Its inputs are urgency, the manager's acceptance history for that trigger type, and impact, all quantities, and the design deliberately never hands it unstructured content. A model would do the same arithmetic slower and less reproducibly. It also has to stay explainable, since it decides whose moment gets a manager's attention.

It still adapts. Acceptance weight is a moving average per trigger type, skipped entirely until three real outcomes exist rather than seeded with an invented average. Cooldown is a decaying score penalty rather than a hard time block, so a mediocre nudge arriving first can't lock out a better one two days later. A single high override threshold lets an exceptional candidate through regardless.

---

## Principles

**Never reads message content.** No chat, no email. Every signal is structured platform data: recognition history, reporting lines, budget. Cut early, after review flagged that comms monitoring wouldn't clear infosec or legal. An earlier concept that read calendar density to infer manager bandwidth went for the same reason.

**Nothing sends itself.** The manager approves or edits every recognition. Coaching mode sharpens phrasing but is blocked from inventing a name, number, or outcome the manager didn't supply.

**No escalation.** A blocked recognition returns a reason and a rewrite to the manager. No admin, HR, or leadership ever sees a manager's nudge activity.

**Never a dead end.** Empty budget downgrades to non-monetary and offers a reminder. Expired auth retries once, then hands over a manual link. A failed policy check stops user from proceeding and asks them to remove such language without judgement.

---

## How I'd measure it

The north star is recognitions sent by opted-in managers who'd fallen behind, against a top-quartile benchmark frozen at launch, compared to a matched control group who didn't opt in. Opt-out rate is the guardrail. Top-band rejection rate is the calibration check.

Scope, metric definitions, and the event-level instrumentation plan are in **[MEASUREMENT.md](MEASUREMENT.md)**.

---

## How I tested it

Two models, 39 test cases, run against live Gemini. Both models scored identically on every case, and the cheaper one costs about 20 times less to run.

The full record is in **[TESTING.md](TESTING.md)**: what was tested, what passed, what failed and got fixed, the token and cost comparison, and what testing still cannot tell you.

---

## What's not built

- Two paths run end to end: a new manager on a custom reminder, an experienced one on a detected gap. Five design-plan cases are out of scope, including penalty override, multi-recipient partial failure, and thin-input branching.
- Auth is a pass/fail stub.
- Opportunity detects three known signal types. Open-ended pattern discovery was cut, since a hand-authored dataset can't hide patterns nobody planted.
- Persistence is `localStorage`, not a backend. A run accumulates across reloads (clock, opt-ins, budget, candidates, outcomes) so multi-day behaviour is observable, but it's one browser and Reset wipes it. "A manager who's used this for months" is still fixture data underneath.
- The scoring constants are starting guesses. They sit in readable config, and top-band rejection rate is how you find out they're wrong.

---

## Disclaimer

A personal project, built independently and also submitted as the capstone for an Agentic AI product course. Every person, company, recognition, and benchmark figure in this repository is synthetic.