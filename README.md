<h1 align="center">book-to-principles</h1>

<p align="center">
  <b>Don't hand your agent the book. Teach it the author's words.</b><br>
  Distill a book into the shared vocabulary you and your coding agent use to plan, build and review.
</p>

<p align="center">
  <a href="LICENSE.md"><img src="https://img.shields.io/badge/license-MIT-blue" alt="MIT"></a>
  <img src="https://img.shields.io/badge/status-pre--release-orange" alt="pre-release">
  <a href="https://github.com/virgiliojr94/book-to-skill"><img src="https://img.shields.io/badge/fork%20of-book--to--skill-lightgrey" alt="fork of book-to-skill"></a>
</p>

---

## Two words that change the plan

Ask a coding agent to "add CSV export to the reports page" and a typical plan comes back in layers:

```
1. Design the export schema and DB query
2. Build the export service
3. Add the API endpoint
4. Build the UI button and download flow
5. Integrate and test end-to-end
```

Now add one line to the prompt, *"Start with a tracer bullet"*, and the plan tends to come back in a different shape:

```
Tracer bullet: one hard-coded report → one CSV → one button, running end-to-end today.
Then widen it: real query, all columns, large-file streaming, errors.
```

You didn't explain anything. You didn't write a paragraph about incremental delivery, feedback loops or integration risk. You used **two words** that *The Pragmatic Programmer* made famous in 1999, and that the model has already read thousands of times. The words carried the whole method with them.

