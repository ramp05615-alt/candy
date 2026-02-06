# Candy: Phased Build Plan

## Orchestration & Device Control Layer for OpenClaw

> Candy is **not** a new project. It is an orchestration layer added on top of **OpenClaw** (formerly MoltBot/ClawdBot) — an existing, mature personal AI assistant platform. OpenClaw already handles multi-channel messaging (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Teams, etc.), a Gateway WebSocket control plane, Pi agent runtime, skills system, browser control, voice, and canvas. Candy adds the missing piece: **safe, permission-controlled device and system orchestration** via explicit adapters.

**Operator**: Solo founder + AI-assisted development (Emergent)
**Codebase**: TypeScript/ESM monorepo, Node 22+, pnpm, Vitest
**Philosophy**: Boring, reliable engineering. No magic. No autonomous execution. Human always confirms physical actions.

---

## What OpenClaw Already Provides (Inherited)

Before defining what to build, here's what Candy gets **for free** from OpenClaw:

| Capability | Status | Where |
|-----------|--------|-------|
| Multi-channel messaging | Done | WhatsApp (Baileys), Telegram, Slack, Discord, Signal, iMessage, Teams, WebChat, Matrix, Zalo |
| Gateway WebSocket control plane | Done | `src/gateway/`, sessions, presence, config, cron, webhooks |
| Pi agent runtime (RPC) | Done | `@mariozechner/pi-agent-core`, tool streaming, block streaming |
| CLI surface | Done | `openclaw onboard`, `openclaw agent`, `openclaw message send` |
| Skills platform | Done | `skills/` — bundled, managed, workspace skills |
| Browser control | Done | `src/browser/` — Playwright-based CDP control |
| Voice Wake + Talk Mode | Done | ElevenLabs, macOS/iOS/Android |
| Canvas + A2UI | Done | `src/canvas-host/` |
| Media pipeline | Done | Images, audio, video, transcription |
| Cron + webhooks | Done | `src/cron/`, `src/hooks/` |
| Session management | Done | Main sessions, group isolation, per-channel routing |
| Security model | Done | DM pairing, sandbox modes, Docker sandboxing |
| Multi-agent routing | Done | `src/agents/` — route channels to isolated agents |
| Companion apps | Done | macOS, iOS, Android nodes |
| Config system | Done | `src/config/` — JSON5 config, Zod validation |

**What Candy needs to ADD:**
- Permission engine for device/system actions
- Safety checker + confirmation flows
- Adapter system (Home Assistant, PC Agent, HTTP APIs, IoT)
- Execution planner with rollback
- Audit trail for all device actions
- Multi-tenant SaaS layer (billing, isolation, onboarding)
- Device control skills
- Control dashboard UI

---

## Architecture: Where Candy Fits

```
Existing OpenClaw Stack:
========================
  Channels (WhatsApp/Telegram/Slack/etc.)
         |
         v
  [Gateway WS Control Plane]
         |
         v
  [Pi Agent Runtime] <-- Understands intent, plans actions
         |
         v
  [Skills + Tools] <-- Existing: bash, browser, canvas, cron, etc.


Candy Additions (NEW):
======================
  [Pi Agent Runtime]
         |
         v
  [Candy Orchestration Skill] <---- NEW: intercepts device/system actions
   |        |          |
   v        v          v
  Perm    Safety    Confirm
  Engine  Checker   Manager
         |
         v
  [Execution Planner] <---- NEW: breaks intent into steps
         |
         v
  [Adapter Registry] <---- NEW: routes to correct adapter
   |        |          |
   v        v          v
  Home     PC       HTTP
  Asst.   Agent     API
  Adapter  Adapter  Adapter

  [Audit Log] <---- NEW: records everything
```

Key insight: Candy is implemented as **OpenClaw skills + a gateway extension**, not a fork or a parallel system. It hooks into the existing tool/skill execution pipeline.

---

## Phase 0: Candy Skill Skeleton + Audit Foundation

### Goal
Create the Candy orchestration skill that plugs into OpenClaw's existing skill system. Set up the audit log. Get a basic command -> parse -> log flow working through WhatsApp/any existing channel.

