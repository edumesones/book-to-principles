---
name: book-to-principles
description: "Distills a book or document set (PDF, EPUB, DOCX, HTML, Markdown, TXT, RTF, MOBI/AZW) into a *shared language* between you and your coding agent: leading words (Leitwörter) that steer the agent's reasoning, decision principles tied to those words, anti-pattern smells for review, and paste-ready snippets for CLAUDE.md / AGENTS.md. Optionally binds the book's vocabulary to a project's own domain language (ubiquitous language). Use when the user wants an agent to *think like the author* while planning, implementing or reviewing — not to look the book up."
---

<!--
Cross-agent notes (informational; ignored by host agents):
  - Fork of virgiliojr94/book-to-skill (MIT). Extraction pipeline (scripts/extract.py,
    Steps 0–2.6) is reused unchanged; everything from Step 3 on is different.
  - `allowed-tools` intentionally omitted (agent-neutral). Needs shell + file read/write.
  - Argument hint: <path-to-document-folder-or-glob>... [skill-name-slug] [--bind <project-root>]
-->

# Book-to-Principles

Turn written expertise into a **language you and your agent share**, not a knowledge base to query.

## Philosophy

A coding agent starts every session with no memory of you, but with the whole canon of software engineering in its prior. The cheapest way to change how it reasons is not to explain a concept — it is to **name it with the word the author used**. "Treat this as a tracer bullet" moves the agent from building layer-by-layer to building one thin end-to-end path; "keep this a deep module" moves it away from shallow pass-through wrappers. These are *leading words*: dense terms that encode a procedure and a value judgement in a couple of tokens. Repeated in a prompt, a skill or `CLAUDE.md`, the agent starts echoing them in its own reasoning — and behaving accordingly.

book-to-skill answers "what does the book say about X?". **book-to-principles answers "how do I get my agent to act the way the author would, in as few words as possible?"** The output is small by design: a lexicon, the decision rules behind each word, the smells that signal a violation, and prompt snippets — everything sized to sit permanently in an agent's context without eating its smart zone.

Three rules govern everything below:
- **A leading word must change behaviour.** A term that only names a thing is glossary material; a term that implies a different next action is a leading word. Only the second kind makes the cut.
- **Preserve the author's exact naming.** "Tracer bullet" is not "vertical slice" is not "walking skeleton" — they overlap, but the author chose one for a reason, and the agent's prior is keyed on the exact phrase.
- **Never copy raw text.** Formulations are paraphrased and compact. The skill is your notes on the author's vocabulary, not the book.

---

## Modes of Operation

### 1. Full Distillation (default)
**Trigger:** document/folder/glob paths, no special instructions.
**Action:** Steps 0–9. **Output:** `SKILL.md`, `leading-words.md`, `principles.md`, `smells.md`, `snippets/`.

### 2. Lexicon Only
**Trigger:** "just the words", "lexicon only", "analyze".
**Action:** Steps 0–4, then print the candidate lexicon (Step 4 report) and stop. Nothing written.

### 3. Bind to Project
**Trigger:** `--bind <project-root>`, "bind this to my repo", or a slug of an existing principles skill plus a project path.
**Action:** Skip extraction; load the existing skill; run Step 8 only. **Output:** `bindings.md` + optional block appended to the project's `CLAUDE.md`/`AGENTS.md`/`CONTEXT.md` (only with explicit confirmation).

### 4. Fold-in
**Trigger:** new sources + an existing principles skill slug/folder.
**Action:** Steps 0–4 on the new sources, then merge into the existing lexicon (Step 7 rules), regenerate `SKILL.md` and snippets.

---

## Skill Locations

Same resolution as book-to-skill: generated skills go to `~/.agents/skills/<slug>/` by default; Claude Code gets a verified symlink from `~/.claude/skills/<slug>/`; Hermes and OpenClaw use their own roots; project-local roots only when asked. The converter's own `scripts/extract.py` is located with the same candidate list, with `book-to-principles` in place of `book-to-skill` in every path.

---

## Step 0 — Out-of-scope check

No arguments → stop:
> "book-to-principles needs a document path, folder or glob. Usage: `book-to-principles <paths>... [slug] [--bind <project-root>]`"

Parse: `INPUT_PATHS`, optional `SKILL_NAME` (last arg that is not a path and looks like a slug), optional `--bind PROJECT_ROOT`. If `SKILL_NAME` already exists under `SKILLS_HOME` and there are new sources → Mode 4; if `--bind` and no sources → Mode 3.

## Step 1 — Validate input
Identical to book-to-skill Step 1 (supported extensions, expand globs, fail clearly if nothing found).

