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

**3.** For unit 2's improvement, before writing any code I asked it to predict
whether adding BM25 hybrid search would actually move my numbers, given that
my before-run already scored 5/5 on every criterion. It reasoned that Kestrel
Commons' real document was already the single closest semantic match by a
wide margin (0.191, versus 0.33+ for the nearest distractor), so re-ranking
within an already-correct top-5 had no wrong answer to fix — the prediction
was "probably no measurable change, but expect the *set* of distractors to
shift." That's exactly what happened when I ran it: identical verdicts on
every criterion, but a different distractor mix. What I hadn't predicted, and
Claude hadn't either until we looked at the actual retrieved sources, was
*which* distractor would show up — `transit_walking.txt`, pulled in only
because it names "Kestrel Commons" once in an unrelated sentence about walking
times. I added that specific finding to "What's Still Broken" myself, since it
came from reading the real output, not from anything either of us predicted
going in.

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

I missed nothing. All five criteria came out MET on all three runs — no
question failed, no source was misattributed, no out-of-scope question got
through. There's no failure to trace to a pipeline stage, so there's nothing
to diagnose in the usual sense.

That's a result worth being suspicious of, not proud of. A system that clears
every criterion on the first try usually means the criteria were safe, not
that the system is excellent, so instead of stopping there I checked *how
hard* each criterion had actually been stress-tested rather than just whether
it passed.

The gap I found: criteria 1 and 5 both exist because of one specific risk —
`campus_life` has six dining-hall documents and six housing-hall documents
written from the same template, so embedding similarity between "Kestrel" and
"Pellew" (say) could plausibly retrieve the wrong hall, or the model could cite
the wrong one even when the right chunk is present. But only **1 of my 5
questions** (Kestrel Commons) actually has a near-duplicate sibling in the
corpus. The other four (add/drop deadline, printing quota, parking permits,
library hours) are each the *only* document about their topic, so getting them
right is close to automatic — there's no distractor for retrieval or
generation to get confused by. A target that's only stress-tested by one
question out of five isn't really tested at 4-of-5 confidence; it's passed by
default four-fifths of the time.

**What I'd tighten, and to what:** criterion 5 — "the source cited is the
source that actually backs the answer" — from *"at least 4 of 5 test
questions"* to *"at least 4 of 5 test questions, where at least 2 of the 5
have a near-duplicate templated sibling document in the corpus."* That's a
coverage requirement on the test *design*, not just a stricter number — it
forces the test to actually exercise the mechanism the criterion exists to
catch, instead of letting four easy questions carry one hard one to a passing
average. I'm not making this change to `criteria.md` itself, since nothing
about the criterion was *broken* — it was measurable exactly as written, I
just designed a test suite that didn't stress it as hard as it could have. That
belongs in "What I'd Do Differently" below, not as a revision.

## The Improvement

**What I changed:** Added hybrid search. `store.py::search` now widens the
semantic candidate pool to 15 and, when `config.RETRIEVAL_MODE == "hybrid"`
(`AI201_RETRIEVAL_MODE=hybrid`), re-ranks it with `store.py::_hybrid_rerank` —
a 50/50 blend of cosine similarity and BM25 keyword overlap
(`rank-bm25`, already in `requirements.txt`) — before slicing to `top_k`. The
`distance` field on each `Result` stays the real cosine distance regardless of
mode, so `gate.py::check`'s 0.6 cutoff means exactly what it always meant;
hybrid mode only changes which chunks are in the running and in what order.

