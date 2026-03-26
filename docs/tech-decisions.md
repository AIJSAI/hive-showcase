# Technical Decisions — Hive

This document contains excerpts from the project's Architecture Decision Records (ADRs).

---

## ADR-001: OpenClaw-Native Architecture

**Status**: Accepted  
**Context**: Multiple agent frameworks were evaluated — LangChain, AutoGen, CrewAI, and OpenClaw. The system requires: multi-agent orchestration with depth-2+ nesting, persistent sessions across restarts, Docker sandboxing per agent, Discord/CLI/webhook bindings, and local semantic memory. Cloud-hosted alternatives (AWS Bedrock Agents, Azure AI Agent Service) were rejected on cost and customization grounds.

**Decision**: Build on OpenClaw as the native agent framework. OpenClaw provides:
- Multi-agent gateway with depth-2 nesting (orchestrator → team leads → workers)
- Per-agent Docker sandboxing with configurable capabilities
- Session management with DM pairing and channel-per-domain routing
- Hook system for session-memory, audit, and boot triggers
- Cron scheduler for automated workflows
- QMD memory backend with hybrid search

**Consequences**:
- Full control over agent configuration, security policies, and tool routing.
- Single-node deployment — no orchestration overhead (Kubernetes, etc.).
- Tightly coupled to OpenClaw's release cycle — version pinning required (ADR: gateway version lag).
- Documentation is community-driven — some trial-and-error required for edge cases.

---

## ADR-003: 1Password Hybrid Secrets Management

**Status**: Accepted  
**Context**: The system needs runtime secrets injection (API keys for Gemini, Anthropic, ElevenLabs, etc.) without storing plaintext on persistent disk. 1Password Individual plan doesn't support service accounts or Connect Server — the standard enterprise patterns for automated credential access.

**Decision**: Hybrid model combining two mechanisms:
1. **systemd EnvironmentFile**: At boot, a systemd unit runs `op run` to populate `/run/openclaw-credentials/.env` on tmpfs (RAM-backed filesystem). The OpenClaw gateway unit loads this file via `EnvironmentFile=`.
2. **Config substitution**: `openclaw.json` uses `${ENV_VAR}` syntax for fields that don't support native SecretRef. Fields that support SecretRef use OpenClaw's `secrets.providers` with `exec` type (calls `op read`).

**Consequences**:
- Zero plaintext secrets on persistent disk — `/run/` is tmpfs (cleared on reboot).
- Two credential paths (env substitution + SecretRef) add complexity but cover all config fields.
- 1Password CLI must be authenticated (device trust) — one-time interactive setup per machine.
- `openclaw doctor` may report "unresolved SecretRef" when run outside systemd context — false negative (secrets resolve at runtime).

---

## ADR-012: Docker Privilege Model

**Status**: Accepted  
**Context**: Docker sandboxing with `--cap-drop=ALL` removes all Linux capabilities, including `DAC_OVERRIDE` (bypassing file permissions). Agent processes running as PID 1 inside containers cannot write to bind-mounted workspace directories even when running as root in the container, because the host filesystem enforces POSIX permissions and the container's root lacks `DAC_OVERRIDE`.

**Decision**:
- Apply `--cap-drop=ALL` and `--security-opt=no-new-privileges` to all agent containers.
- Set `chmod 777` on workspace directories before container launch.
- Accept that filesystem permissions are not the security boundary — the **container itself** is the boundary (no network, dropped capabilities, no privilege escalation).
- Cross-agent data isolation is enforced by separate bind mounts (`scope: "agent"`), not POSIX permissions within any single container.

**Consequences**:
- Strongest possible capability restriction (`--cap-drop=ALL`) maintained.
- `chmod 777` is a pragmatic trade-off: the workspace is agent-scoped and container-isolated.
- If a container is compromised, the attacker has no network, no capabilities, and no ability to escalate privileges — file read/write within the workspace is the maximum blast radius.

---

## ADR-014: Modular Domain Team Architecture

**Status**: Accepted  
**Context**: The initial plan defined 4 fixed agents (main, ops, research, startup). As requirements grew, this rigid structure couldn't accommodate new domains (job search, email triage, content creation) without architectural changes.

**Decision**: Shift to modular domain teams with depth-2 nesting:
- **Orchestrator (main)**: Routes tasks to appropriate team leads, manages system config.
- **Team Leads** (depth-1): Domain specialists (research-lead, startup-lead, jobs-lead) that understand their domain context.
- **Workers** (depth-2): Spawned by leads for specific subtasks, inherit parent's sandbox.
- Teams added incrementally — new lead + Discord channel + tool policy is all that's needed.

**Consequences**:
- New domains require config additions, not architectural changes.
- Workers inherit parent sandbox policies — security model scales automatically.
- Orchestrator complexity increases with team count — mitigated by clear delegation patterns and tool policy isolation.

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
- Self-improving behavior within safe guardrails — owner approval required for high-impact changes.
- Weekly review provides observability into agent behavior patterns.
- Cross-agent knowledge sharing prevents domain silos.

---

## ADR-020: Runtime Change Protocol

**Status**: Accepted  
**Context**: Configuration changes to a live agent system carry risk — a bad config can deadlock all agents (see elevated exec deadlock), break sandbox isolation, or wipe security allowlists (see exec-approvals.json trailing comma bug). Ad-hoc changes via interactive sessions lack audit trails and rollback capability.

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
- Slightly more ceremony per change — accepted trade-off for production stability.
