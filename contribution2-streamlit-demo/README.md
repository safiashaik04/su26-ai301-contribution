# Contribution 2: Deploy a hosted Streamlit demo + add a 'Try it live' badge

**Contribution Number:** 2 
**Student:** Safia Shaik 
**Issue:** https://github.com/Rekin226/aquascope/issues/34
**Status:** Phase Phase IV In Progress — Issue closed by maintainer via independent PR before submission; pivoting to a new issue

---

## Why I Chose This Issue

For my second contribution, I wanted to build on a different skill set than my first (which was CLI/backend work). Issue #34 is about deploying an existing Streamlit dashboard to Streamlit Community Cloud, configuring dependencies and making it work gracefully without API keys. That's a strong match for my DevOps/SRE background, where my focus is on infrastructure and deployment rather than application code.

I was also drawn to the impact: it's the top item on the project's roadmap, since a live, no-install demo turns curious visitors into actual users. I'm hoping to come away with hands-on experience deploying a public-facing app end-to-end, from configuration to graceful degradation to the live deploy itself, which speaks directly to the SRE and DevOps roles I'm targeting.

---

## Outcome: Issue Closed Independently by Maintainer

While my branch (`feat/streamlit-cloud-demo`) was ready for review, the maintainer closed #34 by merging their own PR — [#109 "Dashboard 2.0"](https://github.com/Rekin226/aquascope/pull/109) — which they built solo (with Claude Code), not from my branch or PR. It's a full rebuild of the dashboard from the monolith I was patching into a new multipage `views/` architecture, and it satisfies every acceptance criterion from #34: deploy config, no-API-key sample data, a live "Try it live" badge, and the roadmap item ticked. They ultimately hosted it via Hugging Face's stlite/WASM static tier rather than Streamlit Community Cloud.

