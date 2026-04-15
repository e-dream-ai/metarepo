# metarepo

Coordinating repository for the **Infinidream** (e-dream / infinidream.ai) project — a generative AI platform for creating and managing animated dream content.

Infinidream is built as a set of specialized repositories rather than a monorepo. This metarepo is the place where those repositories come together: it holds cross-cutting documentation, design plans, shared policies, and tooling that span more than one service.

## What lives here

- `docs/` — design documents, plans, security scans, and other cross-repo notes
- `CLA.md`, `CONTRIBUTING.md`, `LICENSE` — project-wide policies
- `AGENTS.md` / `CLAUDE.md` — guidance for AI coding assistants working across the project
- Sibling checkouts of the component repositories (see below), cloned alongside this one for convenience

This repo deliberately does **not** contain application code. Each service lives in its own repository and is developed, built, and deployed independently.

## Cross-cutting documentation

Design documents and plans that span multiple services live in [`docs/plans/`](docs/plans/). Anything that can't naturally be owned by a single component repository — architecture decisions, security reviews, migration plans — should land here.
