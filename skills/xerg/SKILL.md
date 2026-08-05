---
name: xerg
description: Audit and reduce AI agent runtime spend in dollars. Use for AI costs, agent spend, token waste, runtime attribution, detector coverage, and FinOps. Works with OpenClaw, Hermes, QM, Claude Code, Cursor, and generic event ingest.
homepage: https://xerg.ai
metadata:
  xerg:
    homepage: https://xerg.ai
    links:
      skill: https://xerg.ai/skill.md
      documentation: https://xerg.ai/docs
      status: https://status.xerg.ai
    requires:
      anyBins:
        - xerg
        - npx
      config:
        - ~/.xerg/config.json
        - ~/.config/xerg/credentials.json
        - ~/.xerg/remotes.json
        - ~/.xerg/qm.json
    install:
      - kind: node
        package: "@xerg/cli"
        bins:
          - xerg
    envVars:
      - name: XERG_API_KEY
        required: false
        description: Advanced CI-only credential for non-interactive hosted push and MCP setup.
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
      - name: fly
        type: other
        url: https://fly.io/docs/flyctl/
---

# Xerg

Xerg is a local-first CLI for finding wasted AI runtime spend. Runtime audits separate identified waste from savings opportunities, report detector coverage, and use `--compare` to measure compatible changes. Runtime costs may be observed, locally estimated, or unpriced; they are not authoritative provider invoices.

## Cold-fetch setup

When this file was fetched from `https://xerg.ai/skill.md` rather than loaded from an installed skill:

1. Identify the current agent host. OpenClaw, Hermes, QM, Claude Code, Cursor, and Codex are setup hosts; they are not automatically the data source being audited. QM can also be the audit source, but only through its explicit administrator-provisioned Xerg connection.
2. Check whether Xerg is already installed for this host. Before writing to a persistent or global skill directory, explain the destination and ask the user for permission.
3. If the host has a supported skill installer, use it after permission. OpenClaw can install `xerg` from ClawHub. Agent Skills hosts can install the public `xergai/skills` repository. Hermes can use Xerg's well-known skill discovery. Deterministic host-specific commands live at [xerg.ai/agents](https://xerg.ai/agents).
4. If persistent installation is unavailable or declined, use these fetched instructions for the current session. Do not invent an install location.
5. Run `npx @xerg/cli@latest doctor`, use its output to choose the non-interactive audit command below, and offer to run the first audit locally. On QM, do not use bare doctor: follow the private-scope QM procedure below.

Always ask before uploading. A local audit needs no account and must not be pushed automatically. After a successful local result, offer to connect and push it with:

```bash
npx @xerg/cli@latest activate --push-latest
```

`activate` opens Xerg in the browser. The signed-in user explicitly approves the currently active workspace; the page shows its full organization ID, live plan, and API environment before any credential is issued. If the intended Clerk ID is already known, recommend `--organization-id org_...` so both the page and API reject another active workspace. The organization switcher preserves the pairing code. The workspace key is encrypted to the initiating CLI and stored with owner-only, environment-bound metadata; never ask the user to paste a key into chat or expose one in a command.

If approval covers pairing only, use `activate --connect-only --organization-id org_...`. It must stop after verified credential storage and must not be combined with a source or push flag. A later audit or push requires its own explicit approval.

Codex is an execution host, not a native Xerg audit source. A Codex user audits whatever `doctor` finds: OpenClaw, Hermes, Claude Code, a Cursor export, or a generic ingest payload.

If the current host is QM, a cold `set up https://xerg.ai/skill.md` request can explain prerequisites but cannot claim it persisted an admin skill pack, created PostgreSQL privileges, or provisioned secrets. Require an administrator-only private scope. Refuse deployment-wide output in public/shared scopes. Never ask anyone to paste a DSN, identity key, Fly token, provider credential, or Xerg API key into Slack or chat.

## First run (agent execution path)

Use the non-interactive commands. Do not run `xerg init` unless the user explicitly asks for guided interactive setup — it is TTY-only and prompts for input.

```bash
npx @xerg/cli@latest doctor
npx @xerg/cli@latest audit --json
```