Since the file I modified no longer exists in that form, opening a PR from my branch would conflict with the new architecture (starting with `streamlit_app.py` and `requirements.txt` both already existing at the root, added independently in #109) and would add no functionality not already shipped. Rather than force a redundant PR, I'm documenting the completed work here as evidence of independent problem-solving and pivoting to a new issue, per CodePath's guidance.

---

## Understanding the Issue

### Problem Description

There's no way to try AquaScope without installing it first. Someone landing on the README has to clone the repo, set up Python, and install a fairly heavy dependency stack before they can see what the dashboard even looks like — which is a real barrier to converting curious readers into users or stars. This is the top "in progress" item on the roadmap for exactly that reason

### Expected Behavior

A visitor should be able to click a "🚀 Try it live" badge on the README and land directly in a working AquaScope dashboard in their browser, populated with real sample data (from the bundled CAMELS catchments), with no sign-up, no API keys, and no pip install required. Anything in the dashboard that genuinely needs a secret (a data-source API key, an LLM key) should just quietly not be offered, rather than being present but broken.

### Current Behavior

There is no hosted demo at all — the badge doesn't exist, and even if someone tried to deploy the existing code as-is to Streamlit Community Cloud, it would fail or be broken in a few ways: there's no entry-point file at the repo root for Cloud to build from, the minimal install path is missing dependencies the plotting pages need, the dashboard has no concept of "I'm a public instance with no secrets" so it can't hide the two collectors and the LLM mode that need keys, and every page starts empty until a visitor manually clicks a "Load demo dataset" button — including on a page that's supposed to be a bundled sample of real catchment data that the app never actually reads from disk.

### Affected Components

- `aquascope/dashboard/app.py` — the dashboard itself; needed new hosted-mode detection, CAMELS data loading, auto-seeding, and collector/LLM gating.
- Repo root — missing `streamlit_app.py` (deploy entry point) and `requirements.txt` (deploy dependencies).
- `pyproject.toml` — the dashboard extra alone doesn't cover what the dashboard's plotting pages actually import.
- `data/camels_benchmark/` — bundled sample data that existed but was disconnected from the dashboard entirely.
- `README.md` / `docs/index.md` — no live-demo badge or link.
- `ROADMAP.md` — item still unchecked.

---

## Reproduction Process

### Environment Setup

No devcontainer.json in the repo, so setup followed CONTRIBUTING.md directly: created/activated .venv and ran `pip install -e ".[dev,all]"`. Verified the baseline was green before touching anything — ruff check aquascope/ tests/, pytest, and mypy all ran clean on a fresh checkout. No dependency errors hit during setup. One extra check specific to this issue: tested the deploy-facing requirements.txt (.[dashboard,viz]) in a separate scratch venv to confirm it installs cleanly on its own, since that's the subset Streamlit Community Cloud will actually install (not the full [dev,all] set).

### Steps to Reproduce

This is a missing-capability issue, not a crash bug, so for "reproducing steps" I confirmed the gaps described in the issue actually exist

1. Clone the repo — confirm there is no top-level `streamlit_app.py` and no root `requirements.txt`, so Streamlit Community Cloud has nothing to build from out of the box.
2. Run `aquascope dashboard` locally with a full dev install — works fine.
3. Now simulate a plain `pip install aquascope[dashboard]` (the extra Community Cloud would install). The dashboard extra only pulls in streamlit + streamlit-folium, not matplotlib/seaborn/folium. Open the Visualization or Hydrology page - `matplotlib.pyplot` import is unguarded, so it throws `ModuleNotFoundError` on nearly every plotting page.
4. Check `data/camels_benchmark/` — 10 real named catchment CSVs + metadata exist, but grepping the dashboard code shows they're never read anywhere; every "demo data" button generates synthetic data on the fly instead.
5. Check ROADMAP.md — "Hosted Streamlit demo (try without installing)" is confirmed as the top unchecked item under "In progress."

Reproduced consistently (these are static file/code-structure gaps, not flaky runtime behavior, so re-checking twice just confirms the same absence).

Branch Link: https://github.com/safiashaik04/aquascope/tree/feat/streamlit-cloud-demo 

### Reproduction Evidence

- **Commit showing reproduction:** Comfirmed the gaps exist in local.
- **Screenshots/logs:**
  <img width="1470" height="956" alt="aquascope dashboard" src="https://github.com/user-attachments/assets/25b3ff94-1d2a-4334-88e7-3ceaeb6e852c" />

  <img width="1470" height="956" alt="analysis" src="https://github.com/user-attachments/assets/ad59d6ce-2928-42c6-946e-187839a2a7c1" />

  <img width="2940" height="1912" alt="analysis eda report" src="https://github.com/user-attachments/assets/091eeeb1-01cb-4f32-b954-775da60da3f9" />


- **My findings:** 
The dashboard itself (`aquascope/dashboard/app.py`) is fully built and already has some good bones for a graceful demo — synthetic "Load demo dataset" buttons that need no network or keys, and a dict (`_API_KEY_SOURCES`) that already flags which of the 15 collectors need a key. But nothing wires it up for actual hosting: there's no `streamlit_app.py` or `requirements.txt` at the repo root for Community Cloud to build from, and the dashboard extra in `pyproject.toml` is missing `matplotlib`/`seaborn`/`folium`, which the plotting pages import unconditionally — so a bare `pip install aquascope[dashboard]` would crash on nearly every visualization page. The bundled `data/camels_benchmark/` sample data (10 real named catchments) is completely unused by the dashboard; every demo button generates fake data on the fly instead. There's also no concept of "hosted vs. local" anywhere in the code, so there was no way for the app to know it should behave differently (auto-load data, hide key-gated features) when running as a public demo versus on someone's own machine.

---

## Solution Approach

### Analysis

The root cause is that the dashboard was built and tested only for local use (`aquascope dashboard`), so nobody had a reason yet to add a deployment entry point, tighten the dependency list for a minimal install, or teach the app to distinguish "I'm a public demo with no secrets" from "I'm running on my maintainer's laptop." The bundled CAMELS data existing but going unused is really the same root cause from a different angle — it was added for regression tests, not surfaced to the dashboard layer at all, so nobody wired it in.

### Proposed Solution

Add the missing deploy scaffolding (`streamlit_app.py` shim + `requirements.txt` with the right extras) so Community Cloud can actually build the app, then teach the dashboard to detect when it's running as the hosted public demo versus a local run. In hosted mode: default the streamflow-related pages to the real bundled CAMELS catchments instead of synthetic data, auto-seed sample data everywhere so no page is ever empty for a first-time visitor, and hide the two API-key-gated collectors plus the LLM-enhanced mode since no secrets exist there. Locally, none of this changes — same buttons, same empty states, same full collector list — so the fix is purely additive rather than a behavior change for existing users.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** AquaScope has no low-friction way for a visitor to try the dashboard without installing anything first. The roadmap's top in-progress item asks for a hosted Streamlit demo that works out of the box on bundled sample data, with no API keys, and degrades cleanly when secrets aren't configured.

**Match:** The codebase already had partial patterns to build on: `_load_demo_data()` / `_load_demo_streamflow()` already show a "no-keys-needed synthetic sample data" pattern (just not backed by the real bundled CAMELS files); `_API_KEY_SOURCES` already flags which of the 15 collectors need a key; `_render_llm_config()` already defaults to rule-based scoring when no key is given. Rather than build new patterns, I extended these existing ones.

**Plan:** 
1. Add a root `streamlit_app.py` shim that imports and calls `aquascope.dashboard.app.main()` — Community Cloud's required entry point.
2. Add a root `requirements.txt` (.[dashboard,viz]) so Cloud actually installs matplotlib/seaborn/folium alongside streamlit.
3. Add an `_is_hosted_demo()` helper that distinguishes the public Cloud deployment (Cloud's `/mount/src` checkout path + no secrets configured, or an explicit `?demo=1` override) from a normal local `aquascope dashboard` run.
4. Wire the real bundled `data/camels_benchmark/` catchments into Hydrology, Extreme Events, and Flow Signatures as a named-catchment picker, replacing the purely synthetic generator as the default. Analysis/Visualization/AI Recommender/Alerts keep the existing synthetic water-quality set, since CAMELS has no water-quality parameters.
5. Auto-seed sample data on hosted mode so no page is ever empty for a first-time visitor — local behavior stays button-driven and unchanged.
6. Hide the 2 key-gated collectors (Taiwan MOENV, Copernicus) and disable LLM-enhanced mode, hosted-mode only — the other 13 public collectors stay fully live.
7. Add a "🚀 Try it live" badge + link to `README.md` and `docs/index.md`.
8. Check off the roadmap item.

**Implement:** https://github.com/safiashaik04/aquascope/tree/feat/streamlit-cloud-demo 

**Review:** self-reviewed against `CONTRIBUTING.md` — branched off main rather than committing to it, kept commits atomic per logical change (deploy config / CAMELS wiring / auto-seed / hosted-mode gating), ran `ruff check`, `pytest`, and scoped `mypy` before each commit, will reference the issue number in the PR description.

**Evaluate:** full test suite passes (1005 passed, 1 skipped, no regressions), `ruff check` clean, `mypy` clean apart from 2 pre-existing unrelated findings. Manually smoke-tested both modes locally (`streamlit run streamlit_app.py` normally, and with `?demo=1` to simulate hosted mode) to confirm auto-seeding, the CAMELS picker, and the hidden collectors/LLM tab all behave correctly, while plain local runs are provably unchanged. The two hard-to-reach UI fallback branches in Flow Signatures were verified programmatically by calling the function directly with a stubbed Streamlit object. 

---

## Testing Strategy

### Unit Tests

- No new pytest files were added for this change. aquascope/dashboard/app.py is explicitly excluded from coverage requirements in pyproject.toml (omit = ["aquascope/dashboard/*"]), consistent with the project's existing convention — there were no dashboard tests before this change either. Instead, the new logic was verified with targeted ad hoc scripts against the real functions:
- [X] `_camels_catchments()` / `_load_camels_streamflow()` — confirmed all 10 bundled catchments load with correct `date/discharge/precipitation` columns and real names.
- [X] `_is_hosted_demo()` — confirmed it returns `True` only when the Community Cloud path signal is present and no secrets are set (or ?demo=1 is passed), and returns `False` for a plain local run with no secrets, proving local behavior can't regress even though local runs also lack secrets.
- [X] Flow Signatures' two fallback branches (no date column / too few rows) — triggered directly by calling `_hydro_signatures()` with a stubbed Streamlit object and crafted DataFrames, since these branches are hard to reach by hand through the UI.

### Integration Tests

- [X] End-to-end: streamlit run streamlit_app.py against the actual requirements.txt in an isolated venv, to approximate what Community Cloud's build step will do.
- [X] Full page click-through in both modes (see Manual Testing).

### Manual Testing

Ran the app locally in two modes and compared behavior side by side:

- **Plain local** (`streamlit run streamlit_app.py`, no query param): confirmed every page behaves exactly as before this change — empty states with buttons, full 15-source Data Collection dropdown, full LLM provider picker.
- **Simulated hosted** (same command, `?demo=1` query param): confirmed Analysis/Visualization/AI Recommender/Alerts land pre-populated with the synthetic water-quality demo set; Hydrology lands pre-populated with a named CAMELS catchment; Extreme Events defaults to the CAMELS catchment picker with zero clicks; Data Collection hides Taiwan MOENV and Copernicus CDS (13 of 15 sources shown); AI Recommender shows a "disabled on this public demo" message instead of the LLM expander.
- Final manual check still pending: confirming `_is_hosted_demo()` fires correctly on real Streamlit Community Cloud infrastructure once deployed (the local test only simulates it via the query param and can't fully replicate Cloud's actual checkout path).

---

## Implementation Notes

### Week 1 Progress

Built the full deploy-and-degrade path. Started by validating the issue against the actual repo state before writing anything (confirmed the missing entry point, the `dashboard` extra's missing viz deps, and that the bundled CAMELS data was completely unused). Went back to the maintainer while building for two clarifications rather than guessing: whether CAMELS data should apply to every page (it can't — no water-quality columns) or just the streamflow ones, and what "default to bundled sample data" actually meant (auto-load with no click, not just "no keys required"). Both answers changed the design meaningfully, so raising them early avoided rebuilding later.

The maintainer's suggested hosted-mode detection ("absence of secrets") would have also matched every ordinary local run, since local runs don't have secrets configured either, that would have silently broken local `aquascope dashboard` behavior. Caught this before implementing and added a second signal (Streamlit Community Cloud's distinctive `/mount/src` checkout path) so local runs stay provably unaffected.

Sandboxed testing had real limits — no PyPI access and an incompatible pre-built `.venv` meant `ruff/mypy/pytest/streamlit` couldn't run directly in the dev sandbox. Worked around it by unit-testing the pure-logic pieces directly in plain Python, and running the actual CONTRIBUTING.md-mandated checks (`ruff`, `pytest`, `mypy`) locally on my own machine instead.


### Code Changes

- **Files modified:** `streamlit_app.py` (new), `requirements.txt` (new), `aquascope/dashboard/app.py`
- **Key commits:** 

      - `485ea49` — root `requirements.txt` for Community Cloud (.[dashboard,viz])
      - `623b4eb` — root `streamlit_app.py` entry-point shim
      - `33bb94b` — wire bundled CAMELS catchments into Hydrology/Extreme Events/Flow Signatures
      - `e4890e0` — auto-seed sample data on hosted mode
      - `be4acdc` — hide key-gated collectors (Taiwan MOENV, Copernicus) on hosted mode
      - `c7363fc` — disable LLM-enhanced mode on hosted mode
- **Approach decisions:** 
    - Kept the synthetic generators (`_load_demo_data()`, `_load_demo_streamflow()`) as fallbacks rather than deleting them, so the app degrades gracefully even if `data/camels_benchmark/` is ever absent (e.g. a PyPI install without the repo's `data/` directory).
    - Seeded per page-family rather than globally at app start, since Hydrology's ideal default (a CAMELS basin) and the other pages' ideal default (synthetic water-quality data) can't share one session-state slot.
    - Only hid the 2 key-gated collectors, not all 15 — the other 13 hit public APIs with no auth and work fine on a hosted instance, so hiding them would have removed a genuinely working feature for no reason.

---

## Pull Request

**PR Link:** Not submitted — superseded before submission (see Outcome above).

**PR Description:** N/A.

**Maintainer Feedback:**
- Issue #34 closed via the maintainer's own PR #109, which independently rebuilt the dashboard and deployed it via Hugging Face static hosting (stlite/WASM), satisfying all of #34's acceptance criteria.
- Reached out to CodePath support for guidance on a closed-but-claimed issue; advised to ask the maintainer about still submitting, and to start a new issue in parallel rather than wait.

**Status:** Not submitted — closed by independent maintainer merge. Pivoting to a new issue.


---

## Learnings & Reflections

### Technical Skills Gained
- Deploy-config engineering for Streamlit Community Cloud: entry-point shims, scoping dependency extras for a minimal hosted install.
- Designing environment-aware code (hosted vs. local detection) without changing existing local behavior including catching a real gap in the maintainer's own suggested detection logic before implementing it, rather than assuming their answer was complete.
- Handling a real data mismatch (bundled CAMELS data covers streamflow only, not water quality) by scoping the fix to where it actually applied instead of forcing one dataset to cover everything.
- Verifying Streamlit UI logic without a running server, by stubbing the `st` object and calling page functions directly useful when the dev sandbox couldn't run the real app.
- Asking targeted clarifying questions before building, instead of guessing at ambiguous instructions.

### Challenges Overcome
- Dev sandbox had no PyPI access and an incompatible `.venv`, so verified logic through direct Python calls with stubbed dependencies, then ran the actual CONTRIBUTING.md-mandated checks (`ruff`, `pytest`, `mypy`, `streamlit run`) locally instead.
- The real challenge came at the end: the issue was closed by a much larger, independent maintainer PR while my branch was ready, with no window to submit first. Confirmed this by reading the actual merged PR rather than assuming a PR might still land, so I could make a clear-eyed call instead of sinking more time into a branch that wouldn't be merged.

### What I'd Do Differently Next Time
- Open a draft PR earlier, even before every acceptance criterion was fully polished an early draft is a low-cost signal to a maintainer that a claimed issue is actively being worked, which might prompt coordination instead of a maintainer solving it solo.
- For roadmap-priority issues especially (the ones most likely to draw the maintainer's own attention), check in mid-way rather than only at claim-time and delivery-time.

---

## Resources Used

- [Issue #34](https://github.com/Rekin226/aquascope/issues/34)
- [PR #109 — maintainer's independent solution](https://github.com/Rekin226/aquascope/pull/109)
- [Streamlit Community Cloud deploy docs](https://docs.streamlit.io/deploy/streamlit-community-cloud/deploy-your-app/deploy)
- `CONTRIBUTING.md` (project's own contribution guidelines)