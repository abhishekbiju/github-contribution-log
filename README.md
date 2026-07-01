# Contribution #1: Examine Load Test Failure Modes

**Contribution Number:** 1  
**Student:** Abhishek Biju Das  
**Issue:** https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine/issues/208  
**Status:** Phase IV Complete — PR submitted, awaiting review

---

## Why I Chose This Issue

This issue stood out to me because it sits at the intersection of systems reliability and AI — examining how a simulation engine behaves under concurrent load when calling external LLM providers is a genuinely interesting problem. Rate limits, stalled requests, and malformed model responses are failure modes I've encountered in AI-integrated systems before, and I want to develop a more rigorous methodology for diagnosing them.

The task is also well-scoped for a first contribution: the load test infrastructure already exists in `scripts/`, so I can focus on analysis and documentation rather than building from scratch. The "help wanted" + "documentation" labels signal that the maintainers want a thorough written examination, which plays to my strengths in breaking down system behavior and communicating findings clearly.

---

## Understanding the Issue

### Problem Description

The project's load tests (located in `scripts/`) simulate concurrent gameplay sessions against the DCS simulation engine, running 10 clients × 10 games/client = 100 concurrent sessions by default. Each session covers an opening scene, 3 player turns, and a close. While the tests run and output HTML results to `docs/`, no one has yet systematically catalogued what failure modes actually occur — things like providers returning bad/empty responses, hitting rate limits, or requests stalling due to too-high concurrency.

### Expected Behavior

The load tests should complete all 100 concurrent sessions successfully, with the engine returning valid, non-empty responses from the AI provider for every turn. Results should be reproducible and documented so AI practitioners can understand the engine's reliability profile.

### Current Behavior

Failure modes are unexamined and unreported. When the engine or underlying models are under load, issues such as empty provider responses, rate-limit errors, or request stalls may occur — but there is no documented analysis of when, why, or how often these happen.

### Affected Components

- `scripts/` — load test scripts
- `docs/` — HTML output from load test runs
- Engine core and AI provider integration layer (wherever LLM calls are made during gameplay sessions)

---

## Reproduction Process

### Environment Setup

The project ships a **VS Code Dev Container** (`.devcontainer/`) plus a `CONTRIBUTING.md`, so setup was on the "best case" end of the spectrum. The container comes with Python 3.13, `uv`, the `dcs` CLI (installed editable), and Docker-outside-Docker already wired up.

Challenges I hit and how I resolved them:

1. **`OPENROUTER_API_KEY` is mandatory — and I don't have one.**
   The engine calls LLM providers through OpenRouter. `compose.yml` even fails at the *variable-interpolation* stage (`OPENROUTER_API_KEY must be set before running docker compose`) before any container starts, and `dcs server` refuses to boot without it (`ai_client.validate_openrouter_configuration()`).
   - *Fix:* The server exposes a `--fake-ai-response` flag (`dcs_simulation_engine/games/ai_client.py`) that returns a fixed string for every AI call instead of hitting OpenRouter. This let me run the entire stack offline with **no API key and zero credits spent**, which is exactly the right tool for exercising the *load-test harness* (the issue is about concurrency/failure-handling, not model quality).

2. **`docker compose up mongo` aborted on the missing key.**
   Compose interpolates the *whole* file, including the `api` service's required `OPENROUTER_API_KEY`, even when you only ask for `mongo`.
   - *Fix:* The repo provides a dedicated `.devcontainer/dev.compose.yml` that provisions **only** MongoDB (this is what the VS Code `start-mongo` task uses). Running `docker compose -f .devcontainer/dev.compose.yml up --detach` started Mongo cleanly.

3. **`pkill`/`ps` aren't in the container image.**
   Minor friction when restarting the background server — I killed processes by PID and confirmed the port was free with `ss -ltn | grep :8000`.

4. **Demo config rejects the load test with `409 Conflict`.**
   My first load-test run died immediately: `409 Conflict` on `POST /api/player/registration`, detail *"Player registration is disabled for this run."* The default `examples/run_configs/demo.yml` sets `ui.registration_required: false`, but the load-test client always calls `register_player`.
   - *Fix:* The repo includes a purpose-built `examples/run_configs/load-test.yml` (`registration_required: true`, `forms: null`). Switching the server to that config fixed it. **Lesson: the load test is coupled to the run config — it must be run against `load-test.yml` (or another `registration_required: true`, form-free config).**

