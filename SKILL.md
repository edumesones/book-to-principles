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

<!-- Inlined from book-to-skill (commit 80ae087): this repo no longer ships that spec,
     so "same as book-to-skill" pointed at nothing. -->
This generator can run from multiple skill systems. When looking for its helper script or writing the generated skill, prefer these locations in order:

1. GitHub Copilot CLI personal skills: `~/.copilot/skills/`
2. Cross-agent personal skills (Copilot, Amp, Codex; OpenClaw with its default state): `~/.agents/skills/`
3. Claude Code personal skills: `~/.claude/skills/`
4. Project-local Copilot skills: `.github/skills/`
5. Project-local Claude skills: `.claude/skills/`
6. Project-local Amp / Copilot / OpenClaw skills: `.agents/skills/`
7. Amp global skills: `~/.config/agents/skills/`
8. Amp legacy global skills: `~/.config/amp/skills/`
9. Hermes Agent personal skills: `$HERMES_HOME/skills/` (defaults to `~/.hermes/skills/`)
10. Hermes Agent project skills: `.hermes/skills/` or `.agents/skills/`
11. OpenClaw personal skills: `${OPENCLAW_STATE_DIR:-~/.openclaw}/skills/` (active state; `~/.agents/skills/` is shared only with the default state)
12. OpenClaw project skills: `.agents/skills/` or `skills/`

For **generated** principles skills, prefer the user-level cross-agent root `~/.agents/skills/`. Copilot CLI and Amp discover it natively; Claude Code needs a symlink from `~/.claude/skills/<slug>` (Step 10). Pick a host-private or project-local root only when the user explicitly asks for one. `BOOK_TO_SKILL_SCOPE=project` or `personal` can make that choice explicit for automation; do not ask a mandatory scope question merely because both scopes are available.

---

## Step 0 — Out-of-scope check

No arguments → stop:
> "book-to-principles needs a document path, folder or glob. Usage: `book-to-principles <paths>... [slug] [--bind <project-root>]`"

Parse: `INPUT_PATHS`, optional `SKILL_NAME` (last arg that is not a path and looks like a slug), optional `--bind PROJECT_ROOT`. If `SKILL_NAME` already exists under `SKILLS_HOME` and there are new sources → Mode 4; if `--bind` and no sources → Mode 3.

## Step 1 — Validate input
Expand directories and globs to supported files: `.pdf`, `.epub`, `.docx`, `.txt`, `.md`, `.markdown`, `.rst`, `.adoc`, `.html`, `.htm`, `.rtf`, `.mobi`, `.azw`, `.azw3`. If none are found, stop with a clear error naming the paths you checked.

## Step 1.5 — Content type
Ask once: "Technical (code, tables, formulas) / text-heavy (mostly prose) / not sure?" → `BOOK_TYPE=technical` for the first, `BOOK_TYPE=text` otherwise. Technical uses Docling (~1.5 s/page — warn the user); text uses the fastest extractor per format. Most books worth distilling here are *text-heavy* (Pragmatic Programmer, Ousterhout, Evans, Fowler): say so and default to text unless the user objects.

## Step 2 — Extract
Locate this generator's `scripts/extract.py` and run it:

