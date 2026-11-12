## Solution plan

**Issue:** Skill extractor fails to detect JavaScript and TypeScript —
https://github.com/codepath/pathreview-ai201-fa26-howard/issues/55

### Understand

**Root cause:** every detector in `SkillExtractor` relies on narrow literal
matches instead of recognizing what the code looks like.

- **JavaScript:** the only text pattern is `\b(import|require)\s+`, which needs
  whitespace after the keyword. `require('fs')` has a parenthesis there, so it
  never matches. The same pattern does match Python's `import psycopg2`, which
  produces a false JavaScript detection.
- **TypeScript:** chosen only when the `filename` argument contains `.ts`.
  Nothing in the text (`interface`, `: string`, `Promise<User>`) is checked.
- **Python false positive:** the annotation pattern
  `:\s*(int|str|float|bool|list|dict)` has no word boundary, so `id: string`
  matches on `str` and TypeScript code is reported as Python.
- **Databases:** `DATABASES` only matches the database's own name
  (`postgresql`). A driver import like `psycopg2` isn't recognized.
- **Docker:** `TOOLS` only matches the word `docker`. Dockerfile instructions
  (`FROM`, `RUN`, `EXPOSE`) and Compose keys (`services:`, `build:`, `ports:`)
  aren't recognized.

**Expected vs. actual** (reproduced in JOURNAL.md, Unit 8):

| Input | Expected | Actual |
|---|---|---|
| `const fs = require('fs'); console.log(...)` | JavaScript | nothing |
| `export interface User { id: string }` … `Promise<User>` | TypeScript | Python |
| `import psycopg2; psycopg2.connect(...)` | PostgreSQL | Python, JavaScript |
| Dockerfile (`FROM`, `RUN`, `EXPOSE`) | Docker | Python |
| Compose YAML (`services:`, `build:`, `ports:`) | Docker | nothing |

### Map

**Files I expect to touch:**

- `ingestion/parsers/skill_extractor.py`
  - `_detect_languages`: JS/TS text patterns, the TS-vs-JS choice, and the
    Python annotation regex
  - `_detect_databases`: driver and package aliases
  - `_detect_tools`: Dockerfile and Compose patterns
  - new class constants for the patterns and aliases, next to `DATABASES` and
    `TOOLS`
- `tests/unit/test_skill_extractor.py`
  - remove the five `xfail(strict=True)` markers
  - add tests for the false positives (see Plan step 5)

**Checked, not touching:**

- `agent/tools/skill_extractor.py` is a separate `SkillExtractor` (a
  `BaseTool`) used by `agent/orchestrator.py`. It doesn't import the parser
  version, so this fix doesn't affect it.
- Nothing outside the tests imports `ingestion.parsers.skill_extractor`.

### Plan

1. **Fix JavaScript detection** in `_detect_languages`. Replace the loose
   `import|require` pattern with JS-specific ones: `require\(`,
   `import … from '…'`, `module.exports`, `export default`, `console.log`,
   `=>`, and `const`/`let` declarations. Python's `import x` and
   `from x import y` must stop counting as JavaScript.
2. **Add TypeScript detection from text.** Check for `interface X {`, type
   annotations such as `: string`, `: number` and `: boolean`, generics like
   `Promise<…>`, and `type X =`. Report **TypeScript** when TS-specific
   evidence exists (from text or a `.ts`/`.tsx` filename), otherwise
   **JavaScript**. Add `\b` to the Python annotation regex so `string` no
   longer matches `str`.
3. **Add database aliases** in `_detect_databases`: a map from driver or
   package names to the canonical key, e.g. `psycopg2`, `psycopg` and
   `asyncpg` → `postgresql`; `pymysql` → `mysql`; `pymongo` and `mongoose` →
   `mongodb`. Aliases resolve to the same display name as the canonical key,
   so a text that mentions both still produces one entry.