## Step 1.5 — Content type
Ask once: technical / text-heavy / not sure → `BOOK_TYPE`. Same extractor choice as book-to-skill (Docling for technical). Most books worth distilling here are *text-heavy* (Pragmatic Programmer, Ousterhout, Evans, Fowler): say so and default to text unless the user objects.

## Step 2 — Extract
Run `scripts/extract.py $INPUT_PATHS --mode <BOOK_TYPE> --install-missing ask`, exactly as book-to-skill Step 2. Take `Workdir ->`, `Text ->`, `Meta ->` from the run output. Confirm the `SOURCE:` header matches what the user asked for.

## Step 2.5 — Cost estimate
Cheaper than book-to-skill: output is ~4–6K tokens total, not per-chapter. Report:
```
📖 Sources: <n> · ~<pages> pages · ~<N>K tokens extracted
💰 Estimated: input ≈ extracted × 1.2 (one structure pass + targeted probes) · output ≈ 6K
📁 Will write: SKILL.md · leading-words.md · principles.md · smells.md · snippets/{claude-md,grill,review}.md
➡  Proceed? ("lexicon only" to preview the words first)
```
Wait for confirmation.

## Step 2.6 — Probe, don't read
For sources > 50k tokens, never `Read` the whole `full_text.txt`. Find the ToC with `grep -n -E "^\s*(Chapter|CHAPTER|Part|PART)\s+[0-9IVX]+"`, pull sections with `sed -n 'a,bp'`, and — critically for this skill — **count candidate terms before believing them**:
```bash
grep -o -i -E "tracer bullet|deep module|ubiquitous language" "$FULL_TEXT_PATH" | sort | uniq -c | sort -rn
```
A term the author uses once in passing is not a leading word. Frequency across chapters is one of the selection signals in Step 4.

---

## Step 3 — Identify the author's vocabulary

Read the first ~8,000 characters plus the ToC to get title, author(s), domain and chapter map. Then, per chapter (targeted reads), list every **named concept**: terms the author capitalises, italicises, defines explicitly ("I call this…"), repeats across chapters, or puts in a heading. Record for each: term, chapter(s), rough frequency, one-line meaning.

Do not filter yet. Aim for 40–120 raw candidates for a full-length book.

---

## Step 4 — Score and select leading words

Score every candidate 0–2 on four axes. Keep candidates scoring **≥ 6/8**; cap the lexicon at **25–40 terms** (more dilutes the effect — every extra token competes for attention).

| Axis | 0 | 1 | 2 |
|---|---|---|---|
| **Named** — is it a proper phrase, not a description? | generic ("good tests") | recognisable but loose | the author's own coined/adopted phrase |
| **Dense** — how much procedure does the phrase encode? | a definition only | a rule of thumb | a whole method or a trade-off decision |
| **Prior-likely** — would a frontier model already know it? | obscure/new | known in niche | canonical (widely cited, decades old, in many other books) |
| **Behaviour-changing** — does saying it imply a *different next action* for a coding agent? | no action implied | nudges style | changes what gets built first, how it is split, or what is rejected |

Also tag each kept term with the **phase** where it bites: `plan` · `implement` · `review` · `refactor` · `design`. A balanced lexicon covers all five; if one phase is empty, say so rather than forcing a term in.

**Mode 2 stops here** with this report:
```
## Candidate lexicon — <Title> (<Author>)
| Term | Score (N/D/P/B) | Phase | Chapters | Freq | One-line meaning |
Rejected (score < 6): <comma list> — kept out because <one reason each, grouped>
Phases without a strong term: <list or "none">
```

---

## Step 5 — Skill name and destination

Slug: `{author-lastname}-principles` by default (`hunt-thomas-principles`, `ousterhout-principles`, `evans-ddd-principles`). Resolve `SKILLS_HOME` and host exactly as book-to-skill Step 5 (incl. the real-directory migration guard on Claude Code). If the slug exists: Fold-in / Overwrite / Rename.

## Step 6 — Directory
```bash
mkdir -p "$SKILLS_HOME/<slug>/snippets"
```

---

## Step 7 — Generate the lexicon and its supporting files

### 7.1 `leading-words.md` (~2,000–3,000 tokens)

One entry per kept term, ordered by phase then by score. **This is the heart of the skill**; every field has a job:

