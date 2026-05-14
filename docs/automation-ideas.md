# Automation Ideas for Partner Workflow

_Source: Slack scan, 2026-04-14 → 2026-05-14. Channels surveyed: `#bd-partnerships-data`, `#partner-intel`, `#product-partnerships`, `#a-mcp-server`, plus ~10 `#ext-<partner>-figma-*` rooms (Dovetail, Maze, Supabase, Anthropic, Firebender, Augment, Factory, Asana, Zapier, Dovetail), and your sent messages._

## What the Slack scan shows you're actually doing

1. **Recurring partner scorecard work** — building MAU/WAU scorecard for ~51 tech partners against `DWH_ANALYTICS.EXTENSIBILITY_PARTNERSHIPS` (`PARTNER_RESOURCES_MAU`, `DIM_MONDAY_PARTNER_RESOURCE`). Currently blocked on Snowflake role/PII masking, but this is the highest-leverage repeating task you have.
2. **Multi-channel partner support** — you're the BD point person in many `#ext-*-figma-*` rooms, fielding MCP-server bugs, gateway requests, and integration questions. Each thread requires the same loop: triage → repro/check known issues → draft a response → escalate to product/leadership if needed.
3. **Information hygiene** — you regularly have to translate internal context (gateway-approval status, exec names, renewal timing) into something safe to post externally. One slip is expensive.
4. **Partner intelligence** — `#partner-intel` exists but is sparsely used; activity reports happen ad-hoc.
5. **Asana sync** — partner workstreams (e.g. MCP Evals in `#t-build-code-experimentation`) live in Asana; status updates happen in Slack and don't always make it back.

## Recommended automations, ranked by leverage

### 1. Partner Scorecard Agent  ⭐ highest leverage
**What it does:** runs the recurring MAU/WAU scorecard for the ~51 tech partners and writes a dated Markdown report (and optionally posts a summary to `#partner-intel`).
**How it works:**
- Reads `accounts/partners.csv` (already exists) to map partners → expected resources.
- Uses the **Hex MCP** (`create_thread`, `continue_thread`) to query `DWH_ANALYTICS.EXTENSIBILITY_PARTNERSHIPS.PARTNER_RESOURCES_MAU` and `DIM_MONDAY_PARTNER_RESOURCE` for the rolling window.
- Computes WAU (north-star) and MAU (headline rank), week-over-week delta, top movers, dormant partners.
- Writes `reports/scorecard-YYYY-WW.md`; optionally drafts a Slack message via `slack_send_message_draft`.
**Form:** new subagent at `.claude/agents/partner-scorecard.md`.
**Effort:** ~half day once Snowflake role unblock lands; can ship in dry-run mode (queries Hex, prints SQL) before then.
**Unblocker tracked elsewhere:** `#bd-partnerships-data` access request.

### 2. Partner Channel Digest Agent  ⭐ daily time-saver
**What it does:** every morning, sweeps the `#ext-*-figma-*` channels you're in plus `#a-mcp-server` and produces a one-screen action list: open threads awaiting your reply, classified by type (MCP issue, gateway request, integration bug, FYI), with the partner's last message quoted and a suggested next step.
**How it works:** `slack_search_public` with `to:@me is:thread` and recent-channel sweeps; `slack_read_thread` for each hit; classify and rank by staleness.
**Form:** subagent + a `/partner-digest` slash command.
**Effort:** ~1–2 hours. No new MCP needed.

### 3. External-Comms Sanitizer  ⭐ risk reduction
**What it does:** before you post in any `#ext-*` channel, runs your draft through a guardrail check. Flags internal-only references — exec names, "gateway approval", "leadership review", renewal/quota language, the Cequence/SoFi pattern you flagged on May 14 — and proposes an externally-safe rewrite.
**How it works:** a slash command `/sanitize <draft>` that uses a tight prompt + a small `docs/internal-terms.md` allow/deny list. Optional: a PreToolUse **hook** on `slack_send_message` that intercepts messages targeting channels matching `ext-*` and asks for confirmation if any term hits.
**Form:** slash command first (low risk); add the hook later if the command proves useful.
**Effort:** ~1 hour for the command; another hour for the hook.

### 4. MCP Issue Triage Agent
**What it does:** when a partner reports an MCP-server bug, the agent gathers context (their setup, the model they're using, error messages), searches `#a-mcp-server` history for prior occurrences, checks the public Help Center articles (you have five live: Claude Code, Codex, etc.), and drafts a response. You approve and send.
**How it works:** Slack search + WebFetch on `help.figma.com/hc/en-us/articles/...` MCP articles; optionally GitHub MCP to check the partner's repo for an mcp config file.
**Form:** subagent `.claude/agents/mcp-triage.md`, invokable as `@mcp-triage <thread permalink>`.
**Effort:** ~half day.

### 5. Promote `partner-activity` to publish to `#partner-intel`
**What it does:** extends the agent we built today so that, in addition to writing a Markdown report, it posts the executive summary as a Slack message to `#partner-intel` (or drafts it). Solves the "sparsely used channel" problem.
**How it works:** add `slack_send_message_draft` to the agent's tools; one more step at the end of its system prompt.
**Form:** edit existing agent.
**Effort:** 10 minutes.

### 6. Asana ↔ Slack thread sync
**What it does:** when you react to a Slack thread with a specific emoji (say `:asana:`), an agent creates or updates an Asana task in the right project (e.g. MCP Evals board in `#t-build-code-experimentation`) with a link back to the thread.
**How it works:** Asana MCP (`create_tasks`, `update_tasks`, `get_projects`) + Slack read. Today this would be a manual `/asana-from-thread` command; full reaction-trigger needs a hook plus a small watcher loop.
**Form:** start as a slash command.
**Effort:** ~half day for the command.

### 7. Config 26 Partner Day prep agent
**What it does:** pulls the confirmed partner list from `#config26-partner-day`, cross-references your `accounts/partners.csv`, identifies missing assets (logo, demo, talking points), and generates a run-of-show stub.
**Form:** one-shot slash command `/config-partner-day-status`.
**Effort:** ~1 hour. Worth doing once Config date gets close.

## Suggested build order

1. **Partner Channel Digest** (week 1) — immediate daily payoff, no blockers.
2. **External-Comms Sanitizer** (week 1) — cheap and reduces a real risk you already navigate manually.
3. **Promote `partner-activity` to Slack** (10 min) — closes the loop on the agent we just built.
4. **Partner Scorecard Agent** (week 2, dry-run; full run once Snowflake unblock).
5. **MCP Triage**, **Asana sync**, **Config Day prep** — opportunistic.

## What I'd need from you to start

- For #1/#2: confirm the canonical list of `#ext-*` channels to include (or "all I'm a member of").
- For #3 (Sanitizer): a short list of internal-only terms/phrases to seed the deny list (you already used a perfect example on May 14 — the Dylan/Loredana/Cequence message).
- For #4 (Scorecard): the column dictionary for `PARTNER_RESOURCES_MAU` once your Snowflake role is upgraded.