```bash
SCRIPT_PATH=""
HERMES_HOME_RESOLVED="${HERMES_HOME:-$HOME/.hermes}"
OPENCLAW_STATE_DIR_RESOLVED="${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"
PROJECT_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || true)"
HERMES_PROJECT_TRUSTED=false
if [ -n "$PROJECT_ROOT" ] && [ "${HERMES_AGENT:-}" = true ] && \
  command -v hermes >/dev/null 2>&1 && \
  command -v python3 >/dev/null 2>&1 && \
  hermes config get skills.trusted_project_dirs --json 2>/dev/null | PROJECT_ROOT="$PROJECT_ROOT" python3 -c 'import json, os, pathlib, sys; root=pathlib.Path(os.environ["PROJECT_ROOT"]).resolve(); sys.exit(not any(pathlib.Path(p).expanduser().resolve() == root for p in json.load(sys.stdin)))' 2>/dev/null
then
  HERMES_PROJECT_TRUSTED=true
fi

CANDIDATES=(
  "$HOME/.copilot/skills/book-to-principles/scripts/extract.py"
  "$HOME/.agents/skills/book-to-principles/scripts/extract.py"
  "$HOME/.claude/skills/book-to-principles/scripts/extract.py"
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills/book-to-principles/scripts/extract.py"
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/book-to-principles/scripts/extract.py
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/*/book-to-principles/scripts/extract.py
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/*/*/book-to-principles/scripts/extract.py
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/*/*/*/book-to-principles/scripts/extract.py
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/*/*/*/*/book-to-principles/scripts/extract.py
  "${OPENCLAW_STATE_DIR_RESOLVED}/skills"/*/*/*/*/*/*/book-to-principles/scripts/extract.py
  "$HERMES_HOME_RESOLVED/skills/book-to-principles/scripts/extract.py"
  "$HERMES_HOME_RESOLVED"/skills/*/book-to-principles/scripts/extract.py
)
if [ "${HERMES_AGENT:-}" != true ]; then
  CANDIDATES+=(
    ".github/skills/book-to-principles/scripts/extract.py"
    ".claude/skills/book-to-principles/scripts/extract.py"
    ".agents/skills/book-to-principles/scripts/extract.py"
    "skills/book-to-principles/scripts/extract.py"
    "skills"/*/book-to-principles/scripts/extract.py
    "skills"/*/*/book-to-principles/scripts/extract.py
    "skills"/*/*/*/book-to-principles/scripts/extract.py
    "skills"/*/*/*/*/book-to-principles/scripts/extract.py
    "skills"/*/*/*/*/*/book-to-principles/scripts/extract.py
    "skills"/*/*/*/*/*/*/book-to-principles/scripts/extract.py
  )
  if [ -n "$PROJECT_ROOT" ]; then
    CANDIDATES+=(
      "$PROJECT_ROOT/skills/book-to-principles/scripts/extract.py"
      "$PROJECT_ROOT/skills"/*/book-to-principles/scripts/extract.py
      "$PROJECT_ROOT/skills"/*/*/book-to-principles/scripts/extract.py
      "$PROJECT_ROOT/skills"/*/*/*/book-to-principles/scripts/extract.py
      "$PROJECT_ROOT/skills"/*/*/*/*/book-to-principles/scripts/extract.py
      "$PROJECT_ROOT/skills"/*/*/*/*/*/book-to-principles/scripts/extract.py
      "$PROJECT_ROOT/skills"/*/*/*/*/*/*/book-to-principles/scripts/extract.py
    )
  fi
fi
CANDIDATES+=(
  "$HOME/.config/agents/skills/book-to-principles/scripts/extract.py"
  "$HOME/.config/amp/skills/book-to-principles/scripts/extract.py"
)
if [ "$HERMES_PROJECT_TRUSTED" = true ]; then
  CANDIDATES=(
    "$PROJECT_ROOT/.hermes/skills/book-to-principles/scripts/extract.py"
    "$PROJECT_ROOT/.hermes/skills"/*/book-to-principles/scripts/extract.py
    "$PROJECT_ROOT/.agents/skills/book-to-principles/scripts/extract.py"
    "$PROJECT_ROOT/.agents/skills"/*/book-to-principles/scripts/extract.py
    "${CANDIDATES[@]}"
  )
fi
for candidate in "${CANDIDATES[@]}"
do
  if [ -f "$candidate" ]; then
    SCRIPT_PATH="$candidate"
    break
  fi
done

if [ -z "$SCRIPT_PATH" ]; then
  # Diagnostic: the install folder must be named book-to-principles (not book-to-skill).
  echo "Could not find scripts/extract.py for book-to-principles — is the skill installed under a folder named 'book-to-principles'?" >&2
  exit 1
fi

PYTHON_BIN="${PYTHON_BIN:-python3}"
if ! command -v "$PYTHON_BIN" >/dev/null 2>&1; then
  PYTHON_BIN="python"
fi

"$PYTHON_BIN" "$SCRIPT_PATH" $INPUT_PATHS --mode <BOOK_TYPE> --install-missing ask
```

