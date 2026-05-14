# Figma Partner Agent

A Claude Code subagent that researches Figma technology/integration partners, tags recent activity to your tracked accounts, and writes a Markdown report.

## Layout

```
.claude/agents/partner-activity.md   # the subagent definition
accounts/partners.csv                # your account list (edit this)
accounts/focus.md                    # current-quarter themes (optional)
reports/                             # generated reports land here
```

## Setup

1. Open `accounts/partners.csv` and add your partners. Columns:
   `name,domain,github_org,figma_plugin_id,tags,owner,notes`
   Lines starting with `#` are ignored. Only `name` is required, but the more
   fields you fill in, the better the research.
2. (Optional) Edit `accounts/focus.md` with current priorities to bias the
   executive summary.

## Usage

In Claude Code, invoke the agent:

```
@partner-activity generate this week's partner report
@partner-activity report on Linear and Notion only, last 14 days
@partner-activity what's new with our partners?
```

The agent will:

- Read `accounts/partners.csv` and `accounts/focus.md`
- Run WebSearch + WebFetch for recent press/blog activity
- Pull GitHub releases and commit activity for each partner's `github_org`
- Look for Figma-specific signals (plugins, Code Connect, Config mentions)
- Tag each finding (`launch`, `funding`, `personnel`, `figma-integration`,
  `partnership`, `content`, `risk`) and attach the account `owner`
- Write `reports/partner-activity-YYYY-MM-DD.md` with sources cited inline

The agent never mutates `accounts/partners.csv` on its own — it proposes tag
updates in the report, and you apply them if you agree.
