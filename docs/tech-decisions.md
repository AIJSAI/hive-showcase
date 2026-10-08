# Technical Decisions: Hive

This document contains excerpts from the project's Architecture Decision Records (ADRs).

---

## ADR-001: OpenClaw-Native Architecture

**Status**: Accepted  
**Context**: The first research documents proposed a custom Docker Compose stack (a Traefik reverse proxy, Qdrant, Redis and custom Python orchestration) for the multi-agent system. That meant building model routing, session management, conversation memory and scheduled automation from scratch. The system needs per-agent isolation, scheduled automation, persistent semantic memory, channel integrations and browser automation, with minimal custom code to maintain.

**Decision**: Install OpenClaw natively and build on its capabilities instead of the custom stack. OpenClaw runs as a systemd user service on the host; Docker is used only for agent sandboxes and the LiteLLM and Redis containers. OpenClaw provides:
- Multi-agent gateway with depth-2 nesting (orchestrator → team leads → workers)
- Per-agent Docker sandboxing with configurable capabilities
- Session management with DM pairing and channel-per-domain routing
- Hook system for session memory, command logging and boot triggers
- Cron scheduler for automated workflows
- QMD memory backend with hybrid search

**Consequences**:
- Deployment measured in days, not weeks: configuration instead of development.
- Single-node deployment with no orchestration overhead (Kubernetes, etc.).
- Dependency on the OpenClaw project for fixes and features, so version pinning is required.
- Configuration has to follow OpenClaw's own patterns and conventions.

---

## ADR-003: 1Password Hybrid Secrets Management

**Status**: Accepted  
**Context**: The system needs runtime secrets injection (API keys for Gemini, Anthropic, ElevenLabs, etc.) without storing plaintext on persistent disk. 1Password Individual plan doesn't support service accounts or Connect Server, the standard enterprise patterns for automated credential access.

**Decision**: Hybrid model combining two mechanisms:
1. **systemd EnvironmentFile**: After the manual 1Password sign-in that follows each reboot, `op run` populates `/run/openclaw-credentials/.env` on tmpfs (RAM-backed filesystem). The OpenClaw gateway unit loads this file via `EnvironmentFile=`.
2. **Config substitution**: `openclaw.json` uses `${ENV_VAR}` syntax for fields that don't support native SecretRef. Fields that support SecretRef use OpenClaw's `secrets.providers` with `exec` type (calls `op read`).

**Consequences**:
- The design rule is zero plaintext secrets on persistent disk, since `/run/` is tmpfs (cleared on reboot); after early incidents broke it, ADR-020's credential sweep enforces it on every runtime change.
- Two credential paths (env substitution + SecretRef) add complexity but cover all config fields.
- Without service accounts, the 1Password CLI needs a manual sign-in after each reboot, the one manual step in an otherwise unattended boot.
- `openclaw doctor` may report "unresolved SecretRef" when run outside systemd context, a false positive (secrets resolve at runtime).

---

## ADR-012: Docker Privilege Model

**Status**: Accepted  
**Context**: Docker sandboxing with `--cap-drop=ALL` removes all Linux capabilities, including `DAC_OVERRIDE` (bypassing file permissions). Agent processes running as PID 1 inside containers cannot write to bind-mounted workspace directories even when running as root in the container, because the host filesystem enforces POSIX permissions and the container's root lacks `DAC_OVERRIDE`.

**Decision**:
- Apply `--cap-drop=ALL` and `--security-opt=no-new-privileges` to all agent containers.
- Set `chmod 777` on workspace directories before container launch.
- Accept that filesystem permissions are not the security boundary; the **container itself** is the boundary (no network, dropped capabilities, no privilege escalation).
- Cross-agent data isolation is enforced by separate bind mounts (`scope: "agent"`), not POSIX permissions within any single container.

**Consequences**:
- Every Linux capability stays dropped (`--cap-drop=ALL`).
- `chmod 777` is an accepted trade-off: the workspace is agent-scoped and container-isolated, though any process on the host can write to it.
- If a container is compromised, the attacker has no network, no capabilities, and no ability to escalate privileges; short of a container escape, file read/write within the workspace is the blast radius.

---

## ADR-014: Modular Domain Team Architecture

**Status**: Accepted  
**Context**: The initial plan defined 4 fixed agents (personal, ops, shared and researcher). As requirements grew, this rigid structure couldn't accommodate new domains (wine and product development, among others) without architectural changes.

**Decision**: Shift to modular domain teams with depth-2 nesting:
- **Orchestrator (main)**: Routes tasks to appropriate team leads, manages system config.
- **Team Leads** (depth-1): Domain specialists that understand their domain context; `research-lead` was the pilot team.
- **Workers** (depth-2): Spawned by leads for specific subtasks, inherit parent's sandbox.
- Teams added incrementally: a new lead plus Discord channel plus tool policy is all that's needed.

**Consequences**:
- New domains require config additions, not architectural changes.
- Workers inherit parent sandbox policies, so each new team gets the same isolation without extra configuration.
- Orchestrator complexity increases with team count, mitigated by clear delegation patterns and tool policy isolation.

---

## ADR-016: Adaptive Self-Improvement via Prompting & Memory

**Status**: Accepted  
**Context**: Static agent configurations require manual tuning as usage patterns evolve. The orchestrator should be able to identify recurring inefficiencies and adjust its own behavior within safe boundaries.

**Decision**: Implement tiered self-improvement:
- **Autonomous** (low-risk): Worker model swaps within tier, tool enable/disable within policy.
- **Ask first** (high-risk): New agent creation, security policy changes, budget cap adjustments.
- **Denied**: Non-main agents cannot modify system configuration.

Mechanisms:
- `MEMORY.md` per agent for structured reflection and learning.
- Weekly cron job (Sunday, on a cost-optimized model) reviews orchestrator memory, recent team spawns, and lead insights.
- Outputs weekly review to `memory/reviews/YYYY-WXX.md` and delivers 3-5 bullet summary to Discord.
- Cross-agent knowledge sharing: orchestrator reads `## Shareable Insights` from domain lead memory files.

**Consequences**:
- Agents adjust their own behavior within set limits, with owner approval required for high-impact changes.
- Weekly review provides observability into agent behavior patterns.
- Cross-agent knowledge sharing is designed to keep one team's lessons from staying with that team.

---

## ADR-020: Runtime Change Protocol

**Status**: Accepted  
**Context**: After the first phases, changes to the live server (config edits, workspace deployments, credential rotations, sandbox adjustments) kept causing incidents. More than 10 documented incidents traced to one root cause: runtime changes applied without pre-flight analysis, a config snapshot, a blast-radius check or structured verification afterward. Examples included wrong model prefixes that broke routing, a sandbox network setting that broke per-agent isolation, a plugin enabled without being allowed, and plaintext API keys left in config files.

**Decision**: Adopt a Runtime Change Protocol: a three-phase gate (Plan → Apply → Verify) inside the existing implementation checkpoints, required whenever a change touches the live system.
- **Plan**: read the official OpenClaw configuration and security docs for the keys being changed, check constraints, and assess blast radius.
- **Apply**: take a config snapshot first, then make the change.
- **Verify**: check health, security, credentials and function after the change, including a sweep of the eight credential locations that OpenClaw's security docs list.

**Consequences**:
- Pre-flight analysis catches constraint violations before they reach the live system.
- Config snapshots enable instant rollback when verification fails.
- The credential sweep keeps plaintext secrets from being deployed.
- More process for every runtime change, mitigated because most steps are quick CLI commands.

---

## ADR-024: Cost Optimization After Google Cloud Credits

**Status**: Accepted
**Supersedes**: Sections of ADR-004 (LiteLLM cost control) and ADR-015 (API keys over subscription).

**Context**: The free Google Cloud credit program that covered Gemini API usage during Phases 3A-10 was exhausted in early April 2026. Every Gemini token now bills directly. The system was built under "use the best model because credits are free" assumptions, and those assumptions no longer hold. The unoptimized projection without credits ran well over the LiteLLM hard cap, which is unacceptable. The goal is **best value for spend**, not cheapest possible. Per-agent model tiering holds steady-state spend under that hard ceiling.

**Decision**: Retain high-capability models for the leads whose output quality depends on them; move the orchestrator to the Flash tier ops already used; replace Sonnet with Haiku 4.5 as the Anthropic fallback for every agent except the research lead.

| Agent | Primary | Google fallback | OpenAI fallback | Anthropic fallback |
|---|---|---|---|---|
| `main` (Queenie) | `gemini-3-flash-preview` | `gemini-2.5-flash` | `gpt-4.1-mini` | `claude-haiku-4-5` |
| `ops` | `gemini-3-flash-preview` | `gemini-2.5-flash` | `gpt-4.1-mini` | `claude-haiku-4-5` |
| `research-lead` | `gemini-3.1-pro-preview-customtools` | `gemini-2.5-pro` | `gpt-4.1` | `claude-sonnet-4-6` |

Infrastructure fixes alongside the model changes:
- `openclaw.json` cost metadata corrected (Gemini 2.5 models were previously marked $0 input/output, a relic of the credits era, so LiteLLM budget tracking was undercounting spend).
- Vertex AI routing order flipped to prefer the Gemini Dev API (which has a free tier) over Vertex (which doesn't).
- Claude Haiku 4.5 registered as a new fallback target with `max_budget: 5.0/mo`.
- All cross-group fallback chains now terminate in `claude-haiku-4-5` instead of `claude-sonnet-4-6`, with `claude-sonnet-4-6: [claude-haiku-4-5]` added as a new chain for Sonnet outage safety.

**Consequences**:
- Orchestrator and ops costs drop substantially without quality risk (their work is classification + dispatch).
- The research lead keeps Pro CustomTools because its prompts were tuned to that model's behavior.
- Fallback chains are a much smaller spike risk during provider outages; Haiku 4.5 costs about a third as much as Sonnet 4.6.
- research-lead keeps Sonnet as its Anthropic fallback (which itself fails over to Haiku) because factual accuracy on research documents still matters when infrastructure is degraded.

---

## ADR-025: Active Memory Plugin Adoption

**Status**: Accepted
**Context**: The work-repo auto-memory is at ~20 files today and projected to land in the 500-1000 range over twelve months as daily briefings and Discord DMs accumulate. Agent workspace memory already spans the orchestrator's and each domain lead's `memory/` folder. QMD has a hybrid vector+text+MMR index with temporal decay. Despite all of this, recall misses still happen: agents answer without relevant prior context, or the operator has to re-supply background that's already in memory. The summary-based context-loading pattern degrades past ~50 files and becomes untenable past a few hundred.

The previous working answer was the parked "Path A" exploration: stand up a local llama.cpp + embedding server on the host and build a client-side RAG layer. OpenClaw 2026.4.12 (released 2026-04-12) added the Active Memory plugin, which solves the same problem inside the gateway without a separate embedding server.

**Decision**: Enable the Active Memory plugin for `main` and the domain leads. Skip `ops`.

Configuration:
- `model: gemini-3-flash-preview` (lowest tier trusted for structured memory-selection work; matches ADR-024's cost-optimization principle).
- `queryMode: recent` (latest user turn plus a small tail, the documented default).
- `promptStyle: balanced`, `timeoutMs: 15000`, `maxSummaryChars: 220`.
- `persistTranscripts: true` for the first review cycle so memory-selection quality can be audited, then flip back to `false` to save disk.

Per-agent rationale:
- **`research-lead`**: deep-reasoning work and a long memory of investigations. Highest-value recall surface.
- **`main` (Queenie)**: orchestrator that also handles Discord DMs directly. User-preference recall matters. Watch DM latency.
- **`ops`**: scheduled health checks and email triage; mostly stateless. No benefit.

**Consequences**:
- Retrieval over the existing QMD memory runs inside the gateway, with no separate embedding server to build or maintain.
- Targets the recall-miss class of failures without requiring users to say "search memory" first; the first review cycle's transcripts show whether it works.
- Replaces the parked "Path A" local-embedding exploration with an option that needs no new infrastructure.
- Active Memory runs a blocking sub-agent call before each primary reply. The estimate was 1-2 seconds added to each DM reply, with the 15-second timeout as the ceiling, so latency is watched and the setting revisited if replies feel slow.