Setup or quality problem? `"$PYTHON_BIN" "$SCRIPT_PATH" --check` prints which extractors are installed and how to install the rest.

Each run gets its own workdir (`<tempdir>/book_skill_work-<pid>/`, or `BOOK_SKILL_WORKDIR`). **Take `Workdir ->`, `Text ->`, `Meta ->` from the run output**, never a fixed path — concurrent runs must not read each other's output. Confirm the `SOURCE:` header of `full_text.txt` matches what the user asked for.

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

Slug: `{author-lastname}-principles` by default (`hunt-thomas-principles`, `ousterhout-principles`, `evans-ddd-principles`); a user-supplied `SKILL_NAME` wins.

Choose `SKILLS_HOME`. First resolve **scope** from an explicit user request or `BOOK_TO_SKILL_SCOPE`, then probe **host**. A request for project-local/project output selects the project-local row; a request for personal/global output selects the personal row. If neither is requested, preserve the established personal default (`~/.agents/skills` for non-Hermes hosts). Do not ask a mandatory scope question solely because project-local roots exist. The selected root may still require host approval before writing.

| Host agent | Personal skill root | Project-local root |
|---|---|---|
| **GitHub Copilot CLI** | `~/.agents/skills` (native) | `.github/skills` → `.claude/skills` → `.agents/skills` |
| **Amp** | `~/.agents/skills` (native) | `.agents/skills` |
| **OpenAI Codex** | `~/.agents/skills` (native; follows symlinks) | `.agents/skills` |
| **Hermes Agent** | `$HERMES_HOME/skills/<category>` (defaults to `~/.hermes/skills/<category>`) | `.hermes/skills/<category>` → `.agents/skills` |
| **Claude Code** | `~/.agents/skills` + symlink from `~/.claude/skills/<slug>` | `.claude/skills` |
| **OpenClaw** | `${OPENCLAW_STATE_DIR:-~/.openclaw}/skills` (active state; `~/.agents/skills` only with default state) | `.agents/skills` → `skills/` |

Rules:
1. Personal install → `~/.agents/skills` (create if missing), unless it does not exist **and** the host's private root already holds skills; then use the private root and say why.
2. Claude Code does not scan `~/.agents/skills` → Step 10 symlinks it.
3. Hermes Agent keeps its own personal root (partitioned by category, no symlink). For project-local Hermes output, run `hermes skills trust <project-root>` and verify with `hermes skills list`.
4. OpenClaw: `~/.agents/skills` is valid only when `OPENCLAW_STATE_DIR` is unset or the default; otherwise use the active state root. Verify with `openclaw skills list`.
5. An explicitly requested host-private or project-local root is honoured; no symlink.
6. If the choice depends on the host and you cannot identify it, ask: "Which agent are you running in — OpenClaw, Hermes Agent, GitHub Copilot CLI, Amp, Codex, or Claude Code?"

On Claude Code, if `~/.claude/skills/<slug>` is a **real directory** (not a symlink), offer to migrate it into `~/.agents/skills/` before continuing. If `$SKILLS_HOME/<slug>/` exists: Fold-in (Mode 4) / Overwrite / Rename (`-2` or a new slug).

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
```bash
SKILL_CONVERTER_ROOT="$(cd "$(dirname "$SCRIPT_PATH")/.." && pwd)"
"$PYTHON_BIN" "$SKILL_CONVERTER_ROOT/tools/scan_generated_skill.py" "$SKILLS_HOME/<slug>"
```
Non-zero → stop and ask a human to review the file/line findings. Do not silently rewrite, load or publish the skill until they are resolved or explicitly accepted.

## Step 9.7 — Echo test (recommended, 5 minutes)

Before reporting success, verify the lexicon actually steers. In a **fresh** session on any repo:
1. Ask for a plan for a medium feature with `snippets/claude-md.md` in context; save the response.
2. Ask the same, same repo, without it.
3. Count echoes: `grep -o -i -f <(cut -d'|' -f1 terms.txt) plan_with.md | sort | uniq -c`.

Pass: ≥ 3 leading words echoed in (1) that are absent in (2), **and** the plan shape differs in the direction of at least one `Behaviour` line (e.g. slices instead of layers). Fail: the words appear but the plan is unchanged → the `Behaviour`/`Displaces` lines are too vague; rewrite them before shipping. Report the result in Step 10.