```markdown
## <Term>  `phase: plan` · score 8/8 · Ch 2, 7

- **Canonical**: <the author's formulation, paraphrased, ≤ 25 words — keep any distinctive wording the prior is keyed on>
- **Trigger**: <the situation in which the agent should reach for this — concrete, e.g. "a feature touches more than one layer or service">
- **Behaviour**: <what the agent does differently once the word is in play — the *next action*, not a definition>
- **Displaces**: <the default agent behaviour this word overrides, e.g. "scaffold all layers, integrate at the end">
- **Say it as**: "<a one-sentence prompt fragment the user can paste — imperative, contains the term>"
- **Echo test**: <the phrase or plan shape you expect to see in the agent's response if the word landed>
- **Not to be confused with**: <neighbouring terms from this or other books, and the difference in one clause> *(omit if none)*
```

Rules:
- `Behaviour` and `Displaces` are mandatory and must describe **actions**, never restate the definition. If you cannot write a `Displaces` line, the term is not a leading word — drop it.
- `Say it as` must be usable verbatim in a prompt and must contain the exact term.
- Keep the author's terminology in English even if the book is translated; add the original in parentheses if the source language differs.

### 7.2 `principles.md` (~1,500 tokens)

The author's decision rules, each anchored to one or more leading words, in the form a review agent can apply:

```markdown
## <Principle name (author's)>
**Rule**: When <situation>, <do X> rather than <Y>, because <author's reason>.
**Words**: <leading words this rule operationalises>
**Check**: <a yes/no question a reviewer asks of a diff or a plan>
**Exception**: <when the author says not to apply it> *(omit if none)*
```

Prefer rules with a *because*: the reason is what lets the agent generalise to cases the book did not cover. 10–25 principles. No prose paragraphs.

### 7.3 `smells.md` (~800–1,200 tokens)

Anti-patterns as **tells** — fast recognisers a review agent runs over code or plans:

```markdown
| Smell (author's name if any) | Looks like | Violates | Fix in the author's words |
```

Include the *tautological test* style smells if the book has them, and anything the author names explicitly. 10–20 rows.

### 7.4 `snippets/`

Three paste-ready artefacts. Each starts with a one-line comment naming the book so future readers know where the words come from.

- **`claude-md.md`** (≤ 300 tokens) — the block that goes into `CLAUDE.md` / `AGENTS.md`. Format: a heading `## Working vocabulary (<Author>)`, then 8–12 leading words, one line each: `**term** — <Behaviour line, compressed>`. Close with one sentence: "Use these terms in plans, commit messages and reviews; if a plan does not fit one of them, say which and why." The 300-token cap is a hard cap: this block is loaded on every session.
- **`grill.md`** (≤ 500 tokens) — 8–15 questions, phrased in the author's language, for a grilling/interview session before a large change. Each question names a leading word and asks the user to decide something ("Which single path is the tracer bullet for this feature, and what does 'it works' look like at the far end?"). Group by phase.
- **`review.md`** (≤ 500 tokens) — a prompt for a review agent: load `principles.md` and `smells.md`, run every `Check`, report findings as *judgement calls in the author's terms*, never as hard violations; a documented repo standard overrides the book.

### 7.5 Fold-in rules (Mode 4)
Re-score the union of old and new candidates; a term may enter, leave, or change phase. Never exceed the 40-term cap — a fold-in that adds must also drop, and the report lists both. Re-emit snippets from the new lexicon; never append to them.

---

## Step 8 — Bind to a project (optional; Mode 3 or offered after Step 7)

This is where the book's language becomes **your** language — the ubiquitous-language move from DDD, applied to the pairing of you, your repo and your agent. Offer once after a Full Distillation:

> "Bind this lexicon to a project? I'll read the repo's context files and interview you briefly to map the author's words onto your codebase. (path / skip)"

If accepted, with `PROJECT_ROOT`:

1. **Read what the project already says about itself** (read-only): `CLAUDE.md`, `AGENTS.md`, `CONTEXT.md`, `docs/adr/*`, `README.md`, top-level `docs/`. Note existing domain terms and any conventions that overlap or conflict with the lexicon.
2. **Propose a mapping per leading word** — at most one probe per term, e.g. `grep -rn -i "<term or its obvious synonyms>" --include=*.py --include=*.md` — to find where the concept already lives under another name.
3. **Interview the user, one question at a time, at most 10 questions.** Only ask what the repo could not answer. Each question is a decision, never a quiz: "In this repo, does *deep module* mean the `pipelines/` packages, the `services/` layer, or something else?", "Is there a term you already use for what the author calls a tracer bullet? If so we keep yours and note the alias." Stop when the mapped terms cover every `plan`/`review` word or the user says enough.
4. **Write `bindings.md`** in the skill folder:
```markdown
# Bindings — <slug> ↔ <project name>
| Author's term | Project term (keep if it exists) | Where it lives (path / grep) | Note |
Conflicts: <lexicon term vs repo convention, and which wins — repo wins by default>
Unbound: <terms with no counterpart yet>
```
5. **Offer, never assume, the context edit**: show the exact block (`snippets/claude-md.md` rewritten with the project's own terms substituted in and aliases in parentheses) and ask "Append this to `<PROJECT_ROOT>/CLAUDE.md`? (yes / show me / skip)". Append only on a bare `yes`; never overwrite; never touch files outside `PROJECT_ROOT`.

---

## Step 9 — Generate the master `SKILL.md`

**Hard budget: 1,500 tokens.** This file is model-invoked and loads whenever the agent thinks the author's lens applies, so it must be short and front-loaded.

```markdown
---
name: <slug>
description: "Shared engineering vocabulary from \"<Title>\" by <Author>. Use when planning, implementing or reviewing code and the user (or the task) invokes <3–5 top leading words>, or asks to work 'the <Author> way'."
---

# <Title> — working vocabulary
**Author**: <Author> | **Terms**: <N> | **Bound to**: <project or "no project yet"> | **Generated**: <YYYY-MM-DD>

## Use these words
<!-- The 8–12 highest-scoring terms, one line each, identical to snippets/claude-md.md.
     This is the part that must survive any compaction. -->

## When to load more
- Planning a change bigger than one session → read `snippets/grill.md`, ask the user those questions first.
- Reviewing a diff or a plan → read `principles.md` + `smells.md`, follow `snippets/review.md`.
- A term is unclear or contested → read its entry in `leading-words.md` (`Not to be confused with`).
- Working inside the bound project → `bindings.md` maps the author's words to the repo's own; **the repo's term wins**.

## How to speak
State which leading word governs a decision before making it ("This is the tracer bullet: …"). If none fits, say so and name the closest. Never pad prose with the vocabulary — a leading word earns its place by changing an action.

## Scope & limits
Vocabulary and decision rules only, synthesised from the book — not the text, not a summary. Where a repo documents a different convention, the repo wins.
```

## Step 9.5 — Scan
Run `tools/scan_generated_skill.py "$SKILLS_HOME/<slug>"` exactly as in book-to-skill. Stop on non-zero.

## Step 9.7 — Echo test (recommended, 5 minutes)

Before reporting success, verify the lexicon actually steers. In a **fresh** session on any repo:
1. Ask for a plan for a medium feature with `snippets/claude-md.md` in context; save the response.
2. Ask the same, same repo, without it.
3. Count echoes: `grep -o -i -f <(cut -d'|' -f1 terms.txt) plan_with.md | sort | uniq -c`.

Pass: ≥ 3 leading words echoed in (1) that are absent in (2), **and** the plan shape differs in the direction of at least one `Behaviour` line (e.g. slices instead of layers). Fail: the words appear but the plan is unchanged → the `Behaviour`/`Displaces` lines are too vague; rewrite them before shipping. Report the result in Step 10.

## Step 10 — Cleanup and report

Symlink for Claude Code (read-back verified), remove this run's `WORKDIR`, and report:

```
✅ Principles skill created: $SKILLS_HOME/<slug>/
📚 <Title> — <Author>   🗣 <N> leading words · <M> principles · <K> smells
Files: SKILL.md (~X) · leading-words.md (~X) · principles.md (~X) · smells.md (~X) · snippets/ (3) [· bindings.md]
Echo test: <passed / failed — reason / skipped>

Use it:
  paste snippets/claude-md.md into CLAUDE.md          → the words are always on
  "grill me with <slug>"                              → interview in the author's terms before a big change
  "review this with <slug>"                           → principles + smells over a diff
  "<slug> bind ./my-repo"                             → map the words to your project's own language
Discoverable by: <from CLAUDE_STATUS, never from the fact that ln ran>
```

Then, once, offer Step 8 (bind) if it did not run, and publishing (book-to-skill Step 11, same copyright gate: third-party books stay private).

---

## Quality Rules

1. **A word must change an action** — no `Displaces` line, no entry.
2. **Exact naming** — the phrase the author used, not a synonym; the prior is keyed on it.
3. **Small beats complete** — 25–40 words, a 300-token context block; attention is the budget, not disk.
4. **Reasons over rules** — every principle carries its *because*.
5. **Repo wins** — a bound project's existing term or convention overrides the book.
6. **Never copy raw text** — paraphrase; formulations ≤ 25 words.
7. **Front-load** — `SKILL.md` and `claude-md.md` put the top words first; compaction truncates from the end.
8. **Prove it** — run the echo test; a lexicon nobody echoes is a glossary.