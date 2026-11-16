# Iteration log

| Date | Version | Failure targeted | Layer | Hypothesis | Change made | Before | After | What I learned |
|---|---|---|---|---|---|---|---|---|
| Nov 2 | v0 (hash ea912be88a27) | Baseline | All | — | Built the agent: 3 tools, read-only SQL guardrails (SELECT only, 200 rows, 10 s, 8 steps, 20 questions/session), traces to JSONL and Supabase, Streamlit UI | First 5 questions run | — | Guardrails passed every attack test; the hard part is answer quality, not plumbing |
| Nov 2 | v0 → v1 (fd9d9058059b) | Model arithmetic (EA04: 63.5% vs. correct 63.4%) | Rules / spec | The model divided by itself; a loose rule (“or directly calculated from”) allowed it | New rule: compute every number in SQL. Removed the conflicting loose rule | 1 of 5 numbers wrong | Pending | Conflicting rules in a prompt are a real failure source; the model follows the convenient one |
| Nov 2 | v0 → v1 | Technical leakage (column names in “What I counted”) | Prompt | The rule was too abstract; an example will anchor it | Plain-words rule plus a bad/good example pair | 5 of 5 leaked | Pending | A rule stated once in a prompt isn’t enough; candidate code eval |
| Nov 2 | v0 → v1 | Format (list instead of table, winner only, irrelevant caveat) | Output format | Explicit format rules fix these | Table for more than 3 values; top 5 for “most/top”; only relevant caveats | EA05 list; EA02 winner only; MISO “(not relevant here)” | Pending | Formatting failures are spec gaps, so fix them in the prompt; no eval needed |
| Nov 2 | v0 → v1 | Latency and wasted steps | Context | The agent re-discovers labels and columns every run | Quick reference: 14 resource labels and 20 key columns; describe_table only if needed | 22–78 s; 2 discovery steps per run | Pending | Context engineering can cut cost and time, not just improve accuracy |
| Nov 2 | v0 → v1 | Missed anomaly; “right now” not flagged | Context / degradation | The agent reports numbers without an analyst’s eye | Flag sudden jumps or drops (don’t guess the cause); note that data isn’t live for “right now” questions | EA05 missed PJM 2024 (2 requests); EA03 no live-data note | EA03 now flags data isn't live ✓; EA05 flags the 2024 drop but invents a cause ✗ (9e6652bd) | Domain judgment has to be written down for the agent |
| Nov 2 | UI v0 → v0.1 | Broken icons, overlapping buttons, cut-off input | UI | Custom font CSS overrode the icon font | Scoped the font CSS, fixed feedback buttons, multi-line input, subtle grid background | Icons showed as text | Pending check | Test in a real browser; automated tests missed the visual bugs |
| Nov 10 | v1 | Invented cause: EA05 explains PJM's 2024 drop as "incomplete filing data" (9e6652bd) | Prompt | "Flag, don't guess the cause" stated once isn't enough | v2 backlog: bad/good example of an invented cause | 1 of 25 scripted v1 answers | Pending (v2) | |
| Nov 10 | v0 | NULL trap: visitor got 1 GW instead of 165.7 GW for MISO solar, and gave it 👍 (f13a90e6) | SQL / ground truth | NULL type columns zero out whole rows when MW is multiplied by a boolean | v2 backlog: capacity SQL template (CASE WHEN + COALESCE) | 1 GW shown, approved by the visitor | Pending (v2) | You have to review each trace in some way because a visitor's marking it as thumbs up doesn't mean that the answer is correct |
| Nov 15 | v1 | Run-to-run inconsistency: same prompt, same EA01 question, correct SQL in a8cb6a68, whole-request SQL in 755ae307 | SQL / prompt | Rules alone don't pin down the SQL; the model picks a different pattern each run | v2 backlog: capacity SQL template | 1 of 2 EA01 runs wrong | Pending (v2) | Strongest evidence for a SQL template |
| Nov 14–15 | Eval process | AI pre-labeler trust | Eval process | Agreement is limited by unclear type definitions, not by the AI | Blind test, then wrote `evals/failure_types.md` (definitions + labeling rules) | Blind test: Pass/Fail 14 of 15 (5 of 5 seen before, 9 of 10 unseen); types 60% | After definitions: Pass/Fail kept 22 of 24, types 22 of 24 | The definitions were the problem, not the AI |
| Nov 15 | Infrastructure | Groq 200K tokens/day limit | Harness | Rate-limited runs say nothing about answer quality | Excluded 3 labeled rate-limit traces (c149277b, 6c6903ec, 36398bf0); friendly error message already shipped (ae28669); decline parser fix for EA18 (d1ec49a4) done (18c0500) | 3 traces counted as fails | Excluded from the quality eval | |