4. **Add Docker detection from syntax** in `_detect_tools`.
   - **Dockerfile:** a line starting with `FROM <image>`, plus at least one
     other line-start instruction (`RUN`, `COPY`, `CMD`, `EXPOSE`,
     `ENTRYPOINT`, `WORKDIR`).
   - **Compose:** a `services:` key on its own line, plus at least one of
     `build:`, `image:` or `ports:`. I can't require `services:` to be
     top-level, because the indented test input would fail.
   - Both produce the existing `Docker` entry, with evidence saying which
     pattern matched.
   - "Line start" has to allow leading whitespace (`^\s*FROM`, with
     `re.MULTILINE`), because the test inputs are indented triple-quoted
     strings.
5. **Update the tests.** Remove the five xfail markers. Add regression tests
   for the false positives: Python `import psycopg2` must not produce
   JavaScript, `id: string` must not produce Python, and SQL
   `SELECT … FROM users` must not produce Docker. Then run the full unit suite
   and the pre-commit hooks (ruff, black, mypy).

### Inputs & outputs

- **Input:** unchanged. `extract_skills(text: str, filename: str | None)`.
- **Output:** unchanged type. A `list[SkillDetection]` sorted by confidence
  (highest first). What changes is which entries appear:
  - JavaScript and TypeScript are detected from code alone.
  - PostgreSQL, MySQL and MongoDB are detected from driver imports.
  - Docker is detected from Dockerfile and Compose syntax.
  - The Python-on-TypeScript and JavaScript-on-Python false positives go away.
- Each new detection carries an `evidence` string naming the pattern that
  fired, and a confidence in the same 0.6–0.95 range the existing detectors
  use.

### Risks & unknowns

- **False positives from broader patterns.** This is the main risk. `FROM` is
  also SQL, `services:` also appears in GitHub Actions and Kubernetes YAML, and
  `=>` appears in PHP. Requiring two signals (line-start uppercase
  instructions, or `services:` plus a Compose-specific key) should keep this
  down. The negative tests in step 5 are there to catch it.
- **Should TypeScript code also report JavaScript?** I'm planning to report
  only TypeScript when TS evidence exists. A text containing both JS and TS
  code, like `test_mixed_language_text`, will report TypeScript. That test only
  asserts more than one skill, so it still passes. I'm unsure whether
  reviewers would expect both, and I'll mention it in the PR.
- **Display names.** Existing names come from `.title()`, so the database shows
  as `Postgresql` rather than `PostgreSQL`. I'll keep that format to stay in
  scope and avoid renaming entries other code might match on, but I'll mention
  it in the PR.
- **Python vs. TypeScript annotation overlap.** `x: int` is valid in both
  languages. After the `\b` fix, `: str` and `: int` still count as Python,
  and `: string`, `: number` and `: boolean` count as TypeScript. That's
  reasonable but not perfect.
- **Out of scope but noticed:** filename checks use substring matching, so
  `.cs` matches `style.css`, `.js` matches `data.json`, and `.ts` matches
  `results.tsv`. I'll note this for a separate issue rather than fix it here,
  unless switching to `endswith` turns out to be needed for the JS/TS work.

### Edge cases

- **Empty text or whitespace only:** returns `[]` without errors (an existing
  test covers this).
- **`filename=None`:** no crash, and the text patterns still run.
- **Mixed-language text:** each language is detected independently, with no
  duplicate entries.
- **Alias and canonical name both present** (`psycopg2` and `postgresql`):
  one PostgreSQL entry, not two.
- **The literal word `docker` and Dockerfile syntax both present:** one Docker
  entry.
- **Prose that mentions languages** (for example a README saying "written in
  JavaScript"): the existing name-based framework matching still applies, and
  the new syntax patterns shouldn't fire on plain sentences.
- **Lowercase `from x import y` (Python) and SQL `SELECT … FROM t`:** neither
  should trigger the Dockerfile pattern, because it requires uppercase
  instructions at the start of a line plus a second instruction.
