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

This is a retrieval-augmented question answerer for the `campus_life` corpus:
88 short posts about student life at a university — dining hall wait times,
housing quirks, course workloads, and the administrative rules nobody explains
properly (add/drop deadlines, parking permits, meal plan changes). Ask
something one of those posts covers, like how fast the west lot parking
permits sell out, and it pulls the matching chunk, names the file it came
from, and answers off that. Ask about something the corpus doesn't cover and
it refuses instead of guessing.

## Chunking Strategy

**Chunk size:** 600 characters (packed by paragraph, not a fixed window)
**Overlap:** 0

Before touching anything I ran the starter's `fallback_split` against all
three corpora to see what it actually did. `campus_life` came out 88
documents to 88 chunks — nothing splits, shortest chunk 178 characters,
longest 549. `city_guides` went from 14 documents to 51, cutting straight
through labelled sections. `advice_threads` produced a 2-character chunk, the
leftover tail of a document that didn't divide evenly into 800-character
windows. None of that is a bug. It's just what a fixed window does when it
doesn't know where a sentence ends.

The 1:1 result for `campus_life` seemed too clean to just accept, so I went
and checked why. Turns out the corpus is already hand-split into single-topic
files — a hall's laundry situation gets its own `_laundry.txt`, its noise
situation its own `_noise.txt`, both separate from the hall's overview file.
One file already equals one complete thought here. My chunker doesn't have to
create that; it just has to not break it.

So I swapped fixed-window slicing for paragraph packing. Each document gets
split on blank-line paragraph breaks, then consecutive paragraphs get merged
back together as long as the running total stays under `CHUNK_SIZE`. I picked
600 for that number — just above the longest real document at 549 — which
makes it more of a safety cap than an actual target: almost every file still
ends up as one chunk, and the packing logic only matters if some future
document ends up covering two topics at once. Overlap is 0 because there's no
seam here to patch in the first place; packing never cuts inside a paragraph,
so there's nothing lost at a chunk boundary for overlap to restore.

I thought about going further and splitting every document down to one chunk
per paragraph — `housing_innisfree_hall.txt`'s "good," "bad," and
laundry/noise lines would each become their own chunk. Decided against it.
Those exact facts already live in their own dedicated files elsewhere in the
corpus, written better than a fragment of the overview post would be.
Splitting further would just give me a worse copy of a chunk I already have.

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

Each one stands alone. No sentence gets cut off at either edge, and each one
names its own topic — the course, the deadline, the dining hall, the dorm —
instead of relying on something said in a chunk before it.

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

The two groups don't overlap. Every in-corpus question landed at 0.412 or
below, every out-of-scope question at 0.825 or above, so there's a clean gap
between those two numbers. 0.6, the starter's default, sits right in the
middle of it, so I left it where it was. Moving it would only have mattered if
the two groups had been crowding each other.

One thing that actually surprised me: I figured "How do I write a for loop in
Rust?" would be the closest call, since `campus_life` has real CS course
documents in it (CS 210, CS 340). Instead it came back as the furthest away of
all ten questions, at 0.896 — closer to `course_hist_118_exams.txt` than to
anything CS-related. Whatever language a Rust tutorial uses, it apparently
isn't close enough to how my documents talk about programming courses for that
to matter.

## How I Used AI

**1.** Before touching `chunker.py` I asked Claude Code to check what the
starter's fixed-window chunker actually did to each corpus instead of just
trusting the docstring's claim about it. It ran `fallback_split` against all
three and reported real numbers: `campus_life` went 88 documents to 88 chunks,
`city_guides` went 14 to 51 and cut straight through labelled sections, and
`advice_threads` left a 2-character chunk dangling off the end of a document.
That last one is what got me to stop treating campus_life's 1:1 result as
luck — I went and opened `housing_innisfree_hall.txt` next to its `_laundry`
and `_noise` sibling files myself and saw that the corpus is already
hand-split into single-topic files. So the plan changed from "shrink the
fixed window" to paragraph packing with a size cap, since a fixed window was
solving a problem this particular corpus didn't have.