1. Run `doctor` first. It reports which local sources exist (OpenClaw, Hermes, Claude Code) and which default paths were checked. QM is never auto-probed.
2. Run `audit --json`. If more than one runtime is detected, add `--runtime openclaw`, `--runtime hermes`, or `--runtime claude-code`.
3. Summarize the result for the user in dollars: total spend, identified waste (`wasteSpendUsd`), assessed spend from `detectionCoverage`, the top findings, and the per-agent spend breakdown (`spendByAgent`) when present. Identified waste and savings opportunities are different claims — do not present opportunities as proven waste. Never describe `$0` as no waste when request-sequence coverage is none, partial, or unknown.
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
- Certified local Hermes trace enrichment: `npx @xerg/cli@latest collect hermes --state-db <path>` (interactive until `Ctrl-C`; state.db remains required)
- Remote OpenClaw over SSH: `npx @xerg/cli@latest audit --remote user@host`
- Railway-hosted OpenClaw: `npx @xerg/cli@latest audit --railway`
- Existing QM snapshot: `npx @xerg/cli@latest audit --runtime qm --qm-snapshot <snapshot.jsonl>`
- Configured QM direct/Fly source: an operator collects outside Slack; follow the private-scope procedure below only for an authorized snapshot

If `xerg` is installed globally, use `xerg` in place of `npx @xerg/cli@latest`.

## What It Audits

- OpenClaw gateway logs and session transcripts
- Optional independent OpenClaw trace captures created by the loopback traces-only collector
- Hermes v0.17+ `state.db` (read-only), with legacy log/transcript fallback where present
- QM durable run/model economics and current retained activity through `xerg_export/v1` and `qm-snapshot/v1`
- Claude Code session transcripts via `xerg audit --runtime claude-code`
- Cursor usage CSV exports via `xerg audit --cursor-usage-csv ./cursor-usage.csv`
- Any framework's exported event payload via `xerg ingest --file payload.json`

Xerg does not currently ingest provider bills, reconcile invoices, or convert runtime observations into FOCUS. If a user asks for one of those capabilities, explain the boundary and do not present modeled runtime spend as invoice-authoritative.

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
- Shared local `analysisCoverage`, `toolActivity`, and `workloadEconomics` blocks for OpenClaw and Hermes; these are neutral evidence and never affect findings, recommendations, waste totals, or CI gates
- QM request-versus-aggregate reconciliation, pricing coverage, source stability, pseudonymous scope attribution, and current-window unassociated tool activity

Costs are priced across input, output, cache read, and cache write tokens using a catalog covering hundreds of current models. Run `xerg doctor --verbose` to see per-file extraction coverage (which economic signals the parser found).

For current Hermes, use `xerg audit --runtime hermes`; Xerg prefers `~/.hermes/state.db`. Use `--state-db` for another profile. `--state-db` is mutually exclusive with legacy `--log-file` and `--sessions-dir`. State-only audits retain aggregate spend, request/token/cache, workflow/model/tool, and delegated-workload totals, but first-request cost, initial context, request growth, retry sequences, and identical-input loops are unavailable. The optional `xergai/hermes-observer` plugin writes content-free local events under `~/.hermes/xerg/events/`; it never adds economics or sends content to Xerg Cloud. Complete observer evidence can split an aggregate only after exact request/token reconciliation.

After installing the observer, restart Hermes, start a new session, and run `xerg doctor --runtime hermes`. Report doctor status and assessed spend. Do not imply that historical aggregate sessions can be reconstructed. Use `--require-detection-coverage full|partial` for CI; unmet coverage exits `5`.

For OpenClaw traces, `xerg collect openclaw` binds only to `127.0.0.1`, accepts OTLP/HTTP protobuf traces, persists a bounded sanitized capture, and audits after shutdown. It does not modify OpenClaw configuration or push automatically. Without explicit `--runtime hermes`, `--otlp-file` is independent OpenClaw evidence and cannot be combined with transcript, log, other-runtime, or remote sources.

