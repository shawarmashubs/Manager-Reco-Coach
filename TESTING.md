# How this was tested, and what it cost

Manager Recognition Coach sends a manager a nudge, helps them write a recognition, and checks it before it publishes. Four of those steps are AI. This is the record of testing those four steps, and everything around them.

Last full run: 4 August 2026, against Google Gemini.

---

## The short version

- Two AI models were tested. **They produced the same quality. One costs 20 times more.**
- The cheap model costs about **$15 per 10,000 recognitions**. The expensive one costs about **$297**.
- All AI checks pass on both. All seven end-to-end paths complete on both.
- Testing found real bugs, including one where the AI was inventing facts in a mode that forbids it. Every one is fixed.
- One thing still needs a person: whether a draft reads well. No test can answer that.

---

## Model comparison

This is the most useful result, so it comes first.

Both models ran the same 32 graded cases, the same 7 end-to-end paths, and the same judgment sweep. 107 calls each.

### Quality

| Test | Gemini Flash (latest) | Gemini 3.1 Flash-Lite |
|---|---|---|
| Policy check, blocked what it should | 6 of 6 | 6 of 6 |
| Policy check, allowed what it should | 5 of 5 | 5 of 5 |
| Opportunity, found real gaps | 2 of 2 | 2 of 2 |
| Opportunity, ignored fake gaps | 2 of 2 | 2 of 2 |
| Draft, followed its rules | 10 of 10 | 10 of 10 |
| Style and tone, followed its rules | 5 of 5 | 5 of 5 |
| End-to-end paths completed | 7 of 7 | 7 of 7 |

No difference.

### Judgment

The Opportunity agent decides whether someone genuinely deserves recognition. There is no right answer to score against, so both models were run over the same five people and their decisions compared by hand.

| Person | Flash | Flash-Lite |
|---|---|---|
| Never recognised, here five years | Flag | Flag |
| Never recognised, skip-level report | Flag | Flag |
| Last recognised six months ago | Flag | Flag |
| Recognised three weeks ago, five times | No | No |
| Hired last month, never recognised | Flag | Flag |

Identical. Both correctly declined the person who was recognised recently, which is the one that separates a careful agent from an eager one.

### Cost

| | Flash (latest) | Flash-Lite |
|---|---|---|
| Calls | 107 | 107 |
| Input tokens | 82,959 | 82,959 |
| Output tokens | 70,435 | 7,696 |
| **Total tokens** | **153,394** | **90,655** |
| Cost of this test run | $0.65 | $0.03 |
| Cost per recognition | $0.0297 | $0.0015 |
| **Cost per 10,000 recognitions** | **$296.68** | **$14.67** |

Input was identical. The whole gap is output. Flash spends nine times more tokens thinking, and arrives at the same answers.

A rough upper bound, charging every token at the higher output rate: $1.15 for Flash, $0.14 for Flash-Lite. The real figures above split input and output properly.

Prices are Google's published paid-tier rates read on 4 August 2026. Flash-Lite is $0.25 per million input and $1.50 output. Flash is $1.50 and $7.50. There is no pricing API, so these need re-checking before being quoted anywhere that matters.

### What this tells you

The prompts are doing the work, not the model. That matters more than the money.

The clearest evidence is the Clean-up bug below. It was fixed by rewriting an instruction, and the fix holds on the cheap model. If the expensive model had been covering for a weak prompt, Flash-Lite would have failed there.

**Recommendation: use Gemini 3.1 Flash-Lite.** Same results, 20 times cheaper.

One caution. The model name should be pinned, not left as an alias like `gemini-flash-latest`. An alias never goes out of date, and it also never tells you which model you are paying for. That alias is how this project ended up on the most expensive Flash by accident.

---

## What was tested

### Four AI steps, one at a time

34 cases. Each has a fixed input and a right answer known in advance. Each runs three times, because a model can get something right once by luck.

