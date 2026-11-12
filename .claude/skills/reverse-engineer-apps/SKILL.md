---
name: reverse-engineer-apps
description: Coordinate an evidence-backed reverse engineering investigation and independent rebuild of an authorized target using Grok Bot + REA. Use when the user wants to study a feature/workflow in a web app, native binary, managed app, or data format and produce a clean-room rebuild with parity report.
---

# Reverse Engineer Apps (Grok Bot + REA)

You are the reverse engineering lead. Use REA to study an authorized target, then coordinate an independent rebuild of the selected behavior.

## Inputs

Collect or confirm these before starting:

- **Target**: `<SYSTEM / URL / REPO / LOCAL FILE>`
- **Scope**: `<FEATURE OR WORKFLOW TO REBUILD>`
- **Access**: `<APPROVED ACCOUNTS, FILES AND TOOLS>`
- **Done**: `<EXPECTED BEHAVIOR / ACCEPTANCE TEST>`

## Workflow

### 1. Scope the run
- Check which REA tools and worker agents are available.
- Classify the target as web, native, managed, or data format.
- Start with observable behavior and choose tools that fit the target.

### 2. Assign agents
Assign these roles:
- **Scout** — map workflows
- **Trace** — collect inputs and outputs
- **Spec** — write contracts
- **Build** — implement
- **Verify** — test

Run independent research in parallel when supported. Otherwise run the roles in sequence and say so.

### 3. Collect evidence
- Save captures, sample inputs, outputs, and source locations.
- Mark each claim **OBSERVED**, **INFERRED**, or **UNKNOWN**.
- Use only approved access.
- Keep secrets out of the shared workspace.

### 4. Handoff
- Document states, data shapes, edge cases, and failure behavior in `SPEC.md`.
- Give Build the behavior spec and sanitized fixtures.
- Do not copy original code or assets into the rebuild.

### 5. Compare
- Implement the smallest complete workflow.
- Run the same authorized inputs against both systems.
- Record matching behavior, differences, and checks you couldn't run.

### 6. Deliver
Return:
- The runnable rebuild
- `SPEC.md`
- Evidence
- A parity report

Ask before publishing, deploying, or spending money.

## Guardrails

- Only work on authorized targets with approved access.
- Prefer independent / clean-room rebuilds from observed behavior.
- Never include secrets, credentials, or private data in shared outputs.
- Explicitly label evidence quality (OBSERVED / INFERRED / UNKNOWN).
