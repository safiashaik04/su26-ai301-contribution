# Contribution 2: Deploy a hosted Streamlit demo + add a 'Try it live' badge

**Contribution Number:** 2 
**Student:** Safia Shaik 
**Issue:** https://github.com/Rekin226/aquascope/issues/34
**Status:** Phase II Complete

---

## Why I Chose This Issue

For my second contribution, I wanted to build on a different skill set than my first (which was CLI/backend work). Issue #34 is about deploying an existing Streamlit dashboard to Streamlit Community Cloud, configuring dependencies and making it work gracefully without API keys. That's a strong match for my DevOps/SRE background, where my focus is on infrastructure and deployment rather than application code.

I was also drawn to the impact: it's the top item on the project's roadmap, since a live, no-install demo turns curious visitors into actual users. I'm hoping to come away with hands-on experience deploying a public-facing app end-to-end, from configuration to graceful degradation to the live deploy itself, which speaks directly to the SRE and DevOps roles I'm targeting.

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

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

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
