---
mode: agent
description: Generate a cross-platform Python script to export KiCad project artifacts (Gerbers, Excellon, STEP, PDFs, BOM) for local use and CI (GitHub Actions).
---

Goal
Create a Python 3.10+ command-line tool that exports standard KiCad project artifacts using KiCad’s CLI (kicad-cli). It must run locally (Windows/macOS/Linux) and in GitHub Actions on tag builds. Provide a README and an example config.

Assumptions and constraints
- KiCad version: prefer KiCad 9.x (fall back to 8.x if feasible). Use kicad-cli for headless exports.
- Primary OS targets: Windows and Ubuntu (macOS optional).
- Project identification: given a project directory, detect the single .kicad_pro there (or accept --project-name to disambiguate).
- Default timestamp: UTC in format yyyyMMdd-HHmm when no tag is supplied.
- Filenames should avoid spaces; use underscores.

Inputs
- Required: --project-dir (path to folder containing .kicad_pro), optional --project-name (basename of the KiCad project without extension).
- Optional: --tag (git tag or build label). If omitted, use UTC timestamp yyyyMMdd-HHmm.
- Optional: --config (path to YAML config overriding export options).
- Optional: --out-dir (default: Exports/{{project-name}}_{{tag}}/).
- Optional: --verbose / --quiet.

Outputs (all under Exports/{{project-name}}_{{tag}}/)
- Gerbers + Excellon + map zipped: {{project-name}}_{{tag}}_gerbers.zip
- Board STEP: {{project-name}}_{{tag}}.step
- Combined schematics PDF: {{project-name}}_{{tag}}.pdf
- PCB layer stack PDF (one layer per page): {{project-name}}_{{tag}}_PCB.pdf
- BOM CSV: {{project-name}}_{{tag}}_BOM.csv

Implementation details
- Use kicad-cli where possible:
  - Gerbers/Drill: kicad-cli pcb export gerbers / drill (then zip outputs).
  - STEP: kicad-cli pcb export step.
  - PCB PDF: kicad-cli pcb export pdf with layer selection from config; one layer per page.
  - Schematics PDF: kicad-cli sch export pdf (combine all sheets).
  - BOM: prefer kicad-cli sch export bom if available in KiCad 8; otherwise call a BOM plugin (e.g., bom_csv_grouped_by_value) with configurable plugin name/path and options.
- Provide a single config file (YAML) with sensible defaults; allow overrides via CLI:
  - gerbers: plot layers, drill options, gerber precision, subtract soldermask from paste, tenting, etc.
  - pcb_pdf: page size (A4/Letter), layers list or “all”, include drills, black/white, border. Keep to 100% scale.
  - schematics_pdf: page size, monochrome/color, include title block.
  - step: include tracks/zones, units, model resolution. Ensure origin is in the board center.
  - bom: plugin name, plugin args, fields to include, group-by keys.
  - general: clean_output (true/false), zip_gerbers (true/false).
- Validate prerequisites:
  - Check kicad-cli is on PATH and report version.
  - On Windows, support KiCad default install path if PATH is missing (optional).
- Behavior:
  - Create output directory if missing; if clean_output=true, remove existing contents first.
  - Fail fast with clear messages and non-zero exit codes.
  - Log all invoked commands and write a manifest.json summarizing inputs, outputs, versions, and config used.

CLI examples
- Local: python export_kicad.py --project-dir . --tag v1.2.3
- Auto-detect name: python export_kicad.py --project-dir c:\proj\BoardA
- With config: python export_kicad.py --project-dir . --config export.yaml

CI (GitHub Actions) requirements
- Provide .github/workflows/release-artifacts.yaml that:
  - Triggers on push tags: "*"
  - Checks out code with tags.
  - Installs KiCad 9 (Ubuntu via apt or a KiCad Docker image; Windows via chocolatey optional).
  - Runs the script with --tag ${{ github.ref_name }}.
  - Uploads Exports/{{project-name}}_{{ github.ref_name }} as an artifact and optionally attaches to a GitHub Release.
- Document environment variables and permissions required.

Deliverables
- export_kicad.py (Python script)
- export.yaml (example config with commented options)
- README.md with:
  - Prereqs (KiCad version, how to install kicad-cli on each OS, Python version)
  - Usage examples (local and CI)
  - Config reference (all keys with defaults)
  - Troubleshooting (common KiCad CLI issues and plugin paths)
- Optional: tests for config parsing and filename generation.

Acceptance criteria
- Running locally with only --project-dir against a valid KiCad 8 project produces all five artifacts with exact filenames and in the expected folder.
- CI workflow runs on a tag and uploads the same artifacts.
- Changing config (e.g., layer list, paper size) affects outputs as expected.
- Script exits non-zero on missing KiCad CLI or missing project files and prints actionable errors.