*(The plans above are illustrative. The [echo test](#prove-it-the-echo-test) is how you measure the effect on your own agent.)*

**book-to-principles makes those words deliberate.** Give it a book and it gives you back the author's working vocabulary. Each term comes with the behaviour it should trigger, the default it should override, and a line you can paste into `CLAUDE.md` so every session starts already speaking that language.

---

## Where the idea comes from

This project grew out of an interview with **Matt Pocock** on *The Pragmatic Engineer Podcast* (2026). Pocock is the author of the [`mattpocock/skills`](https://github.com/mattpocock/skills) collection (`grill-me`, `domain-modeling`, `improve-codebase-architecture`, …). Several of his ideas, taken together, suggested a tool:

1. **It's a communication problem, not an intelligence problem.** The agent is capable enough. It just can't read your values. Every session starts from zero (*Memento-driven development*), so whatever you care about has to be said again, explicitly, and as cheaply as possible.
2. **Leading words (*Leitwörter*).** Some terms (*tracer bullet*, *deep module*, *vertical slice*, *software entropy*, *ubiquitous language*) are already deep in the model's prior. Use them in a prompt or a skill and the agent starts echoing them in its reasoning. More importantly, **it changes what it does**: it stops building horizontal layers and cuts a thin path end-to-end.
3. **Domain language as the interface (DDD, Eric Evans).** Build a glossary while you design and put it in the code itself. Prompts get shorter, answers get less verbose, and the agent finds its way around the repo with a `grep` for the term.
4. **Speaking the same language.** The glossary isn't the valuable part. The valuable part is that **you, your repo and your agent use the same words for the same decisions**.
5. **Context is the budget.** The model reasons best inside a "smart zone" (roughly 150k tokens). Anything loaded into every session has to be tiny. That's why the `CLAUDE.md` block this tool generates is capped at **300 tokens**.

book-to-principles turns those five ideas into a pipeline.

---

## The bridge: from *reading* a book to *thinking* in it

There are two ways to get a book into an agent, and they solve different problems.

```
                        ┌──────────────────────────────┐
                        │   your book / docs / papers  │
                        └──────────────┬───────────────┘
                                       │  same deterministic extractor
                                       ▼
               ┌───────────────────────┴───────────────────────┐
               │                                               │
     book-to-skill                                    book-to-principles
     the agent CONSULTS the book                      the agent THINKS in the book
               │                                               │
   chapters · glossary · patterns                 leading words · principles · smells
   ~10–40K tokens, loaded on demand               ~5–6K total, 300 always-on
               │                                               │
   "What does Ousterhout say                      "Keep this a deep module."
    about module depth?"                           → the agent pushes complexity down,
   → it looks it up and answers                      rejects the pass-through wrapper,
                                                     and says so in its plan
```

| | [book-to-skill](https://github.com/virgiliojr94/book-to-skill) | **book-to-principles** |
|---|---|---|
| Question it answers | *"What does the book say about X?"* | *"How do I get my agent to act like the author, in as few words as possible?"* |
| The book becomes | a **library** the agent consults | a **language** the agent speaks |
| Output | per-chapter summaries + glossary + patterns + cheatsheet | 25–40 leading words + principles + smells + snippets |
| When it's in context | when you ask about it | always, in a tiny block, plus more on demand |
| Success looks like | tokens saved vs. pasting the PDF | the agent **repeats the words and changes the plan** |

They work well together. Use book-to-skill when you want to *study* a book. Use book-to-principles when you want your agent to *work* the way the author would. Both share the same extraction engine; the difference is in what gets distilled.

---

## What a leading word looks like

One term only earns a place in the lexicon if it changes what the agent does next. Each entry has fields that carry out that rule:

```markdown
## Tracer bullet  `phase: plan` · score 8/8

- **Canonical**: build one thin, real path through every layer first; aim by watching where it lands.
- **Trigger**: a feature touches more than one layer or service.
- **Behaviour**: ship the thinnest end-to-end path first, then widen it.
- **Displaces**: scaffold every layer in full, integrate at the end.
- **Say it as**: "Start with a tracer bullet: one path, end to end, running today."
- **Echo test**: the plan's first step names a single path across all layers.
- **Not to be confused with**: *prototype* (thrown away); a tracer bullet is kept and grown.
```

The **`Displaces`** line is the filter. If you can't name the default behaviour a word overrides, it's a glossary entry and not a leading word, so it gets dropped. Candidates are scored on four axes (named · dense · prior-likely · behaviour-changing, 0–2 each). Only terms scoring **6/8 or higher** make the cut, and the lexicon is capped at 40, because every extra word competes for the agent's attention.

---

## What you get

```
~/.agents/skills/<author>-principles/
├── SKILL.md            ≤1.5K tokens · top 8–12 words, when to load more, how to speak
├── leading-words.md    25–40 entries like the one above
├── principles.md       10–25 decision rules: when · do · rather than · because · Check
├── smells.md           anti-patterns as fast tells for review
├── snippets/
│   ├── claude-md.md    ≤300 tokens · paste into CLAUDE.md / AGENTS.md → the words are always on
│   ├── grill.md        8–15 decision questions in the author's language, before a big change
│   └── review.md       a review prompt: run every Check, report judgement calls, repo wins
└── bindings.md         (optional) the author's words ↔ your repo's words
```

Then:

```
paste snippets/claude-md.md into CLAUDE.md      → every session speaks the author's language
"grill me with ousterhout-principles"           → an interview in the author's terms before a big change
"review this with ousterhout-principles"        → principles + smells over a diff
"ousterhout-principles bind ./my-repo"          → map the words onto your codebase
```

---

## Bind it to your repo: the ubiquitous-language move

A book's vocabulary is generic. Your repo's isn't. **Bind** mode reads your `CLAUDE.md`, `AGENTS.md`, `CONTEXT.md` and ADRs (read-only), greps for where each concept already lives, and asks you **at most 10 questions, one at a time**. They are decisions, not a quiz:

> *"In this repo, does* deep module *mean the `pipelines/` packages, the `services/` layer, or something else?"*

The result is `bindings.md`: the author's term, your term, and where it lives in the code. When the two disagree, **your repo's term wins**; the book's term becomes an alias. Nothing is written to your `CLAUDE.md` unless you answer a literal `yes`.

At that point the book is no longer something you and your agent both read. It becomes words you both use.

---

## Prove it: the echo test

A lexicon nobody echoes is just a glossary. Before you trust one, run the check in a fresh session:

1. Ask for a plan for a medium-sized feature **with** `snippets/claude-md.md` in context.
2. Ask for the same plan, in the same repo, **without** it.
3. Count which leading words appear only in the first plan.

**Pass** = at least 3 words echoed **and** the plan's *shape* moved in the direction of a `Behaviour` line (slices instead of layers, one deep module instead of three shallow ones). If the words show up but the plan doesn't change, the `Behaviour`/`Displaces` lines are too vague: rewrite them before relying on the lexicon.

---

## Install

```bash
# As an agent skill (Claude Code, Copilot CLI, Amp, Codex, Hermes, OpenClaw)
npx skills add edumesones/book-to-principles

# Or manually: the folder MUST be named book-to-principles
git clone https://github.com/edumesones/book-to-principles.git ~/.claude/skills/book-to-principles
```

Check which extractors you have: `python scripts/extract.py --check`. Plain text, Markdown, HTML and most PDFs work with the standard library. Calibre is only needed for MOBI/AZW.

## Usage

```
/book-to-principles <path|folder|glob>... [slug] [--bind <project-root>]
```

| Mode | Trigger | Does |
|---|---|---|
| **Full distillation** | a path | extract → score → write the skill → scan → echo test |
| **Lexicon only** | "lexicon only" / "just the words" | show the scored candidate list; write nothing |
| **Bind** | `--bind <repo>` on an existing skill | interview + `bindings.md` + optional `CLAUDE.md` block |
| **Fold-in** | new sources + an existing slug | re-score the union, keep the 40-word cap, regenerate the snippets |

## Books that work best

Books whose vocabulary is already canonical, so the model's prior does half the work:

- Hunt & Thomas, *The Pragmatic Programmer*: tracer bullets, DRY, orthogonality, broken windows
- John Ousterhout, *A Philosophy of Software Design*: deep modules, information hiding, define errors out of existence
- Eric Evans, *Domain-Driven Design* (chapters 1–3): ubiquitous language, bounded context, model-driven design
- Martin Fowler, *Refactoring*: the code smells, named and catalogued

It also works for your own team's design docs. There the prior won't help, so the terms have to earn their place on density and behaviour alone.

---

## Copyright

This repository ships **no book content**. You point it at files you already own.

- **Processing is local.** Extraction runs on your machine. Text you send to a cloud model follows that provider's normal terms.
- **The output is your notes.** A vocabulary and paraphrased rules, with every formulation limited to 25 words or fewer. It never copies raw passages.
- **Keep third-party skills private.** When you publish, the generator creates private repos by default and only makes a repo public if you answer the one-word question with `public` and the source allows it.

## Status

Pre-release fork. The generator spec (`SKILL.md`) is complete. The security scanner and tests are being moved over to the new output format, and the first end-to-end run and echo test on a real book are next. Results will be published here once measured.

## Credits

- [**book-to-skill**](https://github.com/virgiliojr94/book-to-skill) by [@virgiliojr94](https://github.com/virgiliojr94) (MIT): the extraction engine, multi-host install logic and security scanner this fork is built on.
- **Matt Pocock**, interviewed on *The Pragmatic Engineer Podcast*: leading words, shared language and the smart-zone budget. See [`mattpocock/skills`](https://github.com/mattpocock/skills).
- Andrew Hunt & David Thomas, John Ousterhout, Eric Evans, Martin Fowler: the vocabulary this tool exists to pass on.

## License

MIT. It covers the code and the generator spec in this repository, **not** any book or document you process with it.
