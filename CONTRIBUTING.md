# Contributing to Infinidream

Thank you for your interest in contributing! This document explains
how to contribute and the requirements for doing so.

## Contributor License Agreement

Before we can accept your pull request, you must sign our [Individual
Contributor License Agreement](CLA.md) (CLA). The CLA grants us the
rights to use your contribution while you retain ownership of your
work.

**Why do we require a CLA?** Infinidream is open source. The CLA ensures
we can maintain, relicense, and distribute the project while protecting both
contributors and users.

## How to Contribute

1. **Fork** the repository and create a branch from `main`.
2. **Make your changes.** Follow the existing code style and conventions.
3. **Test** your changes. Run the relevant test suite for the repo you're
   modifying (see [AGENTS.md](AGENTS.md) for per-repo commands).
4. **Commit** with a clear message describing what and why.
5. **Open a pull request** against `main`. Describe what your PR does and link
   any related issues.

## Code Style

- **TypeScript/Node repos** (backend, frontend, worker): Follow existing ESLint
  config. Run `pnpm run lint:fix` before submitting.
- **Python repos** (video, gpu-container, engines, python-api): Follow PEP 8.
- **Next.js** (landing-page): Use Biome for linting and formatting
  (`pnpm run biome:check`).

## Reporting Issues

Open an issue on the relevant repository. Include:
- What you expected to happen
- What actually happened
- Steps to reproduce
- Relevant logs or screenshots

## Questions?

Open a discussion or issue and we'll be happy to help.
