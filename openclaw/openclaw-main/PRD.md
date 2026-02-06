# Candy - PRD

## Problem Statement
Build "Candy" — an AI orchestration and device control layer on top of the existing OpenClaw (formerly MoltBot) codebase. Candy adds safe, permission-controlled device/system orchestration via explicit adapters.

## Base Codebase
- **OpenClaw** v2026.2.4 — TypeScript/ESM monorepo, Node 22+, pnpm
- Multi-channel messaging (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Teams, WebChat, Matrix, Zalo)
- Gateway WebSocket control plane, Pi agent runtime, skills system, browser control, voice, canvas, cron
- Extracted to `/app/openclaw/openclaw-main/`

## User Persona
- Solo founder, building with AI-assisted dev
- Wants to add device control layer to existing AI assistant

## Core Requirements
- Permission engine for device actions
- Safety checker + confirmation flows (all channels)
- Adapter system (Home Assistant, PC Agent, HTTP APIs)
- Execution planner with rollback
- Audit trail for all device actions
- Multi-tenant SaaS layer (billing, isolation)

## What's Been Implemented
- [2026-01-xx] Created comprehensive phased roadmap (`/app/candy-roadmap.md`)
  - 9 phases (0-8) mapped to actual OpenClaw architecture
  - Leverages existing skills, channels, gateway, agent runtime
  - Candy implemented as OpenClaw extension + skill, not a fork

## Architecture
- Candy = OpenClaw skill (`skills/candy/`) + Gateway extension (`extensions/candy/`)
- Hooks into existing Pi agent runtime and tool execution pipeline
- Adapters: Home Assistant, PC Agent (via node.invoke), HTTP API
- SaaS layer: separate service for multi-tenancy, billing (Stripe)

## Backlog
- P0: Phase 0 — Candy skill skeleton + audit foundation
- P0: Phase 1 — Permission engine + safety checker
- P1: Phase 2 — Confirmation flow + execution planner
- P1: Phase 3 — Adapter system (Home Assistant + HTTP)
- P2: Phase 4 — PC Agent + node integration
- P2: Phase 5 — Multi-tenant SaaS
- P3: Phase 6-8 — Automations, hardening, platform maturity

## Next Tasks
- Review roadmap
- Begin Phase 0: Create Candy skill + extension scaffold
- Set up audit SQLite store
