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

**Why this target:** The Kestrel Commons question worries me most. `campus_life`
has six dining halls and they're basically clones of each other — same
sentence templates, same "matches what I've seen" phrasing, just the hall
name and numbers swapped. If the embedding can't tell "Kestrel" apart from
"Pellew" or "Halden" well enough, top-5 could hand back a chunk that looks
right but is about the wrong building. The other four questions don't have
that problem — add/drop week, printing quota, parking permits, and library
hours each live in one document with nothing else like it in the corpus, so
I'd be surprised if those didn't retrieve cleanly.

---

## 2. Every answer names a source

Every answer the system produces names at least one source document.

**Why this target:** This is the one I set at 5 of 5 instead of 4. Naming a
source isn't really about retrieval quality — `generate.py`'s prompt template
appends a `Source:` line by construction any time the gate lets a question
through, so it's baked into the code rather than something the model could
forget to do. It held up even when I stress-tested it: the Kestrel Commons
question came back with four other near-identical dining hall documents mixed
in, and the answer still pointed at `dining_kestrel_commons.txt` specifically,
not one of the distractors. About the only way an answer ends up with zero
sources is the gate refusing before generation even runs, and that's
criterion 3's territory, not this one.

---

## 3. The relevance gate stops out-of-corpus questions

When I ask a question my documents clearly don't cover, the relevance gate
stops it and the system returns "I don't have enough information about that" —
in at least 4 of 5 tries.

<!-- The five questions are the ones in `OUT_OF_SCOPE` at the bottom of
     `questions.py`, and `run_eval.py` puts them through the gate and writes
     what happened into your run log. Swap them for your own if you'd rather —
     just keep five of them, or the "4 of 5" above has nothing to be 4 of. -->

**Why this target:** I kept this one at 4 of 5. One of my `OUT_OF_SCOPE`
questions asks how to write a for loop in Rust, and `campus_life` isn't a
totally programming-free corpus — CS 210 and CS 340 are both in there. Before
I'd measured anything, that felt like the question most likely to land close
to the cutoff, closer than a question about a capital city, a diesel engine, a
sports result, or a drug dosage. So I didn't want to bet on all five clearing
it easily.

---

## 4. Chunks read as complete, self-contained thoughts

At least 4 of 5 sampled chunks contain no sentence cut off at either edge, and
each names its own topic (which hall, which course, which deadline) rather
than leaning on whatever chunk came before or after it.

**Why this target:** Once I switched to paragraph packing I checked the
lengths on all 88 chunks — 178 to 549 characters, basically the same spread
as the raw documents, since `campus_life` files are already single-topic and
there's rarely more than one paragraph run for the packer to merge. If
anything's going to trip this up it's a file like `housing_innisfree_hall.txt`,
which crams four separate facts — layout, AC, laundry price, noise — into one
519-character chunk. Nothing's cut off, so it's technically self-contained,
but it's a busier read than a one-fact chunk like `admin_printing_quota.txt`.
I don't want to assume 5 of 5 off a sample size of five.

---

## 5. The source named is the source that actually backs the answer

For at least 4 of my 5 test questions, the source cited in the answer is a
document whose text contains the specific fact quoted — not just any document
that happened to be in the top-k results.

**Why this target:** Criterion 2 only checks that some source gets named, not
whether it's the correct one, and this corpus makes that distinction easy to
get wrong on purpose — six dining halls and six housing halls, all copy-pasted
off the same template with just the names and numbers changed. I already saw
this nearly go wrong once: the Kestrel Commons question pulled
`dining_halden_hall_followup.txt`, `dining_pellew_dining_hall_followup.txt`,
and `dining_the_ridgeway_cafe_followup.txt` into its top-5 right alongside the
two actual Kestrel documents, and the answer still landed on
`dining_kestrel_commons.txt` correctly. That's one question confirmed out of
five. I'm not ready to assume the other four, none of which have a
near-duplicate twin, will hold up the same way under a test I haven't run on
them.

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
