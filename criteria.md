# Acceptance criteria — The Unofficial Guide

Five criteria that say what "working" means for this system, written in unit 1
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"Retrieval works"* is an opinion. *"For at
least 4 of my 5 test questions, the top results include a chunk containing the
answer"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter or looser one. A reason that says something about your corpus or your
pipeline earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## 1. Retrieved chunks contain the answer

For at least 4 of my 5 test questions, the retrieved chunks include one that
contains the answer.

**Why this target:** My Kestrel Commons wait-time question is the one I expect
to be hardest: `campus_life` has six dining halls, and their documents share
almost identical sentence templates ("wait times… matches what I've seen,"
"salad bar wilts after 1:30") with only the hall name and numbers changed.
That boilerplate similarity could make the embedding for "Kestrel" sit close
to "Pellew" or "Halden," so top-5 might pull in the right *shape* of chunk
from the wrong hall. My other four questions (add/drop week, printing quota,
parking permits, library hours) each come from a single document with no
near-duplicate sibling, so I expect those to retrieve cleanly.

---

## 2. Every answer names a source

Every answer the system produces names at least one source document.

**Why this target:** This one I hold to 5 of 5, not 4 of 5, because it isn't
really a retrieval-quality question — `generate.py`'s prompt template appends
a `Source:` line by construction whenever the gate lets a question through, so
it's a code guarantee rather than something the model can flake on. I watched
it hold even under pressure: my Kestrel Commons question pulled back four
other dining halls' near-identical documents alongside the two real Kestrel
ones, and the answer still cited `dining_kestrel_commons.txt` specifically
rather than any of the distractors. The only way an answer has zero sources is
the gate refusing first — and a refusal isn't the "answer" this criterion is
about, since criterion 3 covers refusals separately.

---

## 3. The relevance gate stops out-of-corpus questions

When I ask a question my documents clearly don't cover, the relevance gate
stops it and the system returns "I don't have enough information about that" —
in at least 4 of 5 tries.

<!-- The five questions are the ones in `OUT_OF_SCOPE` at the bottom of
     `questions.py`, and `run_eval.py` puts them through the gate and writes
     what happened into your run log. Swap them for your own if you'd rather —
     just keep five of them, or the "4 of 5" above has nothing to be 4 of. -->

**Why this target:** I'm keeping this at 4 of 5, not 5 of 5, because one of my
five `OUT_OF_SCOPE` questions is "How do I write a for loop in Rust?" and my
corpus isn't entirely unrelated to programming — it includes CS 210 and CS 340
course documents. Before measuring anything, that felt like the one question
where the embedding distance might land closer to my cutoff than the other
four (a capital city, a diesel engine, a sports result, a drug dosage), so I
didn't want to commit to a clean sweep in advance.

---

## 4. Chunks read as complete, self-contained thoughts

At least 4 of 5 sampled chunks contain no sentence cut off at either edge, and
each names its own topic (which hall, which course, which deadline) rather
than leaning on whatever chunk came before or after it.

**Why this target:** After switching to paragraph packing, I checked all 88
chunk lengths — they run 178 to 549 characters, the same range as the raw
documents, because campus_life files are already single-topic and packing
rarely has more than one paragraph run to merge. The one place I'd expect a
miss is a file like `housing_innisfree_hall.txt`, which packs four distinct
facts (room layout, AC, laundry price, noise level) into a single
519-character chunk. It's still self-contained — nothing is cut off — but
it's busier than a single-fact chunk like `admin_printing_quota.txt`, so I'm
not assuming a clean 5 of 5 before I've sampled more than five chunks.

---

## 5. The source named is the source that actually backs the answer

For at least 4 of my 5 test questions, the source cited in the answer is a
document whose text contains the specific fact quoted — not just any document
that happened to be in the top-k results.

**Why this target:** Criterion 2 only checks that *some* source gets named;
it says nothing about whether it's the right one, and campus_life makes that
an easy thing to get wrong on purpose — six dining halls, six housing halls,
all written from the same template with only the names and numbers swapped.
I already watched this almost go sideways: my Kestrel Commons question pulled
back `dining_halden_hall_followup.txt`, `dining_pellew_dining_hall_followup.txt`,
and `dining_the_ridgeway_cafe_followup.txt` in its top-5 alongside the two real
Kestrel documents, and the answer still correctly cited
`dining_kestrel_commons.txt`. That's one question out of five confirmed, which
is why I'm setting the target at 4 of 5 rather than assuming the other four —
none of which have a near-duplicate sibling — will behave the same way under
a test I haven't actually run on them yet.
---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 2 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 1. Retrieved chunks contain the answer

         For at least 4 of my 5 test questions, the retrieved chunks include
         one that contains the answer.

         **Why this target:** ...

         > **Revised in unit 2:** For at least 4 of 5 questions, the top three
         > results contain the answer.
         >
         > **Why revised:** I couldn't judge "the chunks include one that
         > contains the answer" the same way twice — I scored two questions
         > differently on Monday than on Wednesday. The new version is
         > something I can actually check.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said 4 of 5 but got 2 of 5, so 2 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.

     The whole reason the originals stay visible is so someone can see what you
     said before you knew the answer.
     ───────────────────────────────────────────────────────────────────────── -->