### In Scope
- **Candy skill** (`skills/candy/SKILL.md`) that registers with OpenClaw
- **Candy gateway extension** (`extensions/candy/`) for adapter management and orchestration API
- Audit log storage (SQLite, same pattern as OpenClaw's memory/session storage)
- Basic intent classification: is this a device/system action or a regular chat?
- Structured `CandyAction` type: action, target, parameters, source_channel, user_id
- Action logging: every classified device action is logged with timestamp, user, channel, raw input
- Config schema addition: `candy` section in `openclaw.json` for adapter configs
- Basic test suite following OpenClaw's Vitest patterns

### Out of Scope
- Actual device control (no adapters yet)
- Permissions beyond OpenClaw's existing DM pairing
- Confirmation flows
- Multi-tenancy
- Dashboard UI

### Key Components
| Component | Description |
|-----------|-------------|
| `skills/candy/SKILL.md` | Candy skill prompt — teaches the agent about device orchestration |
| `extensions/candy/` | Gateway extension for Candy APIs |
| `extensions/candy/package.json` | Plugin manifest |
| `extensions/candy/src/index.ts` | Extension entry point |
| `extensions/candy/src/audit.ts` | Audit log writer (SQLite) |
| `extensions/candy/src/types.ts` | CandyAction, CandyConfig, AdapterCapability types |
| `extensions/candy/src/classifier.ts` | Intent classifier: device action vs. regular chat |
| `src/config/types.candy.ts` | Candy config schema types |

### APIs Introduced (Gateway Extension Routes)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/candy/health` | Candy extension health |
| GET | `/api/candy/audit` | Query audit log |
| GET | `/api/candy/status` | Candy system status |

### Safety & Permission Rules
- Candy skill inherits OpenClaw's existing security model (DM pairing, allowlists)
- All device-classified intents are logged before any action is taken
- Audit log is append-only, SQLite WAL mode for concurrent reads
- No device action is possible yet — classifier only labels and logs

### Expected Output
User sends "Turn off the kitchen lights" via WhatsApp. The Pi agent, guided by the Candy skill prompt, classifies this as a device action, logs it to the audit trail, and responds: "I understand you want to turn off the kitchen lights. Candy device control is not yet configured. To set up adapters, see [docs]."

Regular chat ("What's the weather?") flows through unchanged.

### Risks & Failure Modes
- **Skill loading conflicts**. Candy skill must not interfere with existing skills. Test with common skill combinations.
- **Extension bootstrap order**. OpenClaw loads extensions after core — ensure Candy's routes don't shadow Gateway routes.
- **SQLite contention**. OpenClaw already uses SQLite for memory. Keep Candy's DB in a separate file (`~/.openclaw/candy/audit.db`).

---

## Phase 1: Permission Engine + Safety Checker

### Goal
Build the permission and safety layers that sit between intent classification and execution. No device is controlled yet, but every action request is validated against permissions and safety rules before proceeding.

### In Scope
- **Permission engine**
  - Per-user device permissions stored in Candy's config/DB
  - Permission types: `device:control`, `device:query`, `automation:create`, `automation:delete`
  - Target scoping: user X can control `lights.*` but not `locks.*`
  - Default deny: no explicit permission = blocked
  - Permission CRUD via CLI (`openclaw candy permissions set/get/list`)
- **Safety checker**
  - Blocklist of dangerous action patterns (configurable, not hardcoded)
  - Rate limiting: max N device commands per minute per user
  - Conflict detection: contradictory actions within time window
  - Time-based restrictions: optional quiet hours
- **Action validation pipeline**: classify -> check permissions -> check safety -> return verdict
  - Verdict: `ALLOWED`, `DENIED_PERMISSION`, `DENIED_SAFETY`, `NEEDS_CONFIRMATION`
- Integration with OpenClaw's existing `tool-policy.ts` pattern for tool allow/deny

### Out of Scope
- Confirmation UI/flow (Phase 2)
- Actual execution (Phase 3)
- Adapters (Phase 3)
- Multi-user beyond OpenClaw's existing multi-agent setup

### Key Components
| Component | Description |
|-----------|-------------|
| `extensions/candy/src/permissions.ts` | Permission evaluation engine |
| `extensions/candy/src/permissions-store.ts` | Permission persistence (SQLite) |
| `extensions/candy/src/safety.ts` | Safety rule checker |
| `extensions/candy/src/safety-rules.ts` | Default safety rules + blocklist |
| `extensions/candy/src/pipeline.ts` | Action validation pipeline |
| `src/commands/candy.ts` | CLI commands for Candy management |

### CLI Commands Introduced
```bash
openclaw candy permissions list
openclaw candy permissions set <user> <scope> <allow|deny>
openclaw candy permissions get <user>
openclaw candy safety rules
openclaw candy safety set-quiet-hours <start> <end>
openclaw candy audit search <query>
```

### Safety & Permission Rules
- **Default deny**. If no explicit permission exists, action is blocked.
- **Blocklist is config-driven** (`candy.safety.blocklist` in `openclaw.json`). Examples:
  - `locks.*.unlock_all` — never auto-unlock all locks
  - `security.*.disable` — never disable security system
- **Rate limit**: 10 device commands/minute default, configurable per user
- **Conflict detection**: contradictory actions on same target within 5 minutes are flagged
- **All permission decisions are audited** with reasoning

### Expected Output
```
User: "Turn off all lights and unlock the front door"

Pipeline:
  1. Classify: device_control (lights + locks)
  2. Permission check:
     - lights.all.turn_off -> ALLOWED (user has lights:control)
     - locks.front_door.unlock -> DENIED_PERMISSION (user lacks locks:control)
  3. Safety check:
     - lights action: PASS
     - locks action: BLOCKED by permission (never reaches safety)
  4. Verdict:
     - lights.all.turn_off: NEEDS_CONFIRMATION
     - locks.front_door.unlock: DENIED_PERMISSION

Agent responds: "I can turn off all lights (pending your confirmation),
but you don't have permission to unlock the front door.
Contact your admin to get lock control access."

Audit log: full decision chain recorded
```

### Risks & Failure Modes
- **Permission model too granular too early**. Start with simple scopes (`lights:control`, `locks:control`). ABAC/RBAC comes later with multi-tenancy.
- **Safety rules too aggressive**. Users will get frustrated if safe actions are blocked. Start conservative, tune based on feedback.
- **Race with OpenClaw's tool policy**. Candy's permissions are separate from OpenClaw's `tool-policy.ts`. Make sure they compose correctly (both must pass).

---

## Phase 2: Confirmation Flow + Execution Planner

### Goal
Build the confirmation system and execution planner. When a device action passes permissions and safety, the user must explicitly confirm before it executes. The planner breaks complex requests into ordered steps.

### In Scope
- **Confirmation manager**
  - Confirmation tokens: unique, single-use, time-limited (60s default)
  - Confirmation context: show user exactly what will happen
  - Channel-native confirmation:
    - WhatsApp: inline buttons or reply "YES"
    - Telegram: inline keyboard buttons
    - Slack: interactive message blocks
    - Discord: button components
    - WebChat: confirm dialog
    - CLI: y/n prompt
  - Batch confirmation for multi-step plans
- **Execution planner**
  - Breaks complex intents into ordered `CandyStep[]`
  - Dependency graph: step B depends on step A
  - Rollback plan: if step 3 fails, undo steps 1 and 2
  - Dry-run mode: show plan without executing
  - Step states: `pending`, `confirmed`, `executing`, `completed`, `failed`, `rolled_back`
- **Plan persistence**: active plans stored in SQLite, expire after 5 minutes
- Integration with OpenClaw's channel reply/reaction system for confirmation UX

### Out of Scope
- Actual adapter execution (Phase 3)
- Conditional/event-based triggers
- Automations
- Plan scheduling (cron-based device actions)

### Key Components
| Component | Description |
|-----------|-------------|
| `extensions/candy/src/confirmation.ts` | Confirmation token manager |
| `extensions/candy/src/confirmation-channels.ts` | Per-channel confirmation UX adapters |
| `extensions/candy/src/planner.ts` | Execution plan builder |
| `extensions/candy/src/plan-store.ts` | Plan persistence |
| `extensions/candy/src/plan-types.ts` | CandyPlan, CandyStep, StepState |
| `skills/candy/SKILL.md` | Updated: teach agent about confirmation flow |

### APIs Introduced
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/candy/confirm/{token}` | Confirm a pending action |
| POST | `/api/candy/deny/{token}` | Deny a pending action |
| GET | `/api/candy/plans/pending` | List pending plans |
| GET | `/api/candy/plans/{id}` | Get plan details |
| POST | `/api/candy/plans/dry-run` | Preview execution plan |

### Safety & Permission Rules
- **All physical-world actions require confirmation. No exceptions.** Even admin users confirm.
- **Confirmation tokens are single-use** and expire in 60 seconds
- **Confirmation must be unambiguous**: "YES" or button tap, not "sure" or "ok"
- **Expired confirmations are audited** — shows user abandoned or was too slow
- **Batch confirmations show all steps** — user sees the full plan before approving
- **Rollback plans are best-effort** — not all actions can be undone (logged as such)

### Expected Output
WhatsApp flow:
```
User: "Set the living room to 72 degrees and turn off the bedroom lights"

Agent: "Here's what I'll do:
  1. Set living room thermostat to 72°F
  2. Turn off bedroom lights

Reply YES to confirm, or NO to cancel. (Expires in 60s)"

User: "YES"

Agent: "Confirmed. Executing..."
  [Step 1: thermostat -> awaiting adapter (Phase 3)]
  [Step 2: lights -> awaiting adapter (Phase 3)]

Agent: "Plan approved but no device adapters are connected yet.
Set up Home Assistant: openclaw candy adapters add homeassistant"
```

### Risks & Failure Modes
- **Confirmation fatigue**. Users will hate confirming every light switch. Plan for "trusted action" tiers in Phase 5+ (never for safety-critical actions like locks/security).
- **Channel-specific UX fragmentation**. Each channel handles confirmations differently. Test thoroughly on WhatsApp and Telegram first, others can follow.
- **Token replay attacks**. Tokens must be cryptographically random, bound to user + plan, single-use. Use `crypto.randomUUID()`.
- **Plan state corruption**. If gateway restarts mid-execution, plans in `executing` state need cleanup. Add a startup recovery routine.

---

## Phase 3: Adapter System + First Integrations

### Goal
Build the adapter framework and ship the first real integrations: Home Assistant and a generic HTTP API adapter. After this phase, Candy can actually control devices.

### In Scope
- **Adapter interface** (TypeScript interface all adapters implement)
  ```typescript
  interface CandyAdapter {
    id: string
    name: string
    capabilities(): CandyCapability[]
    validate(action: CandyAction): Promise<ValidationResult>
    execute(action: CandyAction): Promise<ExecutionResult>
    rollback(action: CandyAction): Promise<RollbackResult>
    status(): Promise<AdapterStatus>
    disconnect(): Promise<void>
  }
  ```
- **Adapter registry**: register, unregister, discover, health check
- **Home Assistant adapter**
  - Connect via HA REST API + WebSocket API
  - Device discovery: lights, switches, locks, sensors, climate, covers, fans
  - Action mapping: Candy actions -> HA service calls
  - State polling + event subscription (WebSocket)
  - Auto-discovery of entities when HA URL + long-lived token provided
  - Reconnect with exponential backoff
- **Generic HTTP API adapter**
  - Configure arbitrary REST endpoints
  - Request templates with parameter substitution (`{{device_id}}`, `{{action}}`)
  - Response parsing (JSONPath or jq-style)
  - Auth: API key, Bearer token, Basic auth
  - URL whitelisting (no open proxy)
- **Adapter management CLI**
  ```bash
  openclaw candy adapters add homeassistant --url <url> --token <token>
  openclaw candy adapters add http --name <name> --config <json>
  openclaw candy adapters list
  openclaw candy adapters remove <id>
  openclaw candy adapters test <id>
  openclaw candy adapters capabilities <id>
  ```
- **Execution engine**: connects confirmed plans to adapter calls
- Full end-to-end flow: WhatsApp message -> intent -> orchestrate -> confirm -> execute via adapter -> respond

### Out of Scope
- PC Agent adapter (Phase 4)
- IoT direct protocols (Zigbee, Z-Wave, Matter)
- Adapter marketplace
- Building the Home Assistant instance itself
- Custom adapter SDK for third parties

### Key Components
| Component | Description |
|-----------|-------------|
| `extensions/candy/src/adapters/base.ts` | Adapter interface + base class |
| `extensions/candy/src/adapters/registry.ts` | Adapter registry |
| `extensions/candy/src/adapters/home-assistant/` | HA adapter |
| `extensions/candy/src/adapters/home-assistant/client.ts` | HA REST + WS client |
| `extensions/candy/src/adapters/home-assistant/discovery.ts` | Entity discovery |
| `extensions/candy/src/adapters/home-assistant/mapper.ts` | Action -> HA service mapping |
| `extensions/candy/src/adapters/http-api/` | Generic HTTP adapter |
| `extensions/candy/src/adapters/http-api/template.ts` | Request templating |
| `extensions/candy/src/executor.ts` | Plan executor — adapter caller |
| `src/commands/candy-adapters.ts` | CLI commands for adapter management |

### APIs Introduced
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/candy/adapters` | List registered adapters |
| POST | `/api/candy/adapters` | Register adapter |
| DELETE | `/api/candy/adapters/{id}` | Remove adapter |
| GET | `/api/candy/adapters/{id}/capabilities` | Adapter capabilities |
| GET | `/api/candy/adapters/{id}/status` | Adapter health |
| POST | `/api/candy/adapters/{id}/test` | Test adapter connection |
| GET | `/api/candy/devices` | List all discoverable devices across adapters |

### Safety & Permission Rules
- **Adapters must declare capabilities explicitly**. Candy never guesses what an adapter can do.
- **No adapter = no action**. Plans referencing devices without adapters fail with a clear message.
- **All adapter calls are logged**: request payload, response, latency, success/failure.
- **Adapter connection requires user confirmation** via the existing channel confirmation flow.
- **Timeout enforcement**: every adapter call has a hard 10s timeout (configurable per adapter).
- **HTTP adapter URL whitelist**: no arbitrary URLs. Must be explicitly configured.
- **HA token stored encrypted** in `~/.openclaw/candy/adapters.enc` (using existing credential patterns).
- **Adapter health checks run on a 60s interval**. Unhealthy adapters are flagged, not auto-removed.

### Expected Output
Full end-to-end via WhatsApp:
```
User: "Turn off the kitchen lights"

Agent (Candy skill):
  "I'll turn off the kitchen lights.
   Reply YES to confirm."

User: "YES"

Agent: "Done — kitchen lights are off."

Under the hood:
  1. Pi agent + Candy skill classifies as device_control
  2. Permission engine: ALLOWED
  3. Safety checker: PASS
  4. Planner: 1 step — lights.kitchen.turn_off
  5. Confirmation: issued + approved
  6. Executor: finds HA adapter -> POST /api/services/light/turn_off {"entity_id": "light.kitchen"}
  7. HA returns 200
  8. Audit log: full trace recorded
  9. Reply sent to WhatsApp
```

### Risks & Failure Modes
- **HA API versioning**. Pin to HA's current REST/WS API version. Abstract HA-specific details behind the adapter interface.
- **HA token rotation**. Long-lived tokens are convenient but risky. Document rotation procedure.
- **Network unreliability**. HA is on LAN. Wi-Fi drops. Implement retry with backoff (max 3), then fail and notify.
- **State desync**. Candy thinks light is off, HA says on. Poll after every action + periodic state refresh.
- **Entity name mapping**. HA entity IDs are ugly (`light.kitchen_ceiling_1`). The agent needs to map natural language ("kitchen lights") to HA entities. Use HA's friendly names + area names for matching.

---

## Phase 4: PC Agent Adapter + Node Integration

### Goal
Add the PC Agent adapter and explore integration with OpenClaw's existing node system (`node.invoke`). OpenClaw already has a macOS node that runs `system.run`, `system.notify`, camera, screen recording. Candy extends this with controlled, permission-gated device actions on the computer itself.

### In Scope
- **PC Agent adapter** — connects to OpenClaw's existing node system
  - Leverage `node.invoke` for macOS/iOS/Android nodes already in OpenClaw
  - Action whitelist: launch app, take screenshot, get system info, file operations
  - No arbitrary command execution — whitelist only
  - Maps Candy actions to `system.run` calls with pre-approved commands
- **Candy node integration**
  - Extend OpenClaw's `node.list` / `node.describe` to include Candy capabilities
  - Permission layer between Candy orchestrator and node.invoke
  - Candy actions on nodes go through same permission -> safety -> confirm flow
- **Smart Home Hub adapter** (if applicable)
  - Matter/Thread via Home Assistant (HA acts as the hub)
  - Philips Hue via `openhue` skill integration (already exists in `skills/openhue/`)
- **Adapter status dashboard** (WebChat-based)
  - List connected adapters and their health
  - Recent action history
  - Device state overview

### Out of Scope
- Building new native apps (macOS/iOS/Android apps exist)
- Direct Bluetooth/Zigbee/Z-Wave control (always via HA or a hub)
- Voice-triggered device control (OpenClaw's voice already works — just route through Candy)
- Automation engine (Phase 6)

### Key Components
| Component | Description |
|-----------|-------------|
| `extensions/candy/src/adapters/pc-agent/` | PC Agent adapter |
| `extensions/candy/src/adapters/pc-agent/whitelist.ts` | Command whitelist |
| `extensions/candy/src/adapters/pc-agent/node-bridge.ts` | Bridge to OpenClaw node.invoke |
| `extensions/candy/src/adapters/openhue-bridge.ts` | Bridge to existing openhue skill |
| `extensions/candy/src/dashboard/` | WebChat-based adapter dashboard |
| `skills/candy/SKILL.md` | Updated: teach agent about available adapters |

### Safety & Permission Rules
- **PC Agent whitelist is mandatory**. Only pre-approved commands can run.
- **Elevated actions require separate permission scope** (`pc:elevated`) — tied to OpenClaw's `/elevated on|off` toggle
- **Node actions go through full Candy pipeline** — no shortcuts via direct `node.invoke`
- **Screenshot/camera actions require explicit `pc:screen` permission**
- **File operations scoped to approved directories** — no access to `~/`, system dirs, etc.

### Expected Output
```
User (via Telegram): "Open Spotify and play my Discover Weekly"

Agent (Candy skill):
  "I'll open Spotify on your Mac. Reply YES to confirm."

User: "YES"

Agent: "Spotify opened."
  (Under the hood: Candy -> PC Agent adapter -> node.invoke -> system.run 'open -a Spotify')
```

### Risks & Failure Modes
- **Node availability**. macOS node must be running and connected. If offline, action fails with clear message.
- **Whitelist maintenance burden**. Keep it small and pragmatic. 10-20 approved commands, not 200.
- **Elevated access creep**. Users will want more power. Resist. Every whitelist addition is a security decision.

---

## Phase 5: Multi-Tenant SaaS Layer

### Goal
Transform Candy from a single-user OpenClaw extension into a multi-tenant SaaS. This is the biggest architectural shift — OpenClaw is designed for personal, single-user use. Candy SaaS requires tenant isolation, billing, and shared infrastructure.

### In Scope
- **Tenant model**
  - Tenant: org name, plan tier, members, created_at
  - Each tenant gets isolated: adapters, permissions, audit logs, plans
  - Tenant admin vs. member roles
- **Hosted Gateway option**
  - Candy-managed OpenClaw gateway instances per tenant (or shared with isolation)
  - Option: self-hosted gateway that connects to Candy SaaS for permissions/billing
- **Web dashboard** (standalone React app, not just WebChat)
  - Onboarding wizard
  - Adapter management
  - Permission management
  - Audit log viewer
  - Usage analytics
  - Team management
- **Billing** (Stripe)
  - Free: 1 user, 2 adapters, 100 commands/month
  - Pro: 5 users, 10 adapters, 5,000 commands/month
  - Business: 25 users, unlimited adapters, 50,000 commands/month
- **API layer**
  - REST API for dashboard
  - Auth: JWT (or OAuth via OpenClaw's existing auth profiles)
  - Usage tracking and metering

### Out of Scope
- Self-hosted installation tooling (docs only)
- Custom domains per tenant
- White-labeling
- SOC 2 / HIPAA compliance
- Data export/portability tools

### Key Components
| Component | Description |
|-----------|-------------|
| `candy-saas/` | New standalone SaaS service |
| `candy-saas/api/` | REST API (Hono or Express — match OpenClaw patterns) |
| `candy-saas/api/tenants.ts` | Tenant CRUD |
| `candy-saas/api/billing.ts` | Stripe integration |
| `candy-saas/api/auth.ts` | Auth (JWT or piggyback on OpenClaw auth) |
| `candy-saas/dashboard/` | React web dashboard |
| `candy-saas/db/` | Database layer (PostgreSQL for multi-tenant, or MongoDB) |
| `extensions/candy/src/saas-bridge.ts` | Bridge between self-hosted Candy extension and SaaS API |

### Safety & Permission Rules
- **Tenant isolation is non-negotiable**. Every query scoped by tenant_id. Zero exceptions.
- **Billing mutations require tenant admin role**
- **Usage limits enforced at orchestration layer** (reject commands when over limit)
- **Stripe webhook signature verification** on all billing callbacks
- **Self-hosted mode**: Candy extension works without SaaS. SaaS adds management + billing.

### Expected Output
New user signs up at `candy.ai`:
1. Creates tenant
2. Picks a plan
3. Guided adapter setup: enters Home Assistant URL + token
4. Sends first command via WhatsApp (linked during onboarding)
5. "Turn off the lights" -> confirmed -> executed -> billed

### Risks & Failure Modes
- **OpenClaw is personal-first**. Multi-tenancy is a fundamental shift. Consider running per-tenant gateway instances vs. a shared gateway with isolation.
- **Billing edge cases**. Failed payments, plan changes, overages. Lean on Stripe.
- **Onboarding friction**. HA setup requires technical knowledge. Make the wizard as simple as possible.
- **Cost of gateway instances**. If each tenant runs their own gateway, hosting costs scale linearly. Shared gateway + isolation is more efficient but harder to build.

---

## Phase 6: Automations + Event-Driven Actions

### Goal
Let users create automations: "When X happens, do Y." Builds on OpenClaw's existing `cron` system and extends it with device event triggers and conditional logic.

### In Scope
- **Automation engine**
  - Trigger types:
    - Time-based (cron) — already exists in OpenClaw, extend for Candy actions
    - Device event: "when motion sensor activates", "when door opens"
    - State change: "when temperature drops below 65"
    - Channel event: "when I say goodnight"
  - Actions: any Candy device action (through full orchestration pipeline)
  - Conditions: time of day, device state, user presence
  - Automation CRUD via CLI and dashboard
- **Automation builder** (dashboard UI)
  - Visual trigger + condition + action builder
  - Test/dry-run before activating
  - Enable/disable toggle
  - Execution history per automation
- **Safety for automations**
  - All automations pass through permission + safety pipeline on every trigger
  - Rate limits per automation (prevent infinite loops)
  - Admin approval for automations involving locks/security
  - Automation versioning and rollback

### Out of Scope
- Complex multi-step conditional logic (if/else trees)
- Machine learning-based automation suggestions
- Cross-tenant automations
- Scene/routine marketplace

### Key Components
| Component | Description |
|-----------|-------------|
| `extensions/candy/src/automations/engine.ts` | Automation trigger evaluation |
| `extensions/candy/src/automations/builder.ts` | Automation creation + validation |
| `extensions/candy/src/automations/triggers.ts` | Trigger type handlers |
| `extensions/candy/src/automations/store.ts` | Automation persistence |
| Integration with `src/cron/` | Time-based triggers via OpenClaw cron |

### Safety & Permission Rules
- **Automations pass through full safety pipeline** on every trigger, not just at creation
- **Rate limit**: max 100 triggers/hour per automation, 1000/hour per tenant
- **Loop detection**: contradictory automations flagged and disabled
- **Lock/security automations require admin approval**
- **Automation history is audited** — every trigger, every execution, every skip

### Expected Output
```
User creates automation via dashboard:
  Trigger: "Motion sensor detects no motion for 30 minutes"
  Condition: "After 10pm"
  Action: "Turn off all lights"

At 10:45pm, motion sensor goes quiet for 30 min:
  1. HA adapter reports motion sensor state change
  2. Automation engine evaluates trigger: MATCH
  3. Condition check: time is 11:15pm, after 10pm: MATCH
  4. Action: lights.all.turn_off
  5. Permission check: ALLOWED (automation owner has lights:control)
  6. Safety check: PASS
  7. NO confirmation needed (pre-approved automation, non-safety-critical)
  8. Executor: HA -> lights off
  9. Audit: automation #12 fired, lights off
  10. User notified (optional): "Auto: lights turned off (no motion for 30min)"
```

### Risks & Failure Modes
- **Automation storms**. One trigger cascading into many actions. Rate limit hard.
- **HA event flood**. Some sensors fire constantly. Debounce triggers.
- **Confirmation fatigue for automations**. The whole point of automation is hands-off. Allow pre-approval for non-critical actions, always confirm for safety-critical.

---

## Phase 7: Production Hardening + Scale

### Goal
Make Candy production-ready for paying customers. Security, observability, reliability, compliance basics.

### In Scope
- **Security hardening**
  - Dependency scanning (npm audit, Snyk)
  - Adapter credential encryption at rest
  - API input validation audit
  - OWASP Top 10 review
  - Rate limiting on all public endpoints
  - CSP headers on dashboard
- **Observability**
  - Structured logging (OpenClaw already uses tslog — extend for Candy)
  - Metrics: command latency, adapter response times, error rates, usage per tenant
  - Alerting: adapter down, error spike, billing failure
  - Command pipeline tracing (intent -> orchestrate -> execute)
- **Reliability**
  - Adapter reconnection with exponential backoff
  - Plan recovery on gateway restart
  - Graceful degradation: if one adapter fails, others still work
  - Database backup (SQLite for self-hosted, managed DB for SaaS)
- **Performance**
  - Adapter call parallelization (independent steps in a plan run in parallel)
  - SQLite query optimization + indexing
  - Dashboard bundle optimization
- **Compliance basics**
  - Data retention policies per tenant
  - Data deletion capability (GDPR)
  - Audit log export
  - Privacy policy, ToS

### Out of Scope
- SOC 2 Type II (requires 6+ month audit)
- HIPAA, PCI-DSS
- On-premise deployment tooling
- Geographic data residency

### Expected Output
Candy runs reliably for paying customers:
- Handles 100+ concurrent users
- Recovers from adapter failures gracefully
- Full observability into system behavior
- Passes basic security review
- GDPR-compliant data handling

---

## Phase 8: Platform Maturity + Growth

### Goal
Features that turn Candy from a working product into a growing platform.

### In Scope
- **Developer API**: Public API with API keys for third-party integrations
- **Custom adapter SDK**: TypeScript package for building custom adapters
- **Multi-language NLP**: Extend beyond English (Spanish, Portuguese, French, German)
- **Custom command aliases**: "bedtime routine" = turn off lights + lock doors + set thermostat
- **Usage analytics dashboard**: most used commands, peak hours, cost projections
- **Webhook support**: send Candy events to external URLs
- **Voice-first optimizations**: leverage OpenClaw's Voice Wake for hands-free device control

### Out of Scope
- Public adapter marketplace
- White-labeling
- Native mobile apps (OpenClaw's existing apps suffice)
- Competitive features against full home automation platforms (Candy is the AI control layer, not the replacement for Home Assistant)

---

## Summary Table

| Phase | Name | Core Deliverable | Builds On |
|-------|------|-----------------|-----------|
| 0 | Skill Skeleton + Audit | Candy skill, audit log, intent classification | OpenClaw skills system |
| 1 | Permissions + Safety | Permission engine, safety checker, validation pipeline | Phase 0 + OpenClaw tool policy |
| 2 | Confirmation + Planner | Confirmation flow (all channels), execution planner | Phase 1 + OpenClaw channel system |
| 3 | Adapters | Home Assistant + HTTP adapters, full E2E flow | Phase 2 + OpenClaw gateway |
| 4 | PC Agent + Nodes | PC control, node integration, dashboard | Phase 3 + OpenClaw node system |
| 5 | Multi-Tenant SaaS | Tenant isolation, billing, web dashboard | Phase 4 + new SaaS service |
| 6 | Automations | Event-driven automation engine | Phase 5 + OpenClaw cron |
| 7 | Production Hardening | Security, observability, reliability | All previous |
| 8 | Platform Maturity | Developer API, SDK, analytics | All previous |

---

## Guiding Principles

1. **Extend, don't fork.** Candy is an OpenClaw extension, not a replacement. Use OpenClaw's patterns, tools, and infrastructure.
2. **Skills-first architecture.** The Pi agent + Candy skill handles natural language. Candy code handles orchestration. Clean separation.
3. **Human in the loop.** Physical-world actions always require confirmation. Even automated ones pass through safety.
4. **Adapter-only execution.** Candy never directly controls anything. No adapter = no action.
5. **Log everything.** Every device action, every permission decision, every confirmation — audited.
6. **Fail safely.** A missed light switch is better than an unlocked door.
7. **Leverage what exists.** OpenClaw has channels, voice, browser, canvas, nodes, cron. Don't rebuild them.

---

*This roadmap is built on the actual OpenClaw codebase (v2026.2.4). Each phase will be refined as learnings from previous phases are incorporated. The key advantage: most of the hard infrastructure (messaging, channels, agent runtime, security) already exists.*