For certified Hermes traces, `xerg collect hermes` uses the same loopback traces-only protocol but HMACs identifiers with a persistent local key and requires `state.db`. Use only the exact certified `briancaffey/hermes-otel` commit printed in Xerg's docs. Xerg never installs or edits that plugin. The first-party observer stays primary; cache-inclusive plugin prompt totals are normalized into Xerg's separate input/cache buckets, optional enrichment preserves `economicAuditId` while analysis identity reflects changed coverage, and conflicting evidence restores the original aggregate. For non-default `HERMES_HOME` profiles, use the equivalent environment-variable block printed by the command because the pinned plugin resolves YAML under `~/.hermes`.

## QM private-scope procedure

QM 0.19.0 uses a host-independent snapshot adapter. An operator outside Slack creates `qm-snapshot/v1` through either a strict view-only direct reader or the Fly-contained one-shot exporter. Fly contained mode uses QM core's existing `DATABASE_URL` only inside the hidden exporter, records a process boundary with no database-level least-privilege claim, and never exposes that credential to this skill. Do not mutate setup, install/persist the skill, activate, or push without explicit approval.

In a QM Slack scope, never initiate live database or Fly collection. Accept only an authorized sanitized snapshot already made available in an administrator-only private scope, and use only the exact CLI the operator has explicitly provisioned and version-verified inside that runtime. Current Fly Sprites do not apply the configured sandbox OCI image to persistent Sprites, so never assume an image pin installed the CLI. If it is absent, explain that prerequisite instead of installing it without explicit approval. A live capability bridge is deferred to 0.20.0. Never request or receive a DSN, identity key, Fly token, provider credential, or Xerg API key.

Run doctor before every first audit:

```bash
xerg doctor --runtime qm --qm-snapshot <snapshot.jsonl>
xerg audit --runtime qm --qm-snapshot <snapshot.jsonl> --since 7d --json
```

Operator documentation may explain `--database-url-env`, `--fly-app`, `--print-sql`, and `--print-views-sql`, but the agent must not execute or provision those live paths from Slack. Direct, Fly, and `--qm-snapshot` modes are mutually exclusive.

Explain the result conservatively:

- positive QM cost is observed; an explicit deterministic model may be catalog-estimated
- zero/negative placeholders and unresolved `openrouter/auto` are unpriced, never free or actual `$0`
- unpriced patterns say “monetary impact unavailable” and monetary gates exit `5`
- Pi/OpenCode timestamps are flush/capture time; Claude is near result-step recording; Codex rows are turn aggregates
- request-sequence detectors run only on exactly reconciled request observations; aggregate rows are excluded
- QM keeps activity for one hour, so older tool history is unavailable and tool-to-model association is not claimed
- retry attempts are neutral evidence without spend attribution; unsupported findings are not described as assessed

Local snapshot audit is the default. Ask for explicit approval before `xerg activate --runtime qm`, `audit --push`, or any other hosted write. Xerg's hosted MCP can read an already-pushed pseudonymous summary but has no native QM database or Fly access.

## Optional Cloud

Local audits need no account. To connect a workspace and push the latest local result, ask permission and run:

```bash
npx @xerg/cli@latest activate --push-latest
```

For a website-first user with no cached audit, `npx @xerg/cli@latest activate` securely connects, detects a supported local source, runs the audit, and pushes it. Hosted sync and hosted MCP remain optional and never run without explicit user action.

## Advanced authentication

Use `xerg login --replace` only as a manual recovery path when browser pairing is unavailable. It opens Workspace Settings and masks the key pasted into the terminal. Never ask for or paste a workspace key in agent chat.

Use `XERG_API_KEY` only for non-interactive CI or deployment automation. Store it in the CI provider's secret manager; never place it inline in a shell command, source file, log, URL, or conversation.

## Links

- Docs: [xerg.ai/docs](https://xerg.ai/docs)
- Service status: [status.xerg.ai](https://status.xerg.ai)
- Skill: [xerg.ai/skill.md](https://xerg.ai/skill.md)
- npm: [@xerg/cli](https://www.npmjs.com/package/@xerg/cli)
- Support: `query@xerg.ai`
