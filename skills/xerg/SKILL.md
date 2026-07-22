---
name: xerg
description: Audit and reduce AI agent spend in dollars. Use when the user asks about AI costs, agent spend, token waste, retry loops, model downgrades, per-agent spend, or wants to measure whether a workflow or model change saved money. Works with OpenClaw, Hermes, Claude Code, Cursor, and any framework via event ingest.
homepage: https://xerg.ai
metadata:
  xerg:
    homepage: https://xerg.ai
    links:
      skill: https://xerg.ai/skill.md
      documentation: https://xerg.ai/docs
      status: https://status.xerg.ai
    primaryEnv: XERG_API_KEY
    requires:
      anyBins:
        - xerg
        - npx
      config:
        - ~/.xerg/config.json
        - ~/.config/xerg/credentials.json
        - ~/.xerg/remotes.json
    install:
      - kind: node
        package: "@xerg/cli"
        bins:
          - xerg
    envVars:
      - name: XERG_API_KEY
        required: false
        description: Optional Xerg Cloud workspace API key for explicit push, connect, and hosted MCP setup.
      - name: XERG_API_URL
        required: false
        description: Optional override for the Xerg API endpoint; defaults to https://api.xerg.ai.
    dependencies:
      - name: "@xerg/cli"
        type: npm
        url: https://www.npmjs.com/package/@xerg/cli
      - name: ssh
        type: other
        url: https://www.openssh.com/
      - name: rsync
        type: other
        url: https://rsync.samba.org/
      - name: railway
        type: npm
        repository: https://github.com/railwayapp/cli
---

# Xerg

Xerg is a local-first CLI for finding wasted AI spend in OpenClaw, Hermes, Claude Code, and Cursor — plus any other framework that can export a JSON event payload. It audits dollars instead of raw token counts, separates confirmed waste from savings opportunities, attributes spend per agent (including delegated sub-agent spend where the source supports it), and uses `--compare` to measure whether a workflow or model change actually helped.

## First run (agent execution path)

Use the non-interactive commands. Do not run `xerg init` unless the user explicitly asks for guided interactive setup — it is TTY-only and prompts for input.

```bash
npx @xerg/cli@latest doctor
npx @xerg/cli@latest audit --json
```

1. Run `doctor` first. It reports which local sources exist (OpenClaw, Hermes, Claude Code) and which default paths were checked.
2. Run `audit --json`. If more than one runtime is detected, add `--runtime openclaw`, `--runtime hermes`, or `--runtime claude-code`.
3. Summarize the result for the user in dollars: total spend, confirmed waste (`wasteSpendUsd`), the top findings, and the per-agent spend breakdown (`spendByAgent`) when present. Confirmed waste and savings opportunities are different claims — do not present opportunities as proven waste.
4. If the user applies a fix, re-run the same audit with `--compare` to report the before/after delta:

```bash
npx @xerg/cli@latest audit --json --compare
```

If no local data is found, `doctor` prints the paths it checked. Fallbacks:

- Cursor usage CSV export: `npx @xerg/cli@latest audit --cursor-usage-csv ./cursor-usage.csv`
- Claude Code transcripts elsewhere: `--claude-code-dir <path>`
- Hermes profile database: `--runtime hermes --state-db <path>`
- Any framework's exported event payload: `npx @xerg/cli@latest ingest --file payload.json`
- Existing sanitized OpenClaw trace capture: `npx @xerg/cli@latest audit --otlp-file <capture.jsonl>`
- New local OpenClaw trace capture: `npx @xerg/cli@latest collect openclaw` (interactive until `Ctrl-C`; use only when the user asks to collect a workload)
- Remote OpenClaw over SSH: `npx @xerg/cli@latest audit --remote user@host`
- Railway-hosted OpenClaw: `npx @xerg/cli@latest audit --railway`

If `xerg` is installed globally, use `xerg` in place of `npx @xerg/cli@latest`.

## What It Audits

- OpenClaw gateway logs and session transcripts
- Optional independent OpenClaw trace captures created by the loopback traces-only collector
- Hermes v0.17+ `state.db` (read-only), with legacy log/transcript fallback where present
- Claude Code session transcripts via `xerg audit --runtime claude-code`
- Cursor usage CSV exports via `xerg audit --cursor-usage-csv ./cursor-usage.csv`
- Any framework's exported event payload via `xerg ingest --file payload.json`

## What It Finds

- Retry waste from failed calls before a later success (explicit or transcript-inferred on session sources)
- Loop waste from runs that exceed efficient iteration bounds, and repeated identical tool sequences
- Context bloat from unusually large inputs, and per-session context growth that compaction would cut
- Cache-write churn where prompt-cache premiums are paid without reuse
- Downgrade candidates where cheaper models may be enough
- Idle spend from fixed-cadence scheduled loops
- Per-agent spend attribution, including delegated sub-agent spend for Claude Code sidechains and ingest payloads
- Cost per outcome when runs carry outcome signals; declare outcomes with `xerg outcome --workflow <name> --status success|failure`
- Separate local Hermes mechanical metrics when the optional observer is enabled; these have no dollar classification, recommendation impact, or CI-gate effect
- Separate local OpenClaw `toolActivity` and `workloadEconomics` blocks; these are neutral evidence and never affect findings, recommendations, waste totals, or CI gates

Costs are priced across input, output, cache read, and cache write tokens using a catalog covering hundreds of current models. Run `xerg doctor --verbose` to see per-file extraction coverage (which economic signals the parser found).

For current Hermes, use `xerg audit --runtime hermes`; Xerg prefers `~/.hermes/state.db`. Use `--state-db` for another profile. `--state-db` is mutually exclusive with legacy `--log-file` and `--sessions-dir`. The optional `xergai/hermes-observer` plugin writes content-free local events under `~/.hermes/xerg/events/`; it never adds economic calls or sends data to Xerg Cloud.

For OpenClaw traces, `xerg collect openclaw` binds only to `127.0.0.1`, accepts OTLP/HTTP protobuf traces, persists a bounded sanitized capture, and audits after shutdown. It does not modify OpenClaw configuration or push automatically. `--otlp-file` is independent and cannot be combined with transcript, log, other-runtime, or remote sources.

## Optional Cloud

Local audits need no account. Hosted sync and hosted MCP are optional workspace features and only run when you explicitly use `xerg connect`, `xerg audit --push`, `xerg push`, or `xerg mcp-setup`.

## Links

- Docs: [xerg.ai/docs](https://xerg.ai/docs)
- Service status: [status.xerg.ai](https://status.xerg.ai)
- Skill: [xerg.ai/skill.md](https://xerg.ai/skill.md)
- npm: [@xerg/cli](https://www.npmjs.com/package/@xerg/cli)
- Support: `query@xerg.ai`
