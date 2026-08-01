# Xerg

Find wasted AI spend in OpenClaw, Hermes, Claude Code, Cursor, and any framework that can export a JSON event payload. Audit provider-generated FOCUS 1.4 billing data locally.

Xerg is a local-first CLI for auditing AI spend in dollars, not raw token counts. It reads OpenClaw logs/transcripts or an independent sanitized trace capture, Hermes v0.17+ `state.db` with optional observer/certified trace enrichment, Claude Code transcripts, and Cursor usage exports — plus event payloads from any framework via `xerg ingest` — separates identified waste from savings opportunities, reports what detector spend was assessed, and lets you measure compatible fixes with `--compare`.

Everything runs locally by default. The CLI is publicly installable from npm as `@xerg/cli`, but it is not open source. No account is required for local audits. A free hosted workspace keeps the last 30 days of pushed audits; hosted MCP requires a Pro or Enterprise workspace.

The `npx @xerg/cli@latest` path fetches and executes the published npm package before running Xerg. If you want to avoid that fetch on each use, install the CLI globally with `npm install -g @xerg/cli`.

## Install

Give a terminal-capable agent the universal cold-fetch prompt:

```text
set up https://xerg.ai/skill.md
```

The agent must ask before persistent installation, runs `doctor`, and offers a local audit before any upload.

Install this skill into any agent that supports the Agent Skills standard (Claude Code, Codex, Cursor, and others):

```bash
npx skills add xergai/skills
```

Then ask your agent to audit your AI spend. The agent runs the CLI itself and explains the findings.

Install the CLI directly:

```bash
npm install -g @xerg/cli
```

Or run without installing:

```bash
npx @xerg/cli@latest init
```

## What It Finds

- **Retry waste** - failed calls that burned spend before a later success
- **Loop waste** - runs that exceeded efficient iteration bounds
- **Context bloat** - input token volume far above the workflow baseline
- **Downgrade candidates** - expensive models on operationally simple tasks
- **Idle waste** - recurring heartbeat or monitoring loops worth reviewing

Local JSON findings can include `signalSource`, `ruleId`, and evidence references so agents can distinguish observed signals from inferred or legacy unknown provenance. Compare output leads with normalized identified-waste rate and per-unit rows only when detector coverage is compatible, before workload-dependent spend deltas.

## Quick Start

Interactive first run (humans):

```bash
xerg init
xerg audit --compare
```

Non-interactive path (agents, scripts, CI):

```bash
xerg doctor
xerg audit --json
xerg audit --json --compare
xerg collect openclaw
xerg collect hermes --state-db ~/.hermes/state.db
xerg audit --otlp-file ./openclaw.capture.jsonl
```

Add `--runtime openclaw`, `--runtime hermes`, or `--runtime claude-code` when more than one runtime is detected.

## Sources

- Local machine: OpenClaw, Hermes, and Claude Code (`xerg audit --runtime claude-code`)
- Independent local OpenClaw traces: `xerg collect openclaw` or `xerg audit --otlp-file <capture.jsonl>`
- Certified local Hermes trace enrichment: `xerg collect hermes --state-db <path>` or `xerg audit --runtime hermes --state-db <path> --otlp-file <capture.jsonl>`
- Local Cursor usage export: `xerg audit --cursor-usage-csv ./cursor-usage.csv`
- Any framework's exported event payload: `xerg ingest --file payload.json`
- Remote OpenClaw sources via SSH or configured remote transports
- Provider-generated FOCUS 1.4 CSV or Parquet: `xerg focus init --input <directory>` then `xerg focus audit --bundle <manifest.json>`

If local defaults are empty, inspect the target directly first with `xerg doctor --remote user@host`.

## Optional Hosted Follow-Up

```bash
xerg connect
xerg mcp-setup
```

- `connect` offers browser auth and pushing the latest audit
- `mcp-setup` prints or writes hosted MCP config for supported clients
- local audits and compare remain available if you skip hosted setup

## Security And Data Flow

- Local audits read OpenClaw, Hermes, Claude Code, Cursor usage, or ingest payload files and may write local JSON snapshots for `--compare`.
- Hermes uses `~/.hermes/state.db` read-only by default and as its sole monetary authority. Optional observer telemetry and certified trace enrichment preserve authoritative token buckets and `economicAuditId`; analysis `auditId` changes when coverage/findings change. Push v5 includes only content-free detector eligibility and source stability; detailed mechanics remain local.
- Both trace collectors bind only to loopback, sanitize before persistence, receive traces only, and never push automatically. `analysisCoverage`, `toolActivity`, and `workloadEconomics` remain local.
- Remote OpenClaw audits pull selected files to local temporary storage before analysis.
- Xerg Cloud sync only happens when you run `connect`, `audit --push`, or `push`.
- Push payloads include audit totals, rollups, findings, recommendations, comparison deltas, and source metadata. They exclude raw prompt and response content, local source file paths, local snapshot store paths, and internal finding details.
- Rollup-level provenance (per-finding `signalSource`, waste-by-signal-source totals, and pricing coverage counts) is carried on the wire; evidence references and internal details stay local.
- FOCUS raw files and rows remain local. Its explicit USD-only push contains bounded aggregate totals and excludes paths, customer/invoice/contract IDs, descriptions, tags, metadata URLs, SKU/model labels, and custom values.

## CI And Automation

```bash
xerg audit --push --fail-above-waste-rate 0.25
xerg audit --fail-above-waste-usd 100
xerg audit --json
```

## Links

- Docs: [xerg.ai/docs](https://xerg.ai/docs)
- Service status: [status.xerg.ai](https://status.xerg.ai)
- Skill: [xerg.ai/skill.md](https://xerg.ai/skill.md)
- npm: [@xerg/cli](https://www.npmjs.com/package/@xerg/cli)
- Pricing: [xerg.ai/pricing](https://xerg.ai/pricing)
- Support: `query@xerg.ai`

## Ownership

This skill is maintained by [Xerg](https://xerg.ai) and synced to the public [xergai/skills](https://github.com/xergai/skills) repo from the Xerg release train, versioned with `@xerg/cli`. The skill content is free to install and use with any agent; the Xerg CLI it drives is publicly installable from npm but not open source.
