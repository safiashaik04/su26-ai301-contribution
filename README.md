# Contribution 1: CLI: add geojson to the collect command's --format option

**Contribution Number:** 1 
**Student:** Safia Shaik 
**Issue:** https://github.com/Rekin226/aquascope/issues/7
**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose the AquaScope --format CLI enhancement (#7) because it fits my background while still pushing me somewhere new. I work mostly in backend and DevOps and Python is one of my strongest languages, so the stack here is a Typer CLI, pandas, and JSON/CSV/GeoJSON. Output is familiar enough that I can get productive fast, and the issue is well scoped: add a --format option and an --output flag, route the output through three format paths, and cover it with tests and docs. 
It also solves a real problem for actual users, researchers who just want their data as a CSV or GeoJSON without writing a script to convert it. Mostly, though, I have little open-source experience, and I want to use this to go through the full contribution loop for the first time like claiming an issue, working with a maintainer, and getting a PR reviewed and merged, while sharpening my testing and edge-case handling along the way.

---

## Understanding the Issue

### Problem Description

The aquascope collect command can only save data in 2 formats: JSON and CSV. It cannot save data as GeoJSON. GeoJSON is the standard format that mapping tools like QGIS and web maps use, so right now there's no easy way to take the water station data and load it straight into a map. The issue asks me to add GeoJSON as a third output format.

### Expected Behavior

Running `aquascope collect --source gemstat --format=geojson` should save the collected data as a valid GeoJSON file. Each record with location data should become a map point (a GeoJSON Point), and records without coordinates should still be saved with their geometry set to null instead of causing an error. The geojson option should also show up when you run aquascope collect --help.

### Current Behavior

Running the command with --format=geojson fails. argparse rejects it with the error invalid choice: 'geojson' (choose from 'json', 'csv'), because geojson was never added as an allowed format. Only json and csv work today.

### Affected Components

- aquascope/cli.py defines the --format option and its allowed choices.
- aquascope/utils/storage.py contains save_records, which decides how data is written for each format.
- tests/test_utils/test_export_formats.py where the new GeoJSON output tests will go, alongside the existing JSON/CSV format tests.
- docs/ the CLI documentation that lists the available formats.

---

## Reproduction Process

### Environment Setup

I ran into two issues while setting up the project locally.
1. When I tried to install with pip, my Mac couldn't find the command, and using python3 -m pip instead gave an externally-managed-environment error. This happens because my Python is managed by Homebrew, which blocks installing packages directly into the system Python. I solved it by creating a virtual environment (python3 -m venv .venv and source .venv/bin/activate) and installing inside it, which is the recommended approach and keeps the project's dependencies isolated.
2. Some data sources need API keys to collect data. Since the issue I'm working on is about the output format and not about any one source, I chose sources that are open-access and need no key (such as GEMStat) to reproduce the behavior. This let me focus on the format problem without dealing with key setup.

### Steps to Reproduce

1. Fork the AquaScope repository to your own GitHub account.
2. Clone your fork locally and cd into the project folder.
3. Create and activate a virtual environment: `python3 -m venv .venv` then `source .venv/bin/activate`.
4. Install the dependencies: pip install -e ".[all,dev]".
5. Run `aquascope collect --source gemstat`. This collects data and saves it as a JSON file under data/raw/.
6. Run `aquascope collect --source gemstat --format=csv`. This works and saves a CSV file.
7. Run `aquascope collect --source gemstat --format=geojson`. This fails with the error invalid choice: 'geojson' (choose from 'json', 'csv'), which confirms the issue: GeoJSON is not a supported output format.

- Note: The issue's acceptance criteria use `aquascope collect --source usgs --format geojson` to verify the fix. The gap is identical across sources because it's in the shared output path, so gemstat demonstrates it equally well; usgs is used for final verification in Phase III.


### Reproduction Evidence

- **Branch Link:** https://github.com/safiashaik04/aquascope/tree/fix-issue-geoJSON-format
- **Commit showing reproduction:** [Link to commit in your fork] "no code change needed to reproduce; reproduction is via CLI commands."
- **Screenshots/logs:**
**1. Default JSON output works.**
  Running `aquascope collect --source gemstat` collects the data and saves it as a JSON file under `data/raw/`.
  
<img width="1470" height="956" alt="Screenshot showing JSON file output" src="https://github.com/user-attachments/assets/b105be8f-60ff-489b-8dc5-8f68274009da" />

**2. CSV format works.**
Running `aquascope collect --source gemstat --format=csv` saves the data as a CSV file.

<img width="1470" height="956" alt="Screenshot showing CSV file output" src="https://github.com/user-attachments/assets/88ca2aef-743b-4ff3-865d-ef9af283ee83" />

**3. GeoJSON format fails (this is the issue).**
Running `aquascope collect --source gemstat --format=geojson` fails with `invalid choice: 'geojson' (choose from 'json', 'csv')`, confirming GeoJSON is not yet a supported format.

<img width="1470" height="956" alt="Screenshot showing error for geoJSON file type" src="https://github.com/user-attachments/assets/96823ccb-5ed1-4a5b-b4ec-c9d90c341389" />


- **My findings:** The --format option only accepts json and csv. Passing geojson fails at the argparse validation stage, before any data is written, confirming the format was never added. The GeoJSON serializer (export_geojson) already exists in storage.py and is tested; the only missing piece is routing fmt="geojson" to it inside save_records() and adding geojson to the CLI's --format choices.
---

## Solution Approach

### Analysis

GeoJSON isn't supported simply because it was never wired in. The --format option in cli.py only allows json and csv, so passing geojson gets rejected right away. The GeoJSON logic itself already exists and is tested (export_geojson() in storage.py); the save_records() function just doesn't have a branch that uses it.

### Proposed Solution

Add geojson to the allowed --format choices in cli.py, and add a branch in save_records() that routes GeoJSON requests to the existing export_geojson() helper. Then add a test for it. No new GeoJSON logic needs to be written.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The aquascope collect command currently supports only json and csv output formats. This issue adds GeoJSON as a third format so collected station data can be loaded directly into GIS tools like QGIS or web maps. Importantly, most of the work already exists in the codebase. The only thing missing is wiring the existing GeoJSON helper into the save path and allowing geojson as a format choice.

**Match:** The patterns and even the core logic already exist. aquascope/cli.py defines --format on the collect subparser with the choices ["json", "csv"]. aquascope/utils/storage.py has save_records() (around line 24) with an if fmt == "json" / elif fmt == "csv" / else: raise ValueError structure. Crucially, the same file already has a working, tested export_geojson(records, path) helper (around line 136) that builds the FeatureCollection and already emits "geometry": null for records without a location. The test file tests/test_utils/test_export_formats.py already covers export_geojson directly and has existing save_records(..., fmt="csv") tests I can mirror.

**Plan:**
1. In aquascope/utils/storage.py, add a geojson branch to save_records(). When fmt == "geojson", build the .geojson filepath and call the existing export_geojson() helper, returning the written path. I will not reimplement the GeoJSON logic, since the helper already handles it (including the no-location case).
2. In aquascope/cli.py, add "geojson" to the choices=[...] list on the collect subparser's --format argument, so it's accepted and appears in --help.
3. In tests/test_utils/test_export_formats.py, add a test for save_records(records, fmt="geojson") following the existing fmt="csv" test pattern, confirming it returns a valid .geojson path.

**Implement:** Branch link: https://github.com/safiashaik04/aquascope/tree/fix-issue-geoJSON-format

**Review:** Following CONTRIBUTING.md: I'll work on a feature branch in my fork, add a test for the new functionality, and make sure pytest, ruff check, and mypy all pass. I'll keep commits atomic with clear messages and reference issue #7 in my pull request description.

**Evaluate:** I'll verify the fix by running aquascope collect --source usgs --format geojson and confirming it writes a valid .geojson FeatureCollection. I'll confirm save_records(records, fmt="geojson") returns the written .geojson path, and that aquascope collect --help now lists geojson as a choice. The no-location case ("geometry": null) is already handled and tested by export_geojson, and my new test in test_export_formats.py will cover the save_records routing.

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