- **Opportunity, 6 cases.** Does it spot a real recognition gap, and does it leave alone someone who was recognised recently? Does it ever invent a benchmark figure? Does an instruction hidden in the data change its answer?
- **Style and tone, 5 cases.** With one recipient the manager has history with and one they don't, does it use real examples for the first and a generic fallback for the second? Does it ever invent a past recognition?
- **Draft, 12 cases.** Does Coaching mode behave differently from Clean-up mode? Does it invent numbers? Does it name people who aren't receiving the recognition? Is the length in proportion to what the manager gave it, four sentences on a rich input, one on a bare thank-you, and one per relationship group when several people are named at once?
- **Policy check, 11 cases.** Six messages it must block and five it must not.

Two of the Draft cases were added on 2026-08-06, both about length, and the second exists because the first was not enough.

Length had only ever been checked at one end: a rich input had to stay under four sentences, and nothing checked the other end, which is where it broke. Given "thanks tobias" the draft came back at two sentences and 22 words, inventing an ongoing pattern of hard work and support for the team out of a bare thank-you.

The fix for that held for one recipient and fell over on a list of them. Given "thanks Elena, Diego, and Hannah, Oscar, Mei, and all" the draft produced 38 words, invented three things nobody had said, sorted the recipients into groups matching no actual relationship, and addressed three people as "both". The count of names was being read as a count of things worth saying. The second case pins that down: five names across two relationship groups, nothing said about any of them, and it must come back as two short sentences with everyone still named.

**Half the policy cases are messages it must allow.** Without those, a checker that blocks everything scores full marks. Over-blocking is the failure a manager actually feels, because a warm harmless message getting refused is what makes someone stop using the tool.

### The whole flow, end to end

14 paths. Each one accepts a nudge, types, generates, edits, sends, and goes through the submission gate. Driven through the same code a real click drives.

- A clean send publishes with a core value recorded.
- A manager approves a clean draft, then types something hostile into it before sending. **The gate must catch the edit, not the draft it was handed.** This is only testable end to end.
- That blocked recognition is then corrected and sent, with both attempts on record.
- Budget running out downgrades the recognition instead of blocking it.
- A failed sign-in keeps the draft.
- One invalid recipient blocks only that person.
- Two invalid recipients are reported together, not one per attempt.
- A departed recipient comes off the list and the recognition still reaches whoever is left.
- Hostile wording and a departed recipient on the same recognition arrive on one screen, not one per Send press.
- A manager who types only a name gets a question, and their typing stays in the box.
- A recognition sent to a direct report and to the manager's own boss uses a different tone for each.

Added 2026-08-06, after clicking around found four defects nothing here covered:

- **Hostile wording typed into the first box.** The two existing policy paths both put the bad text in further downstream, so neither drove what a person actually does. Doing that handed the manager their own sentence back as a suggested recognition with a thank-you in front of it, ready to send.
- **Hostile wording mixed with something real.** The opposite failure. Refusing the whole message loses work the manager did; keeping it publishes the part that has to go. The draft keeps one, drops the other, and reaches Send on its own.
- **The rewrite pencil pressed twice with nothing changed.** The previous draft was being fed back in as though the manager had written it, so each press rewrote a rewrite rather than the manager's own sentence.

The two-model comparison above was run against the suite as it stood before these three were added.

### Everything else

- **15 live checks** run whenever anyone uses the prototype, not only during a test.
- **7 system rules** watch for broken promises, for example that the nudge on screen is the highest-ranked one waiting.
- **139 automated checks** across four scripts.
- **A soak test** that plays 20 to 30 simulated days for both manager profiles, taking every action a user can take, with the policy gate rigged to reject every third submission.
- **A permanent log** of every model call, check and interaction, including someone just clicking around. It survives a page reload and survives a reset, so wiping the demo data does not destroy the evidence.

---

## Bugs found

Testing found real problems. Almost all were the same shape: a value worked out correctly in one place, then read wrong somewhere else.

### The AI was inventing facts

Clean-up mode is meant to tighten the manager's own words and add nothing. It was adding things on most runs. Given "Sam shipped the auth migration a day early and wrote the runbook", it produced "making sure the rollout was smooth end-to-end" and "setting the team up for success". The manager said neither.

Fixed by rewriting that instruction with a worked example naming the exact phrases that are not allowed. Now clean on every run, on both models.

### There was no test for it

