# The Unofficial Guide

Satya Bhargav Maddipati — corpus: `campus_life`

> **This file is your submission.** Fill it in as you go — most sections get
> written during the milestone that produces them, not at the end.
>
> How the starter works, and every command you'll need, is in `RUNNING.md`.
> Leave that file alone.
>
> **Paste everything as text.** No screenshots, no video. A typed table gets
> full credit; a picture of the same table gets none.
>
> Delete these instruction blocks as you replace them. The `<!-- -->` comments
> are notes to you and don't show up when the page renders — you can leave them
> or remove them.

---

# Unit 1

## What This Does

This is a retrieval-augmented question answerer built on the `campus_life`
corpus — 88 short, single-topic posts about student life at a university:
dining hall wait times, housing quirks, course workloads, and the
administrative rules that never get explained plainly (add/drop deadlines,
parking permits, meal plan changes). Ask it something one of those posts
actually answers — "how quickly do west lot parking permits sell out?" — and
it retrieves the matching chunk, cites the source file, and answers from it.
Ask it something outside the corpus and the relevance gate refuses rather than
guessing.

## Chunking Strategy

**Chunk size:** 600 characters (packed by paragraph, not a fixed window)
**Overlap:** 0

I ran the starter's `fallback_split` against all three corpora before touching
anything: `campus_life` came out 88 documents → 88 chunks (nothing splits,
shortest 178, longest 549), `city_guides` came out 14 → 51 (cutting straight
through labelled sections), and `advice_threads` produced a 2-character chunk
— the leftover tail of a document that didn't divide evenly into 800-character
windows. That's not a bug, it's the fixed-window strategy doing exactly what
it does.

For `campus_life`, the 1:1 result isn't an accident worth ignoring — I checked
why. The corpus is already hand-split into single-topic files: a hall's
laundry situation lives in its own `_laundry.txt`, its noise situation in its
own `_noise.txt`, separate from the hall's overview file. So "one file, one
complete thought" is already true of the source material, not something my
chunker has to manufacture.

That's why I replaced fixed-window slicing with paragraph packing: split each
document on blank-line paragraph breaks, then greedily merge consecutive
paragraphs into a chunk as long as the running total stays under
`CHUNK_SIZE`. I set `CHUNK_SIZE` to 600 — just above the longest real document
(549) — so it acts as a safety cap rather than a target: almost every file
still becomes exactly one chunk, and the packing only kicks in if a future or
edited document mixes more than one topic into a single file. I set
`CHUNK_OVERLAP` to 0 because there's no fixed-window seam to patch — packing
never cuts inside a paragraph, so there's nothing lost between adjacent
chunks that overlap would need to restore.

I did consider splitting every document into one chunk per paragraph instead
(so, e.g., `housing_innisfree_hall.txt`'s "good," "bad," and laundry/noise
sentences would each be their own chunk). I didn't, because those specific
facts already have their own dedicated, better-written files elsewhere in the
corpus — splitting the overview file further would just produce a worse
duplicate of a chunk that already exists.

## Sample Chunks

**Chunk 1** — source: `admin_add_drop_deadline.txt#0` — produced by: `chunker.py::split_documents`

```
On the add/drop deadline

You can add a course through the end of the second week. Dropping is a longer window — through the end of week six — but a drop after week two shows as a W on your transcript. Nothing anywhere on the registrar's site says this plainly, and students find out from each other.
```

**Chunk 2** — source: `course_biol_160.txt#0` — produced by: `chunker.py::split_documents`

```
BIOL 160 Cell Biology

I lived here my sophomore year. Format is lecture three times a week with a weekly lab. Assessment: four unit tests and a cumulative final. Not curved.

Expect 9 to 11 hours a week, the heaviest first-year course by reputation.

The one piece of advice: the unit tests come fast, roughly every three weeks; falling behind once is very hard to recover from.
```

**Chunk 3** — source: `course_hist_118_workload.txt#0` — produced by: `chunker.py::split_documents`

```
Workload for HIST 118 Modern World History

People keep asking so: a lot of reading, about 120 pages a week, but no problem sets. That's real time, not optimistic time.

It's front-loaded — the first month is heavier than the rest, partly because you're learning the format.
```

