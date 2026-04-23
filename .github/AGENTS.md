# Workspace AGENTS

This file summarizes the custom agents available for this repository and points to the detailed design documents.

Agents

- `product-designer` — product requirements, PRDs, prioritization, acceptance criteria, and stakeholder-facing artifacts. See `.github/agents/product-designer.agent.md`.
- `software-architect` — repository architecture, file-level change plans, CI/test design, and migration steps. See `.github/agents/software-architect.agent.md`.

Location of canonical design docs

- Long-form design docs and decision memory live in `doc/design/` (see `doc/design/design.md`).
- Use `.github/AGENTS.md` for quick discoverability in the repo root / GitHub UI; keep `doc/design/` for archival and detailed reasoning.

Usage notes

- Agents under `.github/agents/` are focused, small, and have explicit `tools` lists.
 - If you change agent behavior or scope, update both the `.github/agents/*.agent.md` file and `doc/design/design.md`.