**Working server command (offline mock mode):**
```sh
docker compose -f .devcontainer/dev.compose.yml up --detach   # MongoDB only
dcs server \
  --mongo-seed-dir database_seeds/dev \
  --config examples/run_configs/load-test.yml \
  --dump ./runs \
  --fake-ai-response '{"type":"ai","content":"A calm beat passes as the scene settles."}'
```
The fake response is valid JSON matching the engine's output contract (`{"type": "ai"|"info"|"warning", "content": <non-empty str>}`), so games complete normally.

### Steps to Reproduce

1. Start MongoDB: `docker compose -f .devcontainer/dev.compose.yml up --detach` (wait until `dcs-mongo` is healthy).
2. Start the engine in offline mock mode using the working server command above (leave it running in its own terminal).
3. Confirm readiness: `curl -s http://127.0.0.1:8000/healthz` → `{"status":"ok"}`.
4. Run the standard load profile (10 clients × 10 games × 3 turns = **100 concurrent sessions**):
   ```sh
   uv run python scripts/load_test_and_report.py --clients 10 --games 10 --turns 3
   ```
5. Observe the printed **Load Test Summary** (completions, failures, and per-phase wait-for-response latency).
6. To generate the HTML report: stop the engine (Ctrl-C / `kill`) so it dumps to `runs/<timestamp>/`; the script then writes `docs/reports/load_test_results_report.html`. (I generated it manually with `dcs report results runs/<timestamp> --only system-performance` since I run the server detached.)

**To reproduce a real failure mode** (issue calls out "provider returning defective or empty responses"), restart the server with a contract-violating fake response and rerun a small load test:
```sh
dcs server ... --fake-ai-response 'I am not valid JSON at all.'
uv run python scripts/load_test_and_report.py --clients 3 --games 2 --turns 1
```

### Reproduction Evidence

- **Branch:** https://github.com/abhishekbiju/dcs-simulation-engine/tree/fix-issue-208
- **Logs / observed output:**

  *Healthy run — 100 concurrent sessions, mock provider (reproduced twice, consistent):*
  ```
  Total games attempted: 100
  Total games completed: 100
  Total game failures:   0
  Wall time: ~7.2–7.6s
  Per-game duration (ms): min=29  mean=333  max=1083
  Wait-for-response (turn phase, ms): min=8  mean=164  max=922
  ```
  Even with an *instant* mock model, the per-turn "wait for response" latency fans out from ~8 ms to ~922 ms under 100 concurrent sessions — a clear **engine-side contention / tail-latency** signature that is independent of the LLM provider.

  *Defective-provider run — non-JSON response, 3×2×1 = 6 sessions:*
  ```
  Total games completed: 0
  Total game failures:   6
  Errors: client N game M: Server error: Session is closed   (×6)
  ```
  Server-side log shows the exact root-cause chain:
  ```
  ai_client.py | Model output contract failed: component=opener ... attempt=1 will_retry=True  detail=response was not valid JSON
  ai_client.py | Model output contract failed: component=opener ... attempt=2 will_retry=False detail=response was not valid JSON
  game.py:531  | Opening scene failed due to model provider error: ... OpenRouter returned unusable output ...
  ```

- **My findings (catalogued failure modes — the deliverable the issue asks for):**
  1. **Defective / empty provider response** → `ModelOutputContractError`. The engine retries **once** (2 attempts total), then fails the *opening scene*, which **closes the whole session** — the client only sees the generic `Server error: Session is closed`. Confirmed reproducible.
  2. **Config coupling** → running the load test against a `registration_required: false` config produces a blanket `409 Conflict` and 0 completions. Easy to mistake for an engine bug; it's a config mismatch.
  3. **Concurrency tail latency** → response wait time grows ~100× (8 ms → 922 ms) from min to max under 100 sessions even with a zero-latency model, pointing at engine-side serialization/contention rather than the provider. This is the "request stalling from excessive concurrency" failure mode, observable without real keys.
  4. **Rate limiting** → cannot be reproduced in mock mode (no real provider calls). Documented as requiring a live `OPENROUTER_API_KEY` run; see plan below.

---

## Solution Approach

### Analysis

This is a **documentation/analysis** issue (labels: `documentation`, `codebase`, `help wanted`), not a code bug. The deliverable is a systematic, reproducible **catalogue of load-test failure modes** so the team can report on engine reliability "once the engine and associated models reach stability."

Root-cause observations from reproduction:
- The load harness (`scripts/load_test_and_report.py`) already records per-game success/failure and per-phase latency, and already routes per-game exceptions into `ClientResult.errors` — so the *data* is mostly there; what's missing is a **written taxonomy** of what those errors mean and when they occur.
- Failure surfaces are concentrated in `dcs_simulation_engine/games/ai_client.py` (provider call + contract validation + single retry) and `game.py` (a failed opening scene tears down the whole session). The client only ever sees a generic `Session is closed`, which **hides the underlying cause** — a documentation gap worth flagging.

