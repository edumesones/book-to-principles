# Mode 5 — Dictionary (merge several books into one shared lexicon)

A lexicon per book does not scale: each book adds its own always-on block, authors name the same decision
differently (*DRY* ≈ *information leakage*), and some contradict each other (*crash early* vs *define errors
out of existence*). Mode 5 merges per-book lexicons into **one dictionary organised by the kind of decision it
governs**, with one always-on block. Per-book lexicons (Mode 2 reports or `leading-words.md`) are its raw
material; per-book skills become optional.

**Trigger:** "build the dictionary", "merge these lexicons", or Mode 1/2 finishing on a second book while a
dictionary exists ("add to the dictionary").
**Input:** two or more per-book lexicons (each term with Behaviour and Displaces).
**Output:** the `engineering-lexicon` skill (default slug), installed like any generated skill (Steps 5–10).

---

## D1 — Normalise the inputs

Put every kept term from every book into one list with: term · author · score (N/D/P/B per axis) · phase ·
Behaviour · Displaces. If an input lexicon lacks per-axis scores, estimate them and mark the estimate.
If a book has two lexicons for the same edition (e.g. two independent runs), take the union; a term kept by
only one run keeps its score but is flagged `single-run`. A term's **combined score** is the mean of its total
across the runs that kept it; a concept's combined score is the highest among its merged terms.

## D2 — Group into concepts

A **concept** is one decision. Two terms belong to the same concept **only if their Behaviour lines drive the
same next action and their Displaces lines override the same default**. Overlapping but different actions
stay separate and point at each other (`Not to be confused with`). Author opposites are never merged — they
become a conflict (D4).

## D3 — Pick the canonical term

Per concept, the canonical term is the one the agent should *say*. Choose by, in order:
0. **polarity first**: the canonical term names the practice to *do* (strategic programming, deep module), never
   the anti-pattern (tactical programming, shallow module); the anti-pattern becomes an alias used in reviews;
1. highest **Prior-likely** (the phrase most widely used outside the book — the prior is keyed on it);
2. then highest **Behaviour-changing**;
3. then the higher frequency in its source.
All other terms become **aliases** with their author. Record the reason in one clause. Never invent a new
phrase: the canonical term is always an author's exact wording.

## D4 — Resolve conflicts

A conflict is two concepts whose triggers overlap and whose Behaviours point in opposite directions. Keep
both, and give each a **`When it wins`** line keyed on the distinguishing condition (e.g. *crash early* when
the state is impossible or corrupt; *define errors out of existence* when the case is a normal edge the
operation can absorb). If no condition separates them, say so and prefer the one with the higher combined
score — never silently drop one.

## D5 — Assign the typology

Exactly one typology per concept (phase stays a tag, possibly two):

| Typology | Governs decisions about |
|---|---|
| `delivery` | how work is sliced, sequenced, prototyped, estimated and how much quality is enough |
| `modularity` | boundaries, what a module hides, coupling, duplication of knowledge, layers |
| `errors` | failure handling, contracts, assertions, exceptions |
| `testing` | tests, debugging and how correctness is established |
| `readability` | comments, names, obviousness, domain vocabulary |
| `maintenance` | entropy, refactoring, when to invest in design over time |
| `performance` | where to spend effort on speed: critical path, measurement, algorithmic order |

A concept that fits none goes to `other` with a one-line justification; more than three `other` entries means
the table needs a new typology — propose it, do not force.

## D6 — Write `dictionary.md`

Grouped by typology, then by combined score. Cap: **60 concepts**; above it, cut the lowest combined scores
first, `single-run` concepts before the rest. One entry per concept:

```markdown
### <Canonical term>  `modularity` · phases: design (+review) · <Author>
- **Aliases**: <term> (<Author>), …   *(omit if none)*
- **Behaviour**: <next action, ≤ 25 words>
- **Displaces**: <default it overrides>
- **Say it as**: "<imperative prompt fragment containing the canonical term>"
- **When it wins**: <condition> — vs <conflicting concept>   *(only for conflicts)*
- **Not to be confused with**: <neighbour> — <difference in one clause>   *(omit if none)*
```

Paraphrase only; every formulation ≤ 25 words; never copy book text.

## D7 — The always-on block (`snippets/claude-md.md`)

**12 concepts**, ≤ 300 tokens, chosen by combined score with **at least one per typology** that has a concept
scoring 8. Ties for the last slots are broken by the D3 order (polarity, Prior-likely, Behaviour-changing). One line each: `**canonical term** — <Behaviour, compressed>`. If a chosen concept has a conflict,
compress its `When it wins` into the same line. Close with the two lines required by Step 7.4 (the
`[term]`-per-plan-step line is mandatory — the passive form did not change plans in echo tests).

## D8 — Also write

- `SKILL.md` (≤ 1,500 tokens): the block's 12 lines, the typology table, when to load `dictionary.md`.
- `sources.md`: each book (title, author, edition) and how many concepts it contributed.
- `decisions.md`: every merge (D2), canonical choice (D3) and conflict resolution (D4), one line each with
  its reason — this is what a second, blind run is compared against.
- `snippets/grill.md` and `snippets/review.md` as in Step 7.4, drawing from all typologies; `review.md` runs the
  `Displaces` and `When it wins` lines of `dictionary.md` (Mode 5 has no principles.md/smells.md).

## D9 — Verify blind, then echo test

Run D1–D8 twice, independently (two agents that cannot see each other's output). Compare `decisions.md`:
the concepts both runs merge the same way and the canonical terms both pick are stable; every disagreement is
a judgement call to settle explicitly (and a hint that a rule above is under-specified). Then run the echo
test (Step 9.7) with the dictionary's block against a single-book block on the same design task.
