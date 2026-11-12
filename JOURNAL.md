# Journal

## Unit 7 — Issue selection

**Issue link:** https://github.com/codepath/pathreview-ai201-fa26-howard/issues/55


**Issue title:** Skill extractor fails to detect JavaScript and TypeScript

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The skill extractor in `ingestion/parsers/skill_extractor.py` misses JavaScript
and TypeScript unless the evidence comes in a narrow form. `_detect_languages`
checks for `.js`/`.ts` only in the `filename` argument; from the text itself it
only looks for an `import`/`require` pattern and `package.json`, and that
pattern needs whitespace after the keyword, so ordinary code like
`require('fs')` or a list of `.ts` files in a README gets no JavaScript or
TypeScript entry. `_detect_tools` and `_detect_databases` have the same narrow
matching: they only fire when a technology's own name appears, so a Compose file
with `services:` and `build:` isn't recognized as Docker because the word
"docker" never shows up. A successful fix detects these languages and
technologies from the content itself, so the five `xfail`-marked tests in
`tests/unit/test_skill_extractor.py` pass and their markers can be removed.

**Branch name:** fix/55-skill-extractor-js-ts-detection

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

### Selection notes — "Is this right for me?" checklist

**Part 1 — Understanding the issue**

- **Can I explain it in my own words?** Yes. The extractor can't detect
  JavaScript or TypeScript unless they're spelled out explicitly. Right now it
  only finds them from the filename, from `import`/`require` statements, or from
  a `package.json` reference in the text; anything else goes undetected.
- **Which part of the app is affected?** The `ingestion` area (issue labels:
  `bug`, `ingestion`, `tier-1`). The file is
  `ingestion/parsers/skill_extractor.py`, and I confirmed it exists.
- **What does "done" look like?** Five xfail tests in
  `tests/unit/test_skill_extractor.py` define it:

  | Input | Before | After |
  |---|---|---|
  | JS code using `require('fs')` | no JavaScript | JavaScript detected |
  | TS code with `export interface`, `Promise<User>` | no TypeScript | TypeScript detected |
  | Python code with `import psycopg2` | no database | PostgreSQL detected |
  | Dockerfile text (`FROM`, `RUN`, `EXPOSE`) | no Docker | Docker detected |
  | Compose YAML (`services:`, `build:`) | no Docker | Docker detected |

**Part 2 — Tier fit**

- This is my first open source contribution, so Tier 1 is the right fit.
- Scope note: the title says JavaScript/TypeScript, but two tests cover Docker
  and one covers PostgreSQL. The real task is inferring technologies from what
  the code looks like rather than from their names. It's still one file and
  three functions, so it stays within Tier 1.

**Part 3 — Codebase readiness**

- **Found the code:** I read `_detect_languages`, `_detect_tools` and
  `_detect_databases` in `ingestion/parsers/skill_extractor.py`.
- **Safe to change:** nothing outside the tests imports this class. The agent
  uses its own separate `SkillExtractor` in `agent/tools/skill_extractor.py`,
  so changes here won't affect it.
- **Things to watch in my fix:**
  - The `import`/`require` pattern needs whitespace after the keyword, so
    `require('fs')` doesn't match.
  - TypeScript is chosen only from the filename, so text-based TypeScript
    evidence needs its own check.
  - The TypeScript test text currently gets falsely tagged as Python, because
    the type-annotation pattern `:\s*(int|str|...)` matches `id: string`.
- **Test file:** I read `tests/unit/test_skill_extractor.py`. It uses a single
  `extractor` fixture and asserts on detected skill names. I'll add at least one
  new test.

**Part 4 — Scope and time**

- **Others on this issue:** the cohort ledger shows one claim (mine), and the
  only GitHub comment is my claim. Nobody is assigned.
- **Time estimate:** about 3–4 hours, which is within the typical Tier 1 range
  of 2–4 hours. I'm confident I can finish before the Unit 9 deadline.
- **Blockers:** none. The issue is open and doesn't reference any other issue.

## Unit 8 — Reproduction

**Reproduced:** yes, reliably (same result on every run, no setup beyond the
project virtualenv).

**Steps:**

1. `git checkout fix/55-skill-extractor-js-ts-detection`
2. `pytest tests/unit/test_skill_extractor.py -q -rx`

   Result: `13 passed, 5 xfailed`. The five tests marked for #55 are reported
   as XFAIL.
3. `pytest tests/unit/test_skill_extractor.py -q --runxfail`

   Result: all five fail with `AssertionError`, which confirms their
   assertions really fail rather than being skipped.

**What the extractor returns today** (from calling
`SkillExtractor().extract_skills(text)` on each test's input):

| Test | Expected | Actual | Where it goes wrong |
|---|---|---|---|
| `test_javascript_detection` | JavaScript | `[]` | `_detect_languages`: the pattern `\b(import\|require)\s+` needs whitespace after the keyword, so `require('fs')` doesn't match. Nothing looks for `const`, `console.log` or `=>`, and `JS_TS_KEYWORDS` is defined but never used. |
| `test_text_with_typescript_files` | TypeScript | `Python` | `_detect_languages`: TypeScript is chosen only from a `.ts` filename. The Python annotation pattern `:\s*(int\|str\|...)` matches `id: string`, so the code gets tagged as Python instead. |
| `test_database_technology_detection` | PostgreSQL/SQL | `Python`, `JavaScript` | `_detect_databases`: `DATABASES` only matches literal names such as `postgresql`, so the driver name `psycopg2` isn't recognized. `import psycopg2` also matches the JS import pattern, which makes JavaScript a false positive. |
| `test_devops_tool_detection` | Docker | `Python` | `_detect_tools`: `TOOLS` only matches the word `docker`. Dockerfile syntax (`FROM`, `RUN`, `EXPOSE`) isn't recognized. |
| `test_docker_compose_detection` | Docker | `[]` | `_detect_tools`: same cause. Compose syntax (`services:`, `build:`, `ports:`) isn't recognized. |

**Where it lives:** `ingestion/parsers/skill_extractor.py`, in
`_detect_languages`, `_detect_databases` and `_detect_tools`.

**Note for the fix:** the five tests use `xfail(strict=True)`, so once they
pass they'll show as XPASS, which strict mode counts as a failure. The markers
have to be removed in the same commit as the fix.