### v1 baseline (Nov 15)

Scope: prompt v1 (`fd9d9058059b`), 35 traces, all labeled in `evals/annotations.csv`. Test rows and the 5 rate-limited runs are excluded.

**Pass/Fail**
- Scripted (EA01–EA25, plus a second EA01 run): 11 pass, 15 fail, 42% pass rate
- Visitor: 8 pass, 1 fail, 89%
- All v1: 19 pass, 16 fail, 54%

**Failure types on Fails only** (16 fails; a fail can have more than one type)
- no ranking table: 6
- caveat missing or doesn't fit: 5
- jargon in answer: 5
- method described wrong: 3
- wrong filter or definition: 2
- wrong capacity SQL: 2
- invented cause: 1

**Latency**
- Median 28.7 s overall (scripted 35.0 s, visitor 2.6 s)
- Slowest: EA17 66.7 s, EA02 61.9 s, EA04 59.3 s (all scripted)

**AI pre-labels** (`evals/suggestion_review.csv`, 24 reviewed)
- Pass/Fail kept 22 of 24 (92%), types kept 22 of 24 (92%), first-failure wording kept 14 of 24 (58%)
- Erred in both directions: EA17 too strict on the COALESCE rule (the missing COALESCE only skips nuclear rows with no recorded MW, so I passed it); EA13 missed a failure (tagged the raw status code as jargon but called it a Pass; I failed it)

**Cautions**
- The visitor sample is small (9) and mostly repeats of the MISO solar chip, so its 89% isn't comparable to the scripted 42%. The scripted set is the baseline to beat.
- Re-run Nov 15 on the paid tier: median 2.4 s vs 44.3 s on Nov 2, same prompt, code and step counts (EA02, EA04, EA06, EA08, EA11). The Nov 2 slowness was most likely the free tier near its daily limit, not the agent.

**What I learned:** Most v1 failures were in how answers were presented (jargon, no tables, etc.), not the math. But the math failures are the dangerous ones, and the same prompt produced correct and incorrect SQL on different runs, so prompt rules alone can't be trusted for calculations.

### Notes (Nov 15)
- The planned answer validator wouldn't catch the NULL trap: 1 GW really did come from a query (f13a90e6). Only ground truth caught it.
- The "every number from a query" rule may block comparison sentences, a rule with a side effect (EA11, f15130fb).
- v1's prompt examples overlapped eval questions EA10 (word for word), EA03 and EA16, so v1's results may be slightly inflated on those three. v2 replaces them with non-eval questions.
- Nov 16: after a push, reboot Streamlit and confirm `prompt_version` in Supabase before testing live. The first live Virginia tests after the v2 push (cde8727f, 64681516, e4309612, 4e7d4f5b) still ran on v1 (`fd9d9058059b`) with no checks.

### v2 results (Nov 15)

Scope: prompt v2 (`992c40f59054`) with the `resource_mw` table and the jargon and table checks (one rewrite each), 2 runs of the 25 scripted questions, all 50 labeled.

**Pass rate**
- v1 scripted (baseline): 11 of 26, 42%
- v2 run 1: 18 of 25, 72%
- v2 run 2: 17 of 25, 68%
- v2 combined: 35 of 50, 70%