**Chunk 4** — source: `dining_pellew_dining_hall_followup.txt#0` — produced by: `chunker.py::split_documents`

```
Re: Pellew Dining Hall

Adding to what people have said about Pellew Dining Hall. The wait figure of 12 to 18 minutes at peak matches what I've seen. If you're trying to eat between classes, go before 11:45 and it's a different building entirely.

Also worth saying: the furthest hall from anywhere, next to the athletics centre. Nobody tells you this at orientation.
```

**Chunk 5** — source: `housing_innisfree_hall.txt#0` — produced by: `chunker.py::split_documents`

```
Innisfree Hall — what it's actually like

Transferred in last year, so take this with a grain of salt. Built 1991, renovated 2022. Rooms are doubles arranged as pairs sharing one bathroom between two rooms.

The good: the shared-bathroom-between-two-rooms arrangement is the best compromise on campus.

The bad: no air conditioning, which matters for the first three weeks of September.

Laundry costs $1.75 wash, $1.75 dry, app-based. On noise: moderate; the building is L-shaped and the short wing is much quieter.
```

Each of these stands alone: no sentence is cut off at either edge, and each
names its own topic (course, deadline, dining hall, dorm) explicitly rather
than relying on something said in a previous chunk.

## Sample Answer

**Question:** How long is the wait to eat at Kestrel Commons during the lunch rush?

**Answer:**

```
(best distance 0.191, cutoff 0.6)

The wait time at Kestrel Commons is 20 to 25 minutes between 12:15 and 1:00.

Source: `dining_kestrel_commons.txt` (also mentioned in `dining_kestrel_commons_followup.txt`).

Sources retrieved: dining_halden_hall_followup.txt, dining_kestrel_commons.txt, dining_kestrel_commons_followup.txt, dining_pellew_dining_hall_followup.txt, dining_the_ridgeway_cafe_followup.txt
```

**My relevance cutoff:** 0.6 (the starter's default — I measured my own
distances before deciding whether to move it, and the gap made changing it
unnecessary).

I ran all five `QUESTIONS` and all five `OUT_OF_SCOPE` questions through
`store.search` and recorded the best (lowest) distance for each:

| Question | In corpus? | Best distance |
|---|---|---|
| How long is the wait to eat at Kestrel Commons during the lunch rush? | Yes | 0.191 |
| How quickly do west lot parking permits sell out? | Yes | 0.226 |
| What is the last week a student can add a course? | Yes | 0.382 |
| How much printing credit does a student get each semester? | Yes | 0.391 |
| What time does the library close during reading week? | Yes | 0.412 |
| What is the capital of Mongolia? | No | 0.825 |
| Who won the 1994 World Cup? | No | 0.886 |
| How do I write a for loop in Rust? | No | 0.896 |
| What is the recommended dosage of ibuprofen for a headache? | No | 0.844 |
| How do I change the oil in a diesel engine? | No | 0.934 |

The two groups don't overlap at all: every in-corpus question landed at 0.412
or below, every out-of-scope question at 0.825 or above, leaving a clean gap
between 0.412 and 0.825. The starter's default of 0.6 sits comfortably in the
middle of that gap, so I left it alone rather than moving it — moving it would
only have mattered if the groups had been close together or overlapping.

One thing that surprised me: I'd expected "How do I write a for loop in Rust?"
to be the closest call, since `campus_life` includes real CS course documents
(CS 210, CS 340). It actually came back as the *furthest* of all ten (0.896),
closer to `course_hist_118_exams.txt` than to anything CS-related. My
`campus_life` documents apparently don't use language close enough to a Rust
tutorial for that to matter.

## How I Used AI

**1.** Before touching `chunker.py`, I asked Claude Code to check what the
starter's fixed-window chunker actually did to each corpus, rather than just
reading the docstring's claim about it. It ran `fallback_split` against all
three corpora and reported exact numbers: `campus_life` 88 documents → 88
chunks, `city_guides` 14 → 51 (cutting through labelled sections), and
`advice_threads` producing a 2-character leftover chunk. That last number is
what made me stop treating campus_life's 1:1 result as luck — I opened
`housing_innisfree_hall.txt` next to its `_laundry` and `_noise` sibling files
myself and confirmed the corpus is already hand-split into single-topic files.
So I changed my plan from "just shrink the fixed window" to a paragraph-packing
strategy with a size cap, since a fixed window was solving a problem this
corpus didn't actually have.