This is the more important half. The test set only checked things that are easy to count, like numbers and stray names. The question "did it add meaning that wasn't there" was handed to a second AI to judge. **That judge caught it once in three runs.**

Replaced with a check that counts. It strips out small words, strips anything the manager already wrote, strips names, and whatever is left is content the model introduced. Three or more new words fails. Blunt, but it gives the same answer every time, which the judge did not.

### Other AI problems found

- The Draft instructions never stated a rule the app was already enforcing, about not naming people who aren't receiving the recognition. The AI was being marked down for a rule nobody gave it.
- On the cheaper model, a short input like "Diego did a great job this week" made the AI ask a question instead of writing. That is the friction this product exists to remove. Fixed by rewriting the rule as one test: if the manager said anything about what the person did, write; if they only gave a name, ask.

### Product bugs found earlier

- The chat showed the wrong nudge. The ranking picked the best one and the screen displayed the third best.
- Every sent recognition saved a blank core value, which is a field one of the agents reads.
- A nudge rewrote its own wording after being sent, so the reason it gave stopped being true.
- Dismissing a nudge freed up a slot, so dismissing produced more nudges instead of fewer.
- Opt-in settings were not saving. Turning everything off looked like never having set up.
- Skip-level reports were labelled "your unknown".
- Three of the app's own status rules were wrong and flagged correct behaviour as broken.
- The deployed page showed a setup form and nothing else, with 17 nudges waiting behind a screen nobody could get past.

### Found by clicking around, 2026-08-06

None of these were caught by the test suite, because every path it drove put hostile text in *after* the draft step. A person typing it into the first box found all of them in about ten minutes.

- **The draft handed the manager's own abuse back to them.** Typing "get lost, you're too old" produced a suggested recognition reading `Thank you for this. "Get lost Hannah. You're too old."`, ready to send. Nothing checks the manager's wording before drafting, by design, since the policy check is meant to be the only gate. But the draft agent's instructions never mentioned hostile input at all, so it had no reason to refuse. It now reads the same C1 to C4 rules as the submission gate, drafts only from whatever part of the message is publishable, and asks what the person actually did when none of it is. Deliberately not a generic "thank you for your work": that is sendable, so it would hand the manager a recognition to send to the person they just wrote that about.
- **The draft compounded on every rewrite press.** Pressing the pencil replaced the manager's typed input with the current draft, whether or not they had edited it. Five presses produced five stacked copies of the same opening phrase. The swap now happens only when the text actually changed, so an unchanged press asks for a fresh attempt at the same sentence. The Clean-up drift check had been silently measuring against this moving target, so it passed on exactly the runs that were breaking.
- **The submission gate read a truncated message.** With no model connected, the check pulled the text out of its prompt with a pattern that stopped at the first double quote. Any recognition containing a quotation mark was cut off before the check saw it. The reported message was `Thank you for this. "Get lost..."`, so the check read four harmless words and passed it.
- **The gate's stand-in matched one literal sentence.** It tested for `not miss working with you` and passed everything else, which is the same sentence one fixture uses. That case passed by construction while plain abuse published. Replaced with a rule table organised by the same four compliance sections, shared with the draft step so the two cannot disagree about what is publishable.
- **The quote marks were the app's own.** The prompt wraps the manager's text in quotes as formatting, and the stand-in was capturing the wrapper along with the text. That is why the compounding showed nested quotes piling up.
- **The watchdog on the policy check had a hole.** A separate list of seven phrases watches what the gate lets through. "Get lost" and "you're too old" were on neither, so when that message published the watchdog recognised nothing and the panel row stayed grey, which reads as "not tested yet". It now reads the same shared rule table, so there is one list instead of two.
- **Test runs were leaking into the manager's real record.** Publishing saves to browser storage immediately. The end-to-end harness undid each scenario in memory and never saved, so on the next page load those test recognitions came back as part of the manager's genuine history. It also switched trigger types on for a manager and never switched them back. Both are now included in the rollback, and the rollback writes to storage.
- **The run log recorded the agents and almost nothing about the person.** A bug found by clicking around could not be reconstructed afterwards. Every interaction is now logged and tagged, including which mode was picked, which pencil was pressed and whether anything had been edited first.
- **Coaching mode padded two words into twenty-two.** Typing "thanks tobias" produced "Tobias, thank you for all your hard work lately. I really appreciate the effort you've been putting in to support the team." The manager never mentioned a team, effort, or anything ongoing. The cause was the writing guideline itself: W2 read "two to four sentences", which is a floor as well as a ceiling, so a two-word input got padded up to meet it. W2 now reads as a ceiling that only a rich input earns, with length following the number of distinct things the manager actually said. Coaching mode is still allowed to add phrasing around the manager's words. What it may not do is reach a length by inventing content.
- **The same padding came back as soon as several people were named, and the fix had to change shape.** "thanks Elena, Diego, and Hannah, Oscar, Mei, and all" produced 38 words. Two rules were pulling against each other. W3 requires a different register per relationship type, so a mixed group cannot be one sentence, while the new W2 said a thin input is worth one sentence. W3 won, and the model then read five names as five things worth saying. W2 now takes its floor from the number of relationship groups rather than from a flat count, states plainly that names never add length, and caps at four groups by combining the two closest registers. Separately, the model had been left to sort the recipients into groups itself and got it wrong, producing sets that matched no real relationship and calling three people "both". Grouping is a lookup rather than a judgment, so the app now works the groups out and hands them over in the prompt.