**Failure types on Fails only** (v1 → v2 run 1 / run 2 / combined)
- no ranking table: 6 → 0 / 0 / 0
- jargon in answer: 5 → 2 / 1 / 3
- caveat missing or doesn't fit: 4 → 4 / 1 / 5
- method described wrong: 3 → 0 / 0 / 0
- wrong filter or definition: 2 → 4 / 4 / 8
- wrong capacity SQL: 2 → 0 / 0 / 0
- invented cause: 1 → 1 / 1 / 2
- fail with no type (no answer, EA07 run 2): 0 → 0 / 1 / 1

**Consistency:** 5 of 25 questions passed in one run and failed in the other: EA03, EA07, EA17 (pass → fail) and EA24, EA25 (fail → pass). 15 passed in both runs; 5 failed in both (EA05, EA06, EA13, EA19, EA21).

**Checks and data:** checks fired on 8 of 25 first answers in run 1 and 6 of 25 in run 2. The table check fixed all 6 tables it asked for; one jargon rewrite still had a quoted "unknown" (EA04 run 1). `resource_mw` was used in 24 of 26 capacity traces. Median latency 2.3–2.5 s.

**AI pre-labels** (50 reviewed): Pass/Fail kept 50 of 50, types kept 50 of 50, first-failure wording kept 36 of 50. Reviewed with suggestions shown, so this may include anchoring; a blind spot-check is in the backlog.

**Summary:** presentation failures (tables, method wording) and capacity SQL cleared. Wrong filters are now the top failure (8: EA06, EA17, EA19, EA21, EA24). Invented cause persists (EA05, both runs).

**What I learned:** The structural fix (resource_mw) eliminated capacity SQL errors, and code checks fixed every missing table. But wrong filters became the top failure: undefined words like 'proposed' changed answers by up to 4x. And a prompt example didn't stop the invented cause.

### v3 results (Nov 15)

Scope: prompt v3 (`4f4504d77ded`) with the forced final answer, invented-cause check, stricter table and jargon checks and the parser fix; one run of the 25 scripted questions. 15 reviewed by me (the 10 questions that failed or flipped in v2, 4 of them blind, plus 5 spot-checked Passes); the other 10 have AI pre-labels only.

**Pass rate**
- v1 scripted: 11 of 26, 42%
- v2 (2 runs): 35 of 50, 70%
- v3, my 15 reviewed: 13 of 15, 87% (a hard sample: every v2 failure or flip is in it)
- v3, all 25 (my 15 + 10 unreviewed AI pre-labels, all Pass): 23 of 25, 92%

**Failure types on Fails only** (v1 → v2 → v3)
- wrong filter or definition: 2 → 8 → 1 (EA21, proposed-date filter, 266 vs 271)
- number not from query (new): — → — → 1 (EA06, model summed the five counties itself, 49.1 GW)
- invented cause: 1 → 2 → 0
- caveat missing or doesn't fit: 4 → 5 → 0
- jargon in answer: 5 → 3 → 0
- no ranking table: 6 → 0 → 0
- method described wrong: 3 → 0 → 0
- wrong capacity SQL: 2 → 0 → 0
- no answer: — → 1 → 0

**AI pre-labels:** non-blind, Pass/Fail and types kept 11 of 11, first-failure wording 8 of 11. Blind first-call agreement was 3 of 4. The one difference (EA21) was the AI catching a failure I missed, so the pre-labeler adds value beyond agreement. Four blind traces are too few to measure anchoring either way.

**Run:** checks fired on 7 of 25 first answers (5 jargon, 2 table); median latency 2.3 s; median 10,806 tokens per question.

**Notes**
- One run only; v2 showed 5 of 25 questions flip between runs, so stability isn't proven.
- The invented-cause check and the forced final answer never triggered, so they're built but untested on new answers.
- v2's bad example never contained "incomplete filing data"; the model produced it on its own.
- EA21 keeps adding a proposed-date filter despite v3's rule: the word "proposed" matches the `prop_date` column name, and that match beats the prompt rule. It's a naming problem, not a prompt problem (v4: describe the column differently, or a code check).

**What I learned:** Every fix moved the failures somewhere new: from presentation (v1) to definitions (v2) to naming and arithmetic (v3). The remaining failures get harder to find, which is why reading traces still matters at 87%.