**2.** For Milestone 4, I asked it to run my five `QUESTIONS` and all five
`OUT_OF_SCOPE` questions through retrieval and print the best distance for
each before I decided whether to move the default cutoff. I'd expected "How do
I write a for loop in Rust?" to be the risky one, since the corpus has real CS
course documents, and had written that expectation into `criteria.md` before
measuring anything. The actual number came back at 0.896 — the *furthest* of
all ten questions, not the closest — with a clean, non-overlapping gap between
0.412 (worst in-corpus) and 0.825 (best out-of-scope). That changed my plan
from "tune THRESHOLD" to "confirm the default is already fine and write down
why," and I kept my original wrong guess in `criteria.md` rather than editing
it, since the gap between what I expected and what I measured was worth
keeping visible.

<!-- ── Stretch features ─────────────────────────────────────────────────────
     Doing one? Say so here BEFORE you start. A feature this README never
     claims earns nothing.
     ───────────────────────────────────────────────────────────────────────── -->

---

# Unit 2

<!-- These sections get ADDED to what's already above. Don't delete or rewrite
     unit 1 — the point is that someone can see what you said before you knew
     how it went. -->

## Run Log — Before

Source data: `results/run_2026-09-28_0050_before.md`, produced by
`python run_eval.py --label before` (`run_eval.py::main` and
`run_eval.py::check_out_of_scope`, no `scorer.py` yet, so I judged each
answer by reading it against `expects` in `questions.py`).

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Chunks read as complete, self-contained thoughts | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 5. The source named is the source that actually backs the answer | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |

Criteria 3 and 4 show the same number in all three columns on purpose, and for
the same underlying reason even though they're measured by different code.
Criterion 3 is one deterministic pass of `store.py::search` against a fixed
cutoff (`gate.py::check`) — nothing about asking three times changes a
distance. Criterion 4 doesn't touch `run_eval.py` at all; `chunker.py`'s
`split_documents` produces the same 88 chunks from the same 88 documents every
time, so sampling them again wouldn't move the number either. Criteria 1, 2,
and 5 depend on `generate.py::answer_from_chunks`, which is the one thing that
actually varies run to run (uncached model calls), which is why those are the
ones the instructions expect to move — mine just happened not to.

**Criterion 1 — real output.** Retrieved chunk for "How much printing credit
does a student get each semester?" (`store.py::search`, chunk from
`chunker.py::split_documents`, source `admin_printing_quota.txt`):

```
On the printing quota

Every student gets $30 of printing per semester, which is roughly 600 black-and-white pages. It does not roll over. Colour costs eight times as much per page, which people discover after printing one poster.
```

**Criterion 2 — real output.** Full answer, run 2, from
`generate.py::answer_from_chunks`:

```
The wait time at Kestrel Commons is 20 to 25 minutes between 12:15 and 1:00. 

Source: `dining_kestrel_commons.txt` (also mentioned in `dining_kestrel_commons_followup.txt`).
```

**Criterion 3 — real output.** From `run_eval.py::check_out_of_scope`
(`gate.py::check`):

```
refused  (best distance 0.896)  How do I write a for loop in Rust?
```

**Criterion 4 — real output.** Chunk 1 of 5 sampled by `app.py chunks -n 5`
(`chunker.py::split_documents`) — see the Sample Chunks section above for all
five; one repeated here for this row's evidence:

```
On the add/drop deadline

You can add a course through the end of the second week. Dropping is a longer window — through the end of week six — but a drop after week two shows as a W on your transcript. Nothing anywhere on the registrar's site says this plainly, and students find out from each other.
```

**Criterion 5 — real output.** This is the one I was actually worried about
in `criteria.md`: the Kestrel Commons question retrieves four near-identical
dining hall documents alongside the real one
(`store.py::search`), yet `generate.py::answer_from_chunks` still names the
correct one:

