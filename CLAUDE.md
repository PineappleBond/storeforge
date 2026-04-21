# StoreForge

Claude Code plugin for e-commerce development with hard-constraint harness engineering.

## Core Capabilities

- **Harness Engineering**: Hard-constraint system. Violation = block current work.
- **Multi-Agent Self-Loop Review**: 5 review agents, max 5 rounds, auto-escalation.
- **Three-Layer Testing**: L1 unit + L2 integration + L3 E2E with coverage thresholds.
- **User Decision Flow**: Technical issues self-resolved, product issues ask user.

## Architecture

- `skills/` — 13 skills, registered as `storeforge-*`
- `agents/` — 5 review agents (architect, security, performance, api-contract, testing)
- `hooks/` — SessionStart hook for P0 rule injection
- `commands/` — 5 shortcuts (`/sf:brainstorm`, `/sf:review`, `/sf:test`, `/sf:plan`, `/sf:verify`)
- `knowledge/ecommerce-patterns/` — 19 ecommerce pattern files (auth, payment, cart, inventory, order-state-machine, etc.)

## Quick Start

1. Install this repo as a Claude Code plugin
2. P0 rules are auto-injected at session start via `hooks/session-start`
3. Use `/sf:brainstorm` to begin an ecommerce project

## Development Guide

- **Naming**: Skills use `storeforge-*` prefix, commands use `sf-` prefix
- **No generic names**: Never use `brainstorming`, `testing`, `review` (conflicts with superpowers)
- **Commit convention**: English only, Conventional Commits (`feat(skill): add storeforge-using`)
- **After changes**: Re-validate plugin by running `python -m json.tool .claude-plugin/plugin.json`

## Version

0.1.0
