# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

An opinionated AI agent skill that teaches LLMs to build production Ethereum dApps correctly. It's a **meta-router**: one root `SKILL.md` routes to five phase-specific sub-skills, loading only what's needed to keep context lean.

Target consumers are AI agents (Claude Code, Cursor, Copilot), not human developers directly.

## No Build/Test Commands

This is a pure knowledge skill — no dependencies, no build system, no test suite. All files are markdown with YAML frontmatter. To check content size: `wc -c <file>`.

## Architecture

```
SKILL.md              ← Meta router (~4KB), always loaded first
├── contracts/SKILL.md   Phase 1: Solidity + Foundry gotchas
├── testing/SKILL.md     Phase 2: What agents forget to test
├── security/SKILL.md    Phase 3: Audit checklist + historical hacks
├── frontend/SKILL.md    Phase 4: Next.js + wagmi v2 + viem
├── deploy/SKILL.md      Phase 5: Production deployment
├── docs/SPEC.md         Full specification (maintainer reference)
└── README.md            Human-facing overview
```

**Routing flow:** Meta SKILL.md detects what the user wants → loads the relevant phase sub-skill → agent follows that phase's instructions → moves to next phase or exits.

**Full build sequence:** phases 1→2→3→4→5 sequentially. Partial workflows (e.g., "review security") load only the relevant sub-skill.

## Key Conventions

- **Route, don't bloat** — Never load all 5 sub-skills at once. The meta router picks the right one.
- **Only correct what LLMs get wrong** — Don't teach Solidity basics; teach gotchas (wrong decimals, missing approve flows, blank chains).
- **Fork mode always** — `anvil --fork-url`, never blank local chains. This is the #1 rule.
- **Never hallucinate addresses** — Fetch verified addresses from `https://ethskills.com/addresses/SKILL.md`.
- **Nothing is automatic on Ethereum** — Every function needs an incentive for someone to call it.
- **Security embedded, not optional** — Safe code is the default writing style, not an add-on phase.

## Content Constraints

- Meta SKILL.md: ~3-4KB target
- Sub-skill targets: 150 lines (testing/security/deploy), 200 lines (contracts/frontend)
- Hard max: 250 lines per file — if exceeded, split or cut
- Each SKILL.md has YAML frontmatter (`name`, `description`) for skills.sh packaging

## Default Stack (taught by the skill)

| Layer | Tool |
|-------|------|
| Contracts | Foundry + OpenZeppelin v5 + Solidity >=0.8.20 |
| Frontend | Next.js + wagmi v2 + viem + RainbowKit |
| Local dev | Anvil fork of mainnet |
| Deploy | Foundry (contracts), Vercel or IPFS (frontend) |

## When Editing Content

- Changes to sub-skills should reflect patterns from `docs/SPEC.md` — it's the authoritative spec.
- Every gotcha should pair with "why this matters" (ideally a historical hack reference).
- Include a troubleshooting table and exit criteria checklist in every phase sub-skill.
- The `.gitignore` excludes `*.skill` packaged files.