### Proposed Solution

Author a **"Load Test Failure Modes" reference** (a new docs page under `docs/`, wired into `mkdocs.yml`) that, for each failure mode, documents: trigger, how to reproduce it locally, the engine code path responsible, observable client-side symptom, current mitigation (e.g. the single contract retry), and recommended next step. Cover the three modes named in the issue — defective/empty responses, rate limiting, request stalling under concurrency — plus the config-coupling gotcha I hit. Optionally, lightly improve the harness so failures are easier to categorize (group/count errors by class instead of dumping raw strings), if maintainers want code changes.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The engine has no documented analysis of how it behaves under the standard 100-session load profile or what its failure modes are. I need to run the load test, observe and classify the failures, and write them up so future contributors and maintainers have a reliability baseline.

**Match:** Existing patterns I can mirror —
- `scripts/load_test_and_report.py` already aggregates results and emits a `system-performance` HTML report via `dcs report results ... --only system-performance` → my doc should link to / embed that report.
- Existing docs pages (`docs/software_architecture.md`, `docs/faq.md`) show the house style and how pages are registered in `mkdocs.yml`.
- `ai_client.py`'s `ModelOutputContractError` + retry loop and the `errors.py` provider-error taxonomy are the canonical list of provider failures to document.

**Plan:**
1. Run the standard 100-session profile and capture the healthy baseline + generated HTML report. *(done — see Evidence)*
2. Reproduce each named failure mode deterministically using `--fake-ai-response` (defective/empty output) and document the config-coupling 409. *(done for modes 1–3 + config; rate-limit needs a live key)*
3. Write `docs/load_test_failure_modes.md`: a table of {failure mode → trigger → repro command → code path → client symptom → mitigation/recommendation} and register it in `mkdocs.yml`.
4. *(Optional, pending maintainer interest)* Improve `_print_summary`/`ClientResult` to bucket errors by class so the report quantifies each failure mode automatically.
5. Run `make docs` (`mkdocs build --strict`) to confirm the new page builds with no warnings.

**Implement:** _(Phase III)_ Branch: https://github.com/abhishekbiju/dcs-simulation-engine/tree/fix-issue-208

**Review:** Self-review against `CONTRIBUTING.md`: keep it a docs-focused PR; follow the provided PR template (Overview / Changes / Test / Notes), reference `Addresses issue #208`; run `make docs` and `make lint` before pushing; use `dev` as the access key if any game type prompts for one.

**Evaluate:**
- `make docs` builds strict-clean with the new page in the nav.
- Every repro command in the doc is copy-pasteable and produces the documented output (re-verified on a clean checkout).
- A reviewer who has never run the load test can follow the doc to reproduce each failure mode.
- If harness code changes land, `make test` stays green.

---

## Testing Strategy

Because this is a docs-first contribution, "testing" means **verifying every documented repro is reproducible** and that the docs build cleanly.

### Doc/Build Checks

- [x] `make docs` (`mkdocs build --strict`) builds clean — *to run once the new page is added in Phase III*
- [ ] New `docs/load_test_failure_modes.md` appears in the mkdocs nav
- [ ] All repro commands in the doc are copy-pasteable and produce the documented output

### Reproduction Verification (done in Phase II)

- [x] Healthy 100-session run: 100/100 completed, 0 failures (reproduced twice, consistent)
- [x] Defective-response failure mode: 0/6 completed, contract error + single retry confirmed in server logs
- [x] Config-coupling 409: confirmed against `demo.yml`, resolved by `load-test.yml`
- [ ] Rate-limit failure mode: requires a live `OPENROUTER_API_KEY` run (not reproducible in mock mode)

### Manual Testing

- Started Mongo via `dev.compose.yml`, ran the engine in `--fake-ai-response` mock mode, and exercised the full harness (register → auth → list games → start → opening + 3 turns → close) across 100 concurrent sessions.
- Generated the `system-performance` HTML report (`docs/reports/load_test_results_report.html`, ~444 KB) from the engine's shutdown dump to confirm the end-to-end reporting flow works.

---

## Implementation Notes

### Phase III — Reviewing conventions