**Why I picked it:** It's a direct test of the mechanism named in Diagnoses.
`campus_life` has six dining-hall and six housing-hall documents written from
one template, so a question about "Kestrel Commons" risks having its exact
name diluted by boilerplate phrasing ("matches what I've seen," "the salad bar
wilts") shared with Halden, Pellew, and Ridgeway. BM25 is specifically good at
exact-token matches a semantic embedding blurs together, which is the
opposite failure mode from semantic search's strength — so combining them
should help precisely where the near-duplicate-template risk lives, without
giving up anything semantic search already does well.

### Run Log — After

Source data: `results/run_2026-09-28_0107_after.md`, produced by
`AI201_RETRIEVAL_MODE=hybrid python run_eval.py --label after`.

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Chunks read as complete, self-contained thoughts | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 5. Source named actually backs the answer | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |

Identical verdicts to Run Log — Before, question for question. But the
retrieval sets underneath weren't identical — hybrid re-ranking measurably
changed which distractors showed up. For the Kestrel Commons question:

```
Before: dining_halden_hall_followup.txt, dining_kestrel_commons.txt, dining_kestrel_commons_followup.txt, dining_pellew_dining_hall_followup.txt, dining_the_ridgeway_cafe_followup.txt

After:  dining_kestrel_commons.txt, dining_kestrel_commons_followup.txt, dining_the_ridgeway_cafe_followup.txt, dining_verrill_street_grill_followup.txt, transit_walking.txt
```

(`store.py::search` → `store.py::_hybrid_rerank`.) `transit_walking.txt` is a
genuinely interesting entrant — it contains the literal phrase "Morrow House
to Kestrel Commons: 7 minutes," so BM25 rewarded it for naming "Kestrel
Commons" exactly, even though the document is about walking times and answers
nothing about wait times. BM25 doesn't know the difference between a document
*about* something and a document that just *names* it in passing.

**Did it help?** No — not on this test suite, and I can say that precisely
rather than vaguely. Every criterion stayed at exactly the same score, run for
run, before and after (see the two tables above). The two real Kestrel Commons
documents never left the semantic top-5 in the first place (Kestrel's own
document was always the single closest match, distance 0.191, the largest
margin of any of my five questions), so there was no wrong answer for
re-ranking to fix, and no headroom for it to show a gain. What it did do was
swap out *which* distractors ride along in the context window — sometimes for
a worse one (`transit_walking.txt`, which name-drops the entity without
answering the question) — with zero effect on the model's final citation
either way.

This lines up with the Diagnosis exactly: I picked hybrid search because
criteria 1 and 5 exist to catch near-duplicate-template confusion, but only 1
of my 5 questions actually has a near-duplicate sibling, and that one question
was already comfortably correct under semantic-only search. The fix is
real and the reasoning for it holds, but my test suite doesn't contain a
question hard enough to need it — which is the same gap Diagnoses already
named, from a different angle.

## What's Still Broken

No criterion is missed, before or after the fix — but "nothing missed" isn't
the same as "nothing left," and two real gaps remain:

1. **My test suite still under-stresses criteria 1 and 5.** Only 1 of my 5
   questions has a near-duplicate templated sibling in the corpus, so a clean
   5/5 doesn't prove the system handles that risk reliably — it proves it
   handles it once. I'd fix this by adding 2 more `QUESTIONS`, one about a
   different dining hall and one about a different housing hall, specifically
   chosen because they have near-identical sibling documents, and re-running
   both retrieval modes against the expanded set.

2. **The hybrid re-ranker can't tell "about X" from "mentions X."**
   `transit_walking.txt` entered the Kestrel Commons top-5 under hybrid mode
   purely because it names "Kestrel Commons" once, in a sentence that isn't
   about wait times at all. It never displaced the right answer in my runs,
   but a larger corpus with more incidental name-drops could let a
   passing-mention chunk crowd out the real one. I'd address this by requiring
   a minimum semantic similarity floor before BM25 gets a vote (so a document
   has to already be somewhat relevant in meaning, not just contain the right
   word), rather than the flat 50/50 blend I used here.

I stopped here because both of these are about making the *test* harder, not
because I found a live failure — I ran out of scope for this unit, not out of
ideas, and this file being real evidence: my before and after run logs, plus
this write-up naming both gaps, is a complete report of where I stopped.

## What I'd Do Differently

**Criterion 5's target itself was fine; my test design around it wasn't.**
Knowing what I know now, I'd write the QUESTIONS *selection process* into
`criteria.md` alongside the number — something like "at least 2 of the 5 test
questions must have a near-duplicate templated sibling document" — so that
picking five topically-diverse-but-individually-easy questions can't quietly
satisfy a criterion that exists to catch confusion between similar documents.
The lesson from this whole unit wasn't "my system is broken," it was "a target
is only as good as the questions you test it with," and that's a property of
`questions.py`, not of `criteria.md`'s numbers — I'd have caught it earlier if
I'd asked myself in Milestone 2 *which* of my five questions was doing the
work of stress-testing each criterion, instead of just picking five specific,
answerable facts and moving on.
