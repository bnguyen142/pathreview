# Journal

## Week 7 — Issue selection

**Issue link:** [https://github.com/ascherj/pathreview/issues/147](https://github.com/ascherj/pathreview/issues/147)

**Issue title:** Resume section detection fails on text with leading whitespace

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
`ResumeParser._detect_sections()` (in `ingestion/parsers/resume_parser.py`) is supposed to scan a resume's extracted text and report which standard sections it finds — Education, Skills, Experience, and so on — so the rest of the ingestion pipeline knows what structure the document has. Its regex patterns are anchored to the very start of a line, with no tolerance for leading whitespace before a header. PDF-extracted text and indented markdown resumes commonly have that leading whitespace, so `detected_sections` comes back completely empty even when the sections are clearly present to a human reader. A correct fix makes header detection tolerant of leading whitespace without introducing false positives, so indented and unindented resumes are scored the same, and the existing test suite (plus the indented-text case from the issue) passes.

**Branch name:** `fix/147-resume-section-whitespace`

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

---

### Selection notes — scope reasoning

I'm new to contributing to a project this size and have never had a PR merged into an unfamiliar codebase before, so I deliberately picked Tier 1 rather than reaching for something more architecturally ambitious. My goal for this module isn't to prove I can handle the hardest issue available — it's to actually practice the full loop the module is teaching: read and understand a large, unfamiliar codebase, choose a well-scoped bug to work, investigate and explain the bug clearly, propose a fix, and back it with a new test that guards the specific edge case so a future change doesn't silently reintroduce the same bug. A Tier 1 issue lets me go deep on that full loop on a small surface area instead of spending most of my time just tracing how services connect.

Why this issue specifically fits my "is this right for me" check:

- **Contained blast radius.** It's scoped to one file, `ingestion/parsers/resume_parser.py`, with no changes needed in the API, agent, or frontend layers — small enough that I can hold the whole relevant context in my head while I learn how the ingestion pipeline is put together.
- **Objectively verifiable.** The bug has a precise, reproducible trigger (indented input text), and the issue body already includes a minimal repro script and names the three failing tests (`test_parse_single_column_resume_text`, `test_parse_resume_no_work_experience`, `test_detect_sections` in `tests/unit/test_resume_parser.py`), so I can check my understanding — and later my fix — against a concrete pass/fail signal instead of guessing at "done."
- **No hidden design decisions.** It's a regex/string-handling bug, not an architecture choice — there's a clear right answer, so I can focus my first contribution on process (root-causing the bug, writing a correct fix, adding a regression test for the leading-whitespace edge case) rather than also having to make judgment calls I'm not yet equipped for.

I confirmed this by actually reproducing the bug locally (not just reading the issue): `_detect_sections()` returns `[]` for indented text exactly as described, and all three named tests fail. I also found the same anchoring bug independently breaks `_strip_markdown()` (2 more failing tests not mentioned in the issue: `test_parse_markdown_resume`, `test_strip_markdown_syntax`) — worth accounting for when I write the fix and plan next week, since the full correct fix is slightly bigger than the issue body alone suggests.

### Setup notes

- Forked to `bnguyen142/pathreview`; `origin` is my fork, `upstream` is set to `ascherj/pathreview`. Note: this repo's own README uses an older clone URL, `jamjamgobambam/pathreview` — that resolves to the same GitHub account (confirmed via `gh repo view`: identical owner ID and issue list), it's just a redirected handle, not a separate fork.
- **Backing services (Postgres, Redis, ChromaDB) run via Apple's native `container` tool + the third-party `container-compose` bridge, not Docker Desktop.** Both installed via Homebrew (`brew install container container-compose`). `container-compose` reads the existing `docker-compose.yml` as-is — no changes needed to the compose file itself, since all three images (`postgres:16-alpine`, `redis:7-alpine`, `chromadb/chroma`) publish native arm64 builds.
  - Chose Apple's `container` tool partly to try newer tooling, and partly to avoid Docker Desktop's Rosetta-based amd64 emulation path, since all three service images publish native arm64 builds anyway.
  - Rough edge encountered: first-time `container system start` needed an interactive kernel download confirmation (`kata-containers` kernel) — had to auto-confirm it non-interactively. Otherwise setup was a straightforward drop-in replacement for `docker compose up -d`.
  - Postgres also needed one fix beyond plain `docker-compose.yml`: `initdb` refused to start because the container runtime's volume mount left a `lost+found` directory at the mount root, which Postgres treats as "not empty." Fixed by setting `PGDATA` to a subdirectory of the mount (`/var/lib/postgresql/data/pgdata`) instead of the mount root itself — a one-line addition to the `db` service's environment in `docker-compose.yml`.
- Confirmed `make run` serves the frontend at `http://localhost:5173` (200, correct app shell/title) and the API at `http://localhost:8000` (Swagger docs reachable). The `/health` endpoint itself reports Postgres/Redis as unhealthy, but that's a separate known bug (issues #154, #155) — verified both services directly (`SELECT 1` over asyncpg, `redis.ping()`) and they're genuinely up.

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** [1f793a1](https://github.com/bnguyen142/pathreview/commit/1f793a1)

**Reproduction summary:**
Ran `.venv/bin/pytest tests/unit/test_resume_parser.py -v` locally. Result: **6 failed, 5 passed**. Five of the six failures were pre-existing (predicted in the Week 7 investigation); the sixth (`test_parse_pdf_with_indented_sections`) is a new regression test I added this week specifically to cover the PDF ingestion path:

```text
FAILED tests/unit/test_resume_parser.py::TestResumeParser::test_parse_single_column_resume_text
FAILED tests/unit/test_resume_parser.py::TestResumeParser::test_parse_resume_no_work_experience
FAILED tests/unit/test_resume_parser.py::TestResumeParser::test_parse_pdf_with_indented_sections
FAILED tests/unit/test_resume_parser.py::TestResumeParser::test_parse_markdown_resume
FAILED tests/unit/test_resume_parser.py::TestResumeParser::test_detect_sections
FAILED tests/unit/test_resume_parser.py::TestResumeParser::test_strip_markdown_syntax
```

The first three (`test_parse_single_column_resume_text`, `test_parse_resume_no_work_experience`, `test_detect_sections`) are named directly in issue #147. `test_parse_markdown_resume` and `test_strip_markdown_syntax` fail for the same root cause (regex patterns in `_strip_markdown()` anchored to `^`/`\n` with no leading-whitespace tolerance) but aren't mentioned in the issue body. `test_parse_pdf_with_indented_sections` is new: none of the issue's named tests exercise `_parse_pdf()` at all, even though the issue specifically calls out PDF-extracted text as the real-world trigger — so I added a test mocking `PdfReader` (following the existing `test_parse_multipage_pdf` pattern) with indented `Experience:`/`Education:`/`Skills:` headers, confirming the exact same bug reproduces through the PDF code path, not just markdown/plain-text.

Full-suite baseline (`.venv/bin/pytest tests/unit -v -m unit`, run before making any production code change): **54 failed, 375 passed**. Only 6 of those 54 belong to `test_resume_parser.py` — the other 48 are pre-existing failures across ~15 unrelated modules, unrelated to this issue.

**Exact keyword misses observed:**

`_detect_sections()` checks each of the 13 known keywords in `SECTION_HEADERS` (resume_parser.py:9-23) against the whole document independently. The keywords actually exercised by these test fixtures are `experience`, `education`, and `skills`. For each, all 4 patterns (resume_parser.py:134-138) require the keyword immediately after `^` or `\n`, with zero whitespace tolerance before it:

| Raw header line (repr, whitespace visible) | Pattern that should match | Exact miss |
| --- | --- | --- |
| `'    Experience:'` (4-space indent, `sample_resume_text` fixture) | `` ^experience\s*[:\|-] `` | `^` requires `e` as the very next character; finds a space instead |
| `'        Education:'` (8-space indent, `test_parse_resume_no_work_experience`) | `` ^education\s*[:\|-] `` | same — 8 spaces sit between `^` and `e` |
| `'        Skills: Python, JavaScript'` (8-space indent, `test_detect_sections`) | `` ^skills\s*[:\|-] `` | same — regex has no allowance before `s` |

`_strip_markdown()`'s single header regex (resume_parser.py:102, `r"^#+\s+"`) has the identical miss on `'        # Header'` and `'        ## Contact'` / `'        ## Experience'` / `'        ## Skills'` (`test_strip_markdown_syntax`, `test_parse_markdown_resume`): `^#+` requires `#` immediately at line-start, finds a space instead, so the `#`/`##` is never stripped.

**Confirmed the bug also reproduces through the PDF path:** none of the 5 failing tests actually exercise `_parse_pdf()` — they're all markdown/plain-text input. Ran a one-off diagnostic (mocking `PdfReader` the same way `test_parse_multipage_pdf` does, with a page returning indented `Experience:`/`Education:`/`Skills:` headers) and confirmed `parser.parse(pdf_bytes)` also returns `detected_sections: []`. This matters because the issue names PDF-extracted text as the real-world trigger — this confirms the fix needs a dedicated PDF-path regression test (see `PLAN.md` Plan step 5), not just coverage through the markdown path.

**PLAN.md link:** [PLAN.md](https://github.com/bnguyen142/pathreview/blob/fix/147-resume-section-whitespace/PLAN.md)

**Walkthrough video (recommended):** [Week 8 walkthrough](https://youtu.be/I53GGVbN0T4)

**Blockers or open questions:**
Hit and resolved one blocker this week: committing the new `test_parse_pdf_with_indented_sections` test tripped the `mypy` pre-commit hook, which flagged 12 missing-type-annotation errors in `test_resume_parser.py` — 11 of them pre-existing, in tests I didn't write. Root cause: `make typecheck` (the Makefile target `CONTRIBUTING.md` points to) excludes `tests/` entirely, so this file's lack of type annotations had never been caught before, while the pre-commit hook has no such exclusion. Fixed by adding proper type annotations to all 12 functions, plus two `# type: ignore[arg-type]` comments on tests that intentionally pass invalid types to verify runtime validation. Verified all three hooks (ruff, black, mypy) now pass, and the actual test results are unchanged (6 failed / 5 passed). Documented as a general risk in `PLAN.md` for Week 9.

Remaining open question for the fix itself: how to make the regex leading-whitespace-tolerant without introducing false positives (e.g. a bullet point or code snippet that happens to start with a section-header word after indentation) — captured in `PLAN.md`'s Risks & Unknowns.