## Step 10 — Cleanup and report

If the host is Claude Code and `SKILLS_HOME` is `~/.agents/skills`, link the skill in and **read the link back** — on Windows/MSYS `ln -s` may copy instead of link, or fail silently:

```bash
mkdir -p "$HOME/.claude/skills"
LINK="$HOME/.claude/skills/<slug>"
TARGET="$HOME/.agents/skills/<slug>"
if [ -d "$LINK" ] && [ ! -L "$LINK" ]; then
  CLAUDE_STATUS="skipped-realdir"   # migration declined; ln -sfn would nest the link inside it
else
  ln -sfn "$TARGET" "$LINK" 2>/dev/null || true
  if [ -L "$LINK" ] && [ "$(readlink "$LINK")" = "$TARGET" ]; then
    CLAUDE_STATUS="linked"
  elif [ -e "$LINK" ]; then
    CLAUDE_STATUS="copy"
  else
    CLAUDE_STATUS="absent"
  fi
fi
```
Do not hard-fail on `copy`/`absent`: the skill exists at the hub; report that Claude Code will not see it until the link exists. Skip this for host-private or project-local roots.

Then remove **only this run's** workdir (the `Workdir ->` path; never a directory you did not create): `rm -rf "$WORKDIR"`. Report:

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

Reload (if your agent doesn't auto-detect new skills):
  GitHub Copilot CLI:  /skills reload
  Claude Code:         restart the session
  Amp:                 restart the session
  Hermes Agent:         start a new session
  OpenClaw:             openclaw skills list (new session if watcher disabled)
```

Fill "Discoverable by" from `CLAUDE_STATUS`: `linked` → "Copilot CLI, Amp, Codex (natively); Claude Code via symlink"; `skipped-realdir` / `copy` / `absent` → "Copilot CLI, Amp, Codex; **NOT** Claude Code — <reason and fix>"; Hermes/OpenClaw/private/project roots → only the host(s) that scan that root.

Then, once, offer Step 8 (bind) if it did not run, and Step 11 (publish).

## Step 11 — Publish to GitHub (optional)

Offer once, only if the Step 9.5 scan passed: "Publish this skill to GitHub so it installs with `npx skills add`? (yes / skip)". Needs `gh` authenticated (`gh auth status`); without it, the user creates an empty repo in the web UI and you push to it — the visibility rule is the same.

**Visibility is a separate closed question — never inferred from the publish offer or an earlier answer.** Ask on its own: "Private or public repository? Reply with one word: `private` or `public`." Use `--public` only when the reply *is* the bare word `public`. Substring matching is forbidden: a sentence about the source's licence ("it's public domain", "the book is publicly available") is not a visibility answer and resolves to private. A paraphrase, ambiguity or silence → re-ask once, then private, and say so.

**Copyright gate:** the lexicon is paraphrased, but it still derives from the book. Skills from **third-party copyrighted books stay private**; public only for the user's own writing, openly licensed material, or content they confirm they may redistribute — state which case applies. Internal company docs stay private unless the user states they hold publication rights.

If accepted:
1. Add a `README.md` in the skill folder (never overwrite): title, "Generated from *<Title>* by <Author> with [book-to-principles](https://github.com/edumesones/book-to-principles)", the install command, file inventory, and a note that the content is a synthesized vocabulary, not the book text.
2. **Nested-repo guard:** if `git -C "$SKILLS_HOME/<slug>" rev-parse --show-toplevel` succeeds, the folder is inside another repo — copy it to a scratch directory and publish from the copy (a `git init` in place would leave an embedded gitlink that fresh clones silently omit).

```bash
cd "$SKILLS_HOME/<slug>"
git init -b main
git add -A
git commit -m "Add <slug> skill"
gh repo create <repo_name> --private --source . --push
# --private is the default; --public ONLY if the visibility answer was the bare word "public" AND the copyright gate allows it
```
3. Report the URL, visibility, and `npx skills add https://github.com/<owner>/<repo_name> --skill <slug>`.

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