# Technical Decisions: Hive

This document contains excerpts from the project's Architecture Decision Records (ADRs).

---

## ADR-001: OpenClaw-Native Architecture

**Status**: Accepted  
**Context**: Multiple agent frameworks were evaluated: LangChain, AutoGen, CrewAI, and OpenClaw. The system requires: multi-agent orchestration with depth-2+ nesting, persistent sessions across restarts, Docker sandboxing per agent, Discord/CLI/webhook bindings, and local semantic memory. Cloud-hosted alternatives (AWS Bedrock Agents, Azure AI Agent Service) were rejected on cost and customization grounds.

**Decision**: Build on OpenClaw as the native agent framework. OpenClaw provides:
- Multi-agent gateway with depth-2 nesting (orchestrator → team leads → workers)
- Per-agent Docker sandboxing with configurable capabilities
- Session management with DM pairing and channel-per-domain routing
- Hook system for session-memory, audit, and boot triggers
- Cron scheduler for automated workflows
- QMD memory backend with hybrid search

**Consequences**:
- Full control over agent configuration, security policies, and tool routing.
- Single-node deployment with no orchestration overhead (Kubernetes, etc.).
- Tightly coupled to OpenClaw's release cycle, so version pinning is required (ADR: gateway version lag).
- Documentation is community-driven, so some trial-and-error is required for edge cases.

---

## ADR-003: 1Password Hybrid Secrets Management

**Status**: Accepted  
**Context**: The system needs runtime secrets injection (API keys for Gemini, Anthropic, ElevenLabs, etc.) without storing plaintext on persistent disk. 1Password Individual plan doesn't support service accounts or Connect Server, the standard enterprise patterns for automated credential access.

**Decision**: Hybrid model combining two mechanisms:
1. **systemd EnvironmentFile**: At boot, a systemd unit runs `op run` to populate `/run/openclaw-credentials/.env` on tmpfs (RAM-backed filesystem). The OpenClaw gateway unit loads this file via `EnvironmentFile=`.
2. **Config substitution**: `openclaw.json` uses `${ENV_VAR}` syntax for fields that don't support native SecretRef. Fields that support SecretRef use OpenClaw's `secrets.providers` with `exec` type (calls `op read`).

**Consequences**:
- Zero plaintext secrets on persistent disk, since `/run/` is tmpfs (cleared on reboot).
- Two credential paths (env substitution + SecretRef) add complexity but cover all config fields.
- 1Password CLI must be authenticated (device trust), a one-time interactive setup per machine.
- `openclaw doctor` may report "unresolved SecretRef" when run outside systemd context, a false negative (secrets resolve at runtime).

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
- Strongest possible capability restriction (`--cap-drop=ALL`) maintained.
- `chmod 777` is a pragmatic trade-off: the workspace is agent-scoped and container-isolated.
- If a container is compromised, the attacker has no network, no capabilities, and no ability to escalate privileges; file read/write within the workspace is the maximum blast radius.

---

## ADR-014: Modular Domain Team Architecture

**Status**: Accepted  
**Context**: The initial plan defined 4 fixed agents (main, ops, research, startup). As requirements grew, this rigid structure couldn't accommodate new domains (job search, email triage, content creation) without architectural changes.

**Decision**: Shift to modular domain teams with depth-2 nesting:
- **Orchestrator (main)**: Routes tasks to appropriate team leads, manages system config.
- **Team Leads** (depth-1): Domain specialists (research-lead, startup-lead, jobs-lead) that understand their domain context.
- **Workers** (depth-2): Spawned by leads for specific subtasks, inherit parent's sandbox.
- Teams added incrementally: a new lead plus Discord channel plus tool policy is all that's needed.

**Consequences**:
- New domains require config additions, not architectural changes.
- Workers inherit parent sandbox policies, so the security model scales automatically.
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
- Weekly cron job (Sunday 10 AM) reviews orchestrator memory, recent team spawns, and lead insights.
- Outputs weekly review to `memory/reviews/YYYY-WXX.md` and delivers 3-5 bullet summary to Discord.
- Cross-agent knowledge sharing: orchestrator reads `## Shareable Insights` from domain lead memory files.

**Consequences**:
- Self-improving behavior within safe guardrails, with owner approval required for high-impact changes.
- Weekly review provides observability into agent behavior patterns.
- Cross-agent knowledge sharing prevents domain silos.

---

## ADR-020: Runtime Change Protocol

**Status**: Accepted  
**Context**: Configuration changes to a live agent system carry risk: a bad config can deadlock all agents (see elevated exec deadlock), break sandbox isolation, or wipe security allowlists (see exec-approvals.json trailing comma bug). Ad-hoc changes via interactive sessions lack audit trails and rollback capability.

**Decision**: Structured runtime change workflow:
1. **Propose**: Agent (or operator) describes the change and rationale.
2. **Verify preconditions**: Run `openclaw doctor`, check `exec-approvals.json` validity, snapshot current config.
3. **Apply**: Make the change (config edit, restart if needed).
4. **Test**: Run `openclaw security audit --deep`, verify agent functionality.
5. **Commit**: Push to hive-runtime repo with descriptive commit message.
6. **Rollback plan**: If verification fails, restore snapshot.

**Consequences**:
- Every config change has an audit trail (git history in hive-runtime repo).
- Verification gates catch common pitfalls (JSON syntax, doctor warnings).
- Rollback is always available via git revert.
- Slightly more ceremony per change, an accepted trade-off for production stability.

---

## ADR-023: Chef Antoine + Kroger Cart Integration

**Status**: Accepted
**Context**: An external "Chef Antoine" chatbot was useful for recipe discovery and meal planning but had no persistence, no memory of past cooks, no inventory awareness, and no path from "recipe" to "ingredients in cart". Recipes were spread across five Google Docs with duplication. Kroger (parent of the operator's local Bakers store) offers free Public APIs including a Cart API that can programmatically add items to a customer's cart, so the integration was readily available.

**Decision**: Add a `chef-lead` domain team to Hive following the ADR-014 expansion protocol.

- **Agent name**: `chef-lead` (domain team lead, no workers initially)
- **Primary model**: `gemini-3.1-pro-preview-customtools` (creative reasoning + personality maintenance is load-bearing for a culinary mentor persona)
- **Fallback chain** (per ADR-024): `gemini-2.5-pro` → `gpt-4.1` → `claude-haiku-4-5`
- **Sandbox**: `mode: "all"`, `network: none` (consistent with all Hive agents)
- **Discord**: Channel-bound to `#cooking`; also responds to DMs
- **Kroger integration**: Pre-fetch pattern (same as email triage and the job search pipeline). Agent writes a request, host cron script calls the Kroger API, results land in workspace, agent reads. This keeps the agent inside its `network: none` sandbox.

**Consequences**:
- New domain team added without touching the existing architecture, proving out ADR-014's "add by config, not by refactor" claim.
- Cart population is automated; checkout remains manual in the Kroger app for safety.
- Recipe archive is searchable via QMD semantic search, with the same temporal-decay and hybrid-search guarantees as the other agents.
- Multimodal inventory (photo-based pantry/fridge intake) is the first multimodal workload on the platform.

---

## ADR-024: Cost Optimization Post-GCP Credits

**Status**: Accepted
**Supersedes**: Sections of ADR-004 (LiteLLM cost control) and ADR-015 (API keys over subscription).

**Context**: The $300 GCP credit program that covered Gemini API usage during Phases 3A-10 was exhausted in early April 2026. Every Gemini token now bills directly. The system was built under "use the best model because credits are free" assumptions, and those assumptions no longer hold. The unoptimized projection without credits ran well over the LiteLLM hard cap, which is unacceptable. The goal is **best value for spend**, not cheapest possible. Per-agent model tiering holds steady-state spend comfortably under that hard ceiling.

**Decision**: Retain high-capability models where creative or factual quality is load-bearing; downgrade the orchestrator and ops to Flash tier; replace Sonnet with Haiku 4.5 as the Anthropic fallback for every agent except the research lead.

| Agent | Primary | Google fallback | OpenAI fallback | Anthropic fallback |
|---|---|---|---|---|
| `main` (Queenie) | `gemini-3-flash-preview` | `gemini-2.5-flash` | `gpt-4.1-mini` | `claude-haiku-4-5` |
| `ops` | `gemini-3-flash-preview` | `gemini-2.5-flash` | `gpt-4.1-mini` | `claude-haiku-4-5` |
| `chef-lead` | `gemini-3.1-pro-preview-customtools` | `gemini-2.5-pro` | `gpt-4.1` | `claude-haiku-4-5` |
| `research-lead` (Q) | `gemini-3.1-pro-preview-customtools` | `gemini-2.5-pro` | `gpt-4.1` | `claude-sonnet-4-6` |

Infrastructure fixes alongside the model changes:
- `openclaw.json` cost metadata corrected (Gemini 2.5 models were previously marked $0 input/output, a relic of the credits era, so LiteLLM budget tracking was undercounting spend).
- Vertex AI routing order flipped to prefer the Gemini Dev API (which has a free tier) over Vertex (which doesn't).
- Claude Haiku 4.5 registered as a new fallback target with `max_budget: 5.0/mo`.
- All cross-group fallback chains now terminate in `claude-haiku-4-5` instead of `claude-sonnet-4-6`, with `claude-sonnet-4-6: [claude-haiku-4-5]` added as a new chain for Sonnet outage safety.

**Consequences**:
- Orchestrator and ops costs drop substantially without quality risk (their work is classification + dispatch).
- Creative agents keep Pro CustomTools because that's where personality and cooking creativity actually live.
- Fallback chains are no longer a spike risk during provider outages; Haiku 4.5 is 5x cheaper than Sonnet 4.6 with similar quality on routine work.
- Q (research-lead) keeps Sonnet as its last-resort fallback because factual accuracy on career documents is load-bearing even when infrastructure is degraded.

---

## ADR-025: Active Memory Plugin Adoption

**Status**: Accepted
**Context**: Hive's memory surface has grown meaningfully. The work-repo auto-memory is at ~20 files today and projected to land in the 500-1000 range over twelve months as daily briefings, market intel, the job search pipeline, chef inventory, and Discord DMs accumulate. Agent workspace memory already spans `workspace-main/memory/`, `workspace-research-lead/memory/`, and `workspace-chef-lead/memory/`. QMD has a hybrid vector+text+MMR index with temporal decay. Despite all of this, recall misses still happen: agents answer without relevant prior context, or the operator has to re-supply background that's already in memory. The summary-based context-loading pattern degrades past ~50 files and becomes untenable past a few hundred.

The previous working answer was the parked "Path A" exploration: stand up a local llama.cpp + embedding server on the host and build a client-side RAG layer. OpenClaw 2026.4.12 (released 2026-04-12) added the Active Memory plugin, which solves the same problem server-side with zero local infrastructure.

**Decision**: Enable the Active Memory plugin for `main`, `research-lead`, and `chef-lead`. Skip `ops`.

Configuration:
- `model: gemini-3-flash-preview` (lowest tier trusted for structured memory-selection work; matches ADR-024's cost-optimization principle).
- `queryMode: recent` (latest user turn plus a small tail, the documented-recommended default).
- `promptStyle: balanced`, `timeoutMs: 15000`, `maxSummaryChars: 220`.
- `persistTranscripts: true` for the first review cycle so memory-selection quality can be audited, then flip back to `false` to save disk.

Per-agent rationale:
- **`research-lead` (Q)**: deep-reasoning work, long memory of investigations, market intel state, job search pipeline history. Highest-value recall surface.
- **`chef-lead` (Chef Antoine)**: inventory from photos, past recipes, stored preferences, Kroger ops context. Memory-heavy interactive agent.
- **`main` (Queenie)**: orchestrator that also handles Discord DMs directly. User-preference recall matters. Watch DM latency.
- **`ops`**: scheduled health checks and email triage; mostly stateless. No benefit.

**Consequences**:
- Server-side RAG over existing QMD memory, with no local embedding infrastructure to build or maintain.
- Closes the recall-miss class of failures without requiring users to say "search memory" first.
- Replaces the parked "Path A" Gemma 4 exploration with a zero-infrastructure alternative.
- Active Memory runs a blocking sub-agent call before each primary reply, so DM latency increases marginally; watch and revisit if it bites.