### Bugs in the tests themselves

Worth listing, because a test that fails correct behaviour is as damaging as no test.

- The "invented numbers" check called correct arithmetic a fabrication. The agent said someone had 2 recognitions, which was true, but a count is worked out rather than quoted so it never appears in the input.
- One case failed the Draft step for repeating a dollar figure the manager had typed. That is allowed, and blocking money is the policy check's job.
- The end-to-end harness held onto a stale copy of the screen state, so an edit landed somewhere the gate never looked. The test reported a pass while testing nothing.
- The model-call counter only counted a third of the calls.
- The log was storing the name of the rule instead of the text actually sent to the AI, so it could not answer the first question anyone asks about a surprising result.

---

## Where a person is still needed

- **At the approval gate, every time.** Nothing sends without a manager approving or editing it. A blocked recognition goes back to that manager and to nobody else.
- **Deciding what "good" means.** The AI added "was a huge win for the team" to a draft. Whether that counts as making things up is a judgment call. The ruling was: acceptable in Coaching mode, not in Clean-up. Both tests were rewritten to match.
- **Judging whether writing reads well.** A second AI call flags possible problems, but it is reported as an opinion and never counted in the score, because it gave different answers on repeat runs.
- **Looking at the screen.** Several bugs were invisible to tests that checked internal values while the display was wrong.

The evaluation panel used to have a Pass or Fail dropdown that a person set by hand, with no code comparing anything. It could not fail a test. It was deleted and replaced with checks that run automatically.

---

## What is still not tested

- Everything runs on a small made-up dataset. Real use would produce inputs this data does not contain.
- The tests only check rules the instructions actually state. They cannot find a rule nobody wrote down. That is exactly how the Clean-up problem stayed hidden.
- Model prices are copied by hand from Google's pricing page and will go out of date.
- No graded Draft case hands that step something it must refuse to repeat. They all use publishable text. The refuse-to-repeat rule added on 2026-08-06 is proven by the end-to-end paths and not by the Draft step's own twelve cases.
- Two known defects are diagnosed and deliberately not fixed. The Opportunity agent is asked the same question five or six times per person, because its cache only records an answer after the model replies and the daily detection pass has a re-entry guard on one of its callers rather than on itself. Separately, the end-to-end results label every passing row "Sent:", including the two where the gate correctly refused and nothing went out.

---

## Files

Results are saved under `reference-files-github-ignore/live-eval-results/`.

- `compare-<timestamp>/<model>/summary.txt` — scores, tokens and cost for one model
- `compare-<timestamp>/<model>/opportunity-sweep.txt` — who that model flagged, and its reasoning
- `compare-<timestamp>/<model>/full-log.json` — every call, with the exact text sent and returned
- `<timestamp>-drafts.txt` — every draft the model wrote, for reading by eye

Every person, company, recognition and benchmark figure in this project is synthetic.