Per `CONTRIBUTING.md` this is a docs-focused contribution, so the relevant gates are
the doc build (`make docs` → `mkdocs build --strict`) and the project's PR template
(Overview / Changes / Test / Notes, referencing `Addresses issue #000`).
`CONTRIBUTING.md` does **not** specify a commit-message format, so I followed
**Conventional Commits** (the assignment's recommended default) with a `docs:` prefix —
consistent with the repo's recent "Docs cleanup" history.

### Phase III — Implementation

Built the deliverable from the UMPIRE plan:

- **Added `docs/load_test_failure_modes.md`** — a reliability reference page that
  catalogues each failure mode with: trigger, local repro command, engine code path,
  client-visible symptom, current mitigation, and a recommendation. Covers all three
  modes named in the issue plus the config-coupling gotcha I hit:
  1. Defective / empty provider response → `ModelOutputContractError`, 1 corrective
     retry, then opening-scene failure tears down the session (client sees only
     `Session is closed`).
  2. Rate limiting → `ModelProviderError` with provider `status_code`; 1 retry for
     retryable errors via `_should_retry_llm_error`; documented as needing a live key.
  3. Request stalling under concurrency → tail-latency signal (8 ms → 922 ms turn-phase
     wait at 100 sessions, even with an instant mock model).
  4. Config coupling → `409 Conflict` when run against a `registration_required: false`
     config; fixed by using `load-test.yml`.
- **Registered the page in `mkdocs.yml`** under the User Guide nav.

**Decision:** I scoped Phase III to documentation only (no harness code changes). The
issue is labelled `documentation` and asks to *examine and report* failure modes, and
`scripts/load_test_and_report.py` already records the underlying data. I flagged the
optional harness improvement (bucket errors by class; surface a failure category on the
close frame) as a recommendation in the doc and a follow-up in the plan, to let
maintainers decide whether they want code changes.

### Verification

- `make docs` (`mkdocs build --strict`) → exit 0, no warnings/errors; page renders at
  `site/load_test_failure_modes.html`.
- Re-ran every repro command in the doc against a clean offline server; output matches
  what the page documents.

### Code Changes

- **Files modified / added:**
  - `docs/load_test_failure_modes.md` (new)
  - `mkdocs.yml` (nav entry)
- **Key commit:** `docs: add load test failure modes reference page`
  (`f20cc94`) — https://github.com/abhishekbiju/dcs-simulation-engine/commit/f20cc94
- **Branch:** https://github.com/abhishekbiju/dcs-simulation-engine/tree/fix-issue-208
- **Approach decisions:** offline `--fake-ai-response` repros (no API key/credits);
  docs-only scope; mirrored existing docs-page style and `analyze_results.md` for the
  reporting cross-link.

### Self-review checklist (pre-Phase-IV)

- [x] Change works — page builds and renders; repros verified.
- [x] Docs build clean under `--strict`.
- [x] No unnecessary changes — diff is only the new page + one nav line (build artifacts
      `site/`, `runs/`, `docs/reports/` are gitignored; `personal_contribution.md` is
      kept local via `.git/info/exclude`).
- [x] Clean commit history — single, descriptive Conventional Commit.
- [x] README (this file) updated with implementation summary, branch/commit links, and
      testing notes.

---

## Pull Request

**PR Link:** https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine/pull/259

**PR Title:** docs: examine and document load test failure modes

**PR Description (summary):** Adds `docs/load_test_failure_modes.md`, a reliability
reference cataloguing the load test's failure modes — defective/empty provider
responses, rate limiting, request stalling under concurrency, and the
`registration_required` config-coupling gotcha — each with trigger, offline repro
(`--fake-ai-response`), engine code path, client symptom, mitigation, and a
recommendation; wires the page into the mkdocs nav. Documentation-only, no behavior
changes. Submitted from `abhishekbiju:fix-issue-208` → upstream `main`, `Closes #208`.

**Pre-submission checks (2026-07-01):**
- Branch up to date with `upstream/main` (+1 commit); no rebase needed.
- `git diff upstream/main` reviewed — only the 2 intended files (+194 lines), no stray/debug changes.
- `make docs` (`mkdocs build --strict`) clean; every repro command re-verified against a clean offline server.

**Maintainer Feedback:**
- _None yet — PR opened 2026-07-01, awaiting first review._

**Status:** Awaiting review

**Next steps:**
- Watch the PR's CI checks and respond to review comments within ~24h.
- If asked, open follow-up issues/PRs for the code recommendations noted in the doc
  (surface a failure category on the close frame; add backoff/jitter for 429s).
- Optionally @-mention a maintainer if there's no review within 5–7 business days
  (identify the right reviewer from recent merged docs PRs / CODEOWNERS).

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
