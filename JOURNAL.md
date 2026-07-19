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
  - Chose this partly to try Apple's newer tooling, and partly because **Apple has a confirmed timeline for winding down Rosetta 2**: full support through macOS 27 (fall 2026), then largely discontinued starting macOS 28 (fall 2027). Apple began surfacing in-app deprecation warnings starting with macOS 26.4 — I've seen this warning myself on this machine's stable macOS 26.5.2 install (not a beta), so it's already live in production releases. That matters here specifically because SETUP.md's Apple Silicon instructions tell contributors to enable Docker Desktop's "Use Rosetta for x86_64/amd64 emulation" setting — the fast path Docker Desktop uses to run any `linux/amd64`-only image inside the ARM64 VM. Once that support window closes, that path degrades to slow QEMU emulation or stops working outright, which future cohorts on Apple Silicon will hit if any project image lacks a native arm64 build. That's roughly a year out, not an immediate problem, but worth a mention here since this course spans multiple cohorts over time. Apple's `container` tool sidesteps the issue entirely since it doesn't depend on Rosetta at all — it just needs images to publish an arm64 variant, which ours do.
    - References: [macOS 26.4 will notify users of Rosetta 2 discontinuation (9to5Mac)](https://9to5mac.com/2026/02/16/macos-26-4-will-notify-users-of-rosetta-2-discontinuation/), [Rosetta 2 End of Support: macOS 28 Will Break 18,000+ Intel Apps in 2027 (Tech Times)](https://www.techtimes.com/articles/317445/20260530/rosetta-2-end-support-macos-28-will-break-18000-intel-apps-2027.htm)
  - Rough edge encountered: first-time `container system start` needed an interactive kernel download confirmation (`kata-containers` kernel) — had to auto-confirm it non-interactively. Otherwise setup was a straightforward drop-in replacement for `docker compose up -d`.
  - Postgres also needed one fix beyond plain `docker-compose.yml`: `initdb` refused to start because the container runtime's volume mount left a `lost+found` directory at the mount root, which Postgres treats as "not empty." Fixed by setting `PGDATA` to a subdirectory of the mount (`/var/lib/postgresql/data/pgdata`) instead of the mount root itself — a one-line addition to the `db` service's environment in `docker-compose.yml`.
- Confirmed `make run` serves the frontend at `http://localhost:5173` (200, correct app shell/title) and the API at `http://localhost:8000` (Swagger docs reachable). The `/health` endpoint itself reports Postgres/Redis as unhealthy, but that's a separate known bug (issues #154, #155) — verified both services directly (`SELECT 1` over asyncpg, `redis.ping()`) and they're genuinely up.