```
Sources retrieved: dining_halden_hall_followup.txt, dining_kestrel_commons.txt, dining_kestrel_commons_followup.txt, dining_pellew_dining_hall_followup.txt, dining_the_ridgeway_cafe_followup.txt

Source: `dining_kestrel_commons.txt` (also mentioned in `dining_kestrel_commons_followup.txt`).
```

## Verdicts

| # | Criterion | Verdict | How I decided |
|---|---|---|---|
| 1 | Retrieved chunk contains the answer (target: 4 of 5) | MET | I didn't just trust the generated answer — for each question I checked the actual "Sources retrieved" list against the document I know holds the fact. The correct source was in the top-5 for all 5 questions, and since retrieval is deterministic that held for all 3 runs, not just one. |
| 2 | Every answer names a source (target: 5 of 5) | MET | I read all 15 generated answers (5 questions × 3 runs) as literal text and every one carried an explicit source citation. The target was already all five, so there was no room to round up — it either held for all 15 or it didn't, and it did. |
| 3 | Gate stops out-of-corpus questions (target: 4 of 5) | MET | One deterministic pass of the 5 `OUT_OF_SCOPE` questions through the gate: all 5 refused, with best distances (0.825–0.934) sitting well clear of the 0.6 cutoff. Not a close call in either direction. |
| 4 | Chunks read as complete, self-contained thoughts (target: 4 of 5) | MET | I actually applied the "could someone answer using only this" test to all 5 sampled chunks instead of skimming them. The `dining_pellew_dining_hall_followup.txt` chunk was the closest call — it frames itself as a reply to something unstated ("matches what I've seen") — but it still names the hall and states the number outright, so it passes on its own. All 5 held. |
| 5 | Source named is the source that actually backs the answer (target: 4 of 5) | MET | For every run I checked whether the *cited* document's text actually contains the quoted fact, not just whether some source was named. The Kestrel Commons question was the real test, since 4 near-duplicate dining-hall documents sat in the same top-5 results — the model named the correct one in all 3 runs. |

None of these turned out to be broken — each was measurable exactly as written
in `criteria.md` (a fixed document to check against, a literal string to look
for, a distance to compare, a five-chunk sample, a source-to-text match), so
I'm not revising any of them this unit. If something had come back
inconsistent in a way I couldn't pin down to a real cause — e.g. if "contains
the answer" had meant something different to me on two different reads — that
would be a measurement problem worth rewriting the criterion over. That's not
what happened here; every number came out the same way for a reason I could
point to.

## Diagnoses

<!-- For each miss: which stage caused it, and how. The stage alone isn't
     enough — you need the mechanism.

     Not a diagnosis: "Question 3 didn't work."
     A diagnosis:     "Question 3 asks about laundry costs. The answer is in
                       one sentence that got split across two chunks, so
                       neither chunk on its own contains it."

     The five stages: loading → chunking → embedding → retrieval → generation.

     Look for a pattern. If three misses all ask about numbers, that's one
     problem, not three.

     Missed nothing? Say so, then say honestly whether your targets were set
     low, and which one you'd tighten and to what.

     Milestone 3. -->

## The Improvement

**What I changed:**

**Why I picked it:**

<!-- Connect it to a specific diagnosis above in one sentence. If you can't,
     you picked a fix because it sounded impressive. -->

### Run Log — After

<!-- Same format, same five criteria, three runs each.
     `python run_eval.py --label after` -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 |  |  |  |  |
| 2. Every answer names a source | 5 of 5 |  |  |  |  |
| 3. Gate stops out-of-corpus questions | 4 of 5 |  |  |  |  |
| 4. | | | | | |
| 5. | | | | | |

**Did it help?**

<!-- Say plainly whether it did, and how you know. If it made things worse,
     say that — a change that backfired, honestly reported, earns full credit
     and is more interesting than one that worked. What matters is that you can
     tell.

     Milestone 4. -->

## What's Still Broken

<!-- For each criterion still missed after your fix: what you'd do about it,
     and why you stopped where you did.

     "I ran out of time" is fine if it's true. Pretending nothing is left is
     not.

     Milestone 5. -->

## What I'd Do Differently

<!-- Knowing what you know now — which of your five criteria would you write
     differently, and why?

     Milestone 5. -->