**2.** For Milestone 4 I had it run my five `QUESTIONS` and all five
`OUT_OF_SCOPE` questions through retrieval and print the best distance for
each, before I touched the default cutoff. I'd already guessed, in
`criteria.md`, that "How do I write a for loop in Rust?" would be the risky
one since the corpus has real CS course material in it. The number came back
at 0.896 — the furthest away of all ten questions, not the closest — with a
clean gap between 0.412 (the worst in-corpus distance) and 0.825 (the best
out-of-scope one). So the plan changed from "tune THRESHOLD" to "confirm the
default's fine and explain why," and I left my wrong guess sitting in
`criteria.md` rather than fixing it after the fact, since the gap between what
I expected and what actually happened seemed worth keeping.

**3.** For unit 2's improvement I asked it, before writing any code, whether
adding BM25 hybrid search would actually change my numbers, given that the
before-run had already scored 5/5 on everything. Its reasoning: the real
Kestrel Commons document was already the closest semantic match by a wide
margin (0.191 against 0.33+ for the next-closest distractor), so re-ranking a
top-5 that already had the right answer in it had nothing to fix. Prediction
was "probably no measurable change, but expect the distractors themselves to
shift." That's basically what happened — same verdicts on every criterion, a
different mix of distractors underneath. What neither of us saw coming until
we actually looked at the retrieved sources was which distractor showed up:
`transit_walking.txt`, pulled in for no better reason than that it happens to
say "Kestrel Commons" once, in a sentence about how long it takes to walk
there. That finding went into "What's Still Broken" because I noticed it in
the actual output, not because either of us predicted it going in.

## Stretch Feature (declared before building it)

**Attempting: a second measured improvement — a second chunking strategy.**

The Improvement section below covers hybrid search. As a second, independent
change from the Milestone 4 menu, I'm going to index `campus_life` a second
way: the same paragraph-packing algorithm from `chunker.py::split_documents`,
but with `CHUNK_SIZE` dropped from 600 to 250, stored as a second Chroma
variant (`small_chunks`) rather than replacing the existing index. At 600, the
packer almost never actually splits anything, since no document in this
corpus exceeds 549 characters — Milestone 3's whole finding was that one file
already equals one chunk here. At 250, that stops being true: multi-paragraph
files like `housing_innisfree_hall.txt` (519 characters, four separate facts)
will actually get cut into two or more pieces.

This is meant to test two things Diagnoses and Verdicts already flagged: (1)
whether smaller, single-fact chunks make criterion 4 (self-contained chunks)
even cleaner, since the "busiest" chunk I found — `housing_innisfree_hall.txt`
— is exactly the kind of file a 250-character cap would split apart, and (2)
whether that same splitting risks cutting a fact's sentence in half across a
chunk boundary, which would hurt criterion 1 instead. I'm running this with
`RETRIEVAL_MODE` left at `"semantic"` (not combined with hybrid search) so
chunk size is the only variable that changed from the original before-run —
a third, independent comparison against `results/run_2026-09-28_0050_before.md`,
not a combination with the hybrid-search after-run.

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

Criteria 3 and 4 show the same number in all three columns, and it's not
laziness — they're both measured by code that doesn't change between runs,
even though it's different code in each case. Criterion 3 is a single pass of
`store.py::search` against a fixed cutoff in `gate.py::check`; asking the same
question three times doesn't change a cosine distance. Criterion 4 never
touches `run_eval.py` at all — `chunker.py`'s `split_documents` spits out the
same 88 chunks from the same 88 documents no matter how many times you sample
them. The only thing that actually moves between runs is
`generate.py::answer_from_chunks`, since those are uncached model calls, which
is why criteria 1, 2, and 5 are the ones the instructions expect to shift.
Mine just didn't.

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
| 1 | Retrieved chunk contains the answer (target: 4 of 5) | MET | For each question I checked the actual "Sources retrieved" list against the document I already know holds the fact, rather than just trusting whatever the model wrote. The right source showed up in the top-5 every time, for all 5 questions, and since retrieval doesn't change between calls, that's true for all 3 runs too. |
| 2 | Every answer names a source (target: 5 of 5) | MET | Read all 15 answers (5 questions × 3 runs) as plain text, and every one had an explicit citation in it. This target was already 5 of 5, so there's no rounding up to do — either all 15 had a source or they didn't. |
| 3 | Gate stops out-of-corpus questions (target: 4 of 5) | MET | One pass of the 5 `OUT_OF_SCOPE` questions through the gate — it's deterministic, so one pass is all there is. All 5 got refused, distances ranging 0.825 to 0.934, nowhere near the 0.6 cutoff either way. |
| 4 | Chunks read as complete, self-contained thoughts (target: 4 of 5) | MET | Actually ran the "could someone answer using only this" test against all 5 sampled chunks instead of eyeballing them. `dining_pellew_dining_hall_followup.txt` came closest to failing — it reads like a reply to something unstated ("matches what I've seen") — but it still names the hall and gives the number directly, so on its own it's enough. All 5 passed. |
| 5 | Source named is the source that actually backs the answer (target: 4 of 5) | MET | Checked, for every run, whether the document that got cited actually contains the fact being quoted, not just whether a source was named at all. Kestrel Commons was the real test here, with 4 near-duplicate dining-hall documents sitting in the same top-5 — the model picked the right one all 3 times. |

None of these were broken. Each one turned out to be measurable exactly as
written in `criteria.md` — a fixed document to check against, a literal
string to look for, a distance to compare, a five-chunk sample, a
source-to-text match — so I'm not revising anything this unit. A measurement
problem would look different: if "contains the answer" meant something
different to me on two separate reads, or I scored the same question two
different ways on two different days, that's when a criterion needs rewriting
rather than just a fix. That's not what happened. Every number came out the
same way for a reason I can actually point to.

## Diagnoses

Nothing was missed. All five criteria came out MET across all three runs — no
question failed, no source got misattributed, no out-of-scope question snuck
through. So there's no failure sitting in any of the five stages waiting to be
traced, and nothing to diagnose in the normal sense of the word.

I'm suspicious of that, not proud of it. Clearing every criterion on the first
attempt usually just means the criteria were easy, not that the system is
great, so rather than stop there I went back and looked at how hard each
criterion had actually been tested — not whether it passed, but whether
passing it meant anything.

Here's what I found: criteria 1 and 5 both exist because of one specific
risk. `campus_life` has six dining-hall documents and six housing-hall
documents built off the same template, so an embedding could plausibly
confuse "Kestrel" for "Pellew," or the model could cite the wrong hall even
with the right chunk sitting in front of it. Only one of my five questions —
Kestrel Commons — actually has a near-duplicate sibling to run into that
problem. The other four (add/drop deadline, printing quota, parking permits,
library hours) are each the only document on their topic in the whole corpus,
so there's nothing around to confuse retrieval or generation with — getting
those right is close to automatic. A target that's only put to the test by one
question in five isn't really running at 4-of-5 confidence. Four fifths of the
time it passes by default, before the mechanism it exists to catch even shows
up.

**What I'd tighten, and to what:** criterion 5, the one about whether the
cited source is the source that actually backs the answer. Instead of "at
least 4 of 5 test questions," I'd write "at least 4 of 5 test questions, at
least 2 of which have a near-duplicate templated sibling document in the
corpus." That's a requirement on how the test questions get picked, not just a
harder number, and it stops four easy questions from carrying one hard one to
a passing average. I'm not writing this into `criteria.md` itself — nothing
about the criterion was broken, it measured exactly what it said it would. I
just picked a test suite that didn't push on it as hard as it could have, and
that belongs under "What I'd Do Differently" below, not as a revision.

## The Improvement

**What I changed:** I added hybrid search. `store.py::search` now pulls a
wider pool of 15 semantic candidates, and when `config.RETRIEVAL_MODE` is set
to `"hybrid"` (via `AI201_RETRIEVAL_MODE=hybrid`), hands that pool to
`store.py::_hybrid_rerank`, which blends cosine similarity with BM25 keyword
overlap 50/50 (`rank-bm25` was already sitting in `requirements.txt`) before
cutting it down to `top_k`. The distance on each `Result` is still the real
cosine distance no matter which mode is on, so `gate.py::check`'s 0.6 cutoff
hasn't changed meaning — hybrid mode only touches which chunks make the cut
and what order they come back in.

**Why I picked it:** This goes straight at the mechanism from Diagnoses.
`campus_life` has six dining halls and six housing halls all written off one
template, so a question about "Kestrel Commons" risks its exact name getting
drowned out by boilerplate wording it shares with Halden, Pellew, and
Ridgeway — "matches what I've seen," "the salad bar wilts," that kind of
thing. BM25 is good at exactly the thing semantic search is weak at, exact
token matches, so combining the two should help right where the
near-duplicate-template risk actually lives, without costing anything
semantic search already handles fine.

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

The verdicts match Run Log — Before question for question, exactly. What's
not identical is the retrieval underneath — hybrid re-ranking changed which
distractors came back, and I can show it. Take the Kestrel Commons question:

```
Before: dining_halden_hall_followup.txt, dining_kestrel_commons.txt, dining_kestrel_commons_followup.txt, dining_pellew_dining_hall_followup.txt, dining_the_ridgeway_cafe_followup.txt

After:  dining_kestrel_commons.txt, dining_kestrel_commons_followup.txt, dining_the_ridgeway_cafe_followup.txt, dining_verrill_street_grill_followup.txt, transit_walking.txt
```

(`store.py::search` into `store.py::_hybrid_rerank`.) `transit_walking.txt`
is the interesting one here — it literally contains the sentence "Morrow
House to Kestrel Commons: 7 minutes," so BM25 rewarded it for saying "Kestrel
Commons" exactly right, never mind that the document has nothing to do with
wait times and is entirely about how long it takes to walk somewhere. BM25
has no concept of a document being *about* a thing versus just mentioning it
in passing.

**Did it help?** No, not on this test suite — and I mean that precisely, not
as a shrug. Every criterion landed at the exact same score, run for run, both
before and after (compare the two tables). The real Kestrel Commons documents
were never in danger of leaving the semantic top-5 to begin with — Kestrel's
own document sat at distance 0.191, the biggest margin of any of my five
questions — so there wasn't a wrong answer sitting around for re-ranking to
correct, and nothing for it to actually improve. All it changed was which
distractors rode along for the ride, occasionally for a worse one
(`transit_walking.txt` names the entity and answers nothing), with no effect
at all on what the model ended up citing.

This lines up exactly with the Diagnosis. I picked hybrid search because
criteria 1 and 5 exist to catch near-duplicate-template confusion, and only
one of my five questions actually has a near-duplicate sibling — and that
question was already handled fine under semantic-only search. The fix itself
is sound and the reasoning behind it holds up. My test suite just doesn't
have a question hard enough to actually need it, which is the same gap
Diagnoses pointed at, seen from a different angle.

## What's Still Broken

Nothing is missed, either before or after the fix. But "nothing missed" isn't
the same thing as "nothing left" — two real gaps are still sitting here:

1. **My test suite still doesn't push hard enough on criteria 1 and 5.** Only
   one question out of five has a near-duplicate sibling in the corpus, so a
   clean 5/5 doesn't prove the system handles that risk reliably — it proves
   the system handled it once. I'd fix this by adding two more questions, one
   about a different dining hall and one about a different housing hall,
   picked specifically because each has a near-identical sibling document,
   then running both retrieval modes against the bigger set.

2. **The hybrid re-ranker can't tell a document that's about something from
   one that just mentions it.** `transit_walking.txt` got into the Kestrel
   Commons top-5 under hybrid mode for no better reason than saying "Kestrel
   Commons" once, in a sentence with nothing to do with wait times. It never
   bumped the real answer out of my runs, but a bigger corpus with more
   incidental name-drops could let something like that crowd out the actual
   answer eventually. My fix would be a minimum semantic similarity floor
   before BM25 gets any say at all — a document has to already be somewhat
   relevant in meaning before keyword overlap can move it up, rather than the
   flat 50/50 split I used here.

I stopped here because both of these are about making the test harder, not
because I ran into a live failure. It's a scope stop, not an ideas stop — the
before and after run logs plus this write-up are the actual record of where
things stand, and I'd rather leave that record honest than pad it out with a
fix for a problem I haven't actually hit yet.

## What I'd Do Differently

Criterion 5's number was fine. What wasn't fine was how I picked the
questions meant to test it. Knowing what I know now, I'd write the selection
rule into `criteria.md` right alongside the target — something like "at least
2 of the 5 test questions have to have a near-duplicate templated sibling
document" — so five topically different but individually easy questions can't
quietly satisfy a criterion built to catch confusion between similar
documents. The real lesson from this unit wasn't "my system is broken." It
was that a target is only as good as the questions you point at it, and
that's a property of `questions.py`, not of the numbers in `criteria.md`. I'd
have caught this back in Milestone 2 if I'd asked myself which of my five
questions was actually doing the work of stress-testing each criterion,
instead of picking five specific, easily-answerable facts and calling it
done.
