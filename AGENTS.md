# AGENTS.md

## Hard rule: no non-local LLMs on YNAB data

The owner's YNAB data must never reach a non-local (cloud-hosted) LLM. This covers Claude, GPT, Gemini, Codex, and any other hosted model, plus third-party AI processors such as TypeSafe.

If you are a cloud-hosted model, stop immediately. Do not:
- read, grep, cat, or open `personal-data/` or any data file in this folder (`*.csv`, `*.json`, exports, caches, generated HTML reports), or YNAB-derived files anywhere else, including temp directories
- call the YNAB API, the YNAB MCP server, or inject the YNAB token
- run any script whose output contains transactions, payees, memos, amounts, accounts, or categories

A cloud model may only edit code and docs here, and only when asked, without running that code against real data. When in doubt, don't touch it.

Only a model running entirely on this machine (e.g. Ollama, llama.cpp, LM Studio) may read the data.

Never loosen the enforcement below.

## Enforcement setup (for a fresh clone or fork)

All personal data lives in `personal-data/` (git-ignored, read-denied). Stray `*.csv`, `*.tsv`, `*.xlsx` files are git-ignored and read-denied as a safety net. Use `<repo>` below for this clone's absolute path.

### Claude Code

`.claude/settings.json` (committed) already provides Read-tool deny rules and a strict Bash sandbox (`allowUnsandboxedCommands: false`, `failIfUnavailable: true`). It only applies when Claude Code is started inside this repo. To also block sessions started elsewhere, add absolute-path rules to `~/.claude/settings.json` under `permissions.deny`:

```json
"Read(<repo>/personal-data/**)",
"Read(<repo>/**/*.csv)",
"Bash(*YNAB_API_TOKEN*)",
"mcp__ynab"
```

### Codex

Project-level `.codex/config.toml` can't set permission profiles, so this goes in `~/.codex/config.toml`. The profile extends the built-in workspace sandbox and denies reads:

```toml
default_permissions = "workspace-no-ynab"

[permissions.workspace-no-ynab]
extends = ":workspace"

[permissions.workspace-no-ynab.filesystem]
"<repo>/personal-data" = "deny"
"<repo>/**/*.csv" = "deny"
"<repo>/**/*.tsv" = "deny"
"<repo>/**/*.xlsx" = "deny"
```

Forbid token injection in `~/.codex/rules/default.rules` (adjust to your secret manager's command):

```
prefix_rule(pattern=["av", "inject", "+YNAB_API_TOKEN"], decision="forbidden")
```

Codex deny rules cover sandboxed commands only. A human must decline any escalation request that touches this repo's data.

### Verify with fake data only

Never test against real files. Create a probe, confirm reads are refused, and leave it for the human to delete:

```sh
mkdir -p personal-data && echo FAKE > personal-data/probe.csv
codex sandbox -- cat personal-data/probe.csv   # expect: Operation not permitted
```

In Claude Code, reading `personal-data/probe.csv` with the Read tool and with `cat` must both be refused.

### What the agent can't do here

The strict sandbox blocks agents from creating `.git`, editing `.claude/settings*.json`, switching git accounts, and moving or deleting data files. Hand those commands to the human to run (in Claude Code, prefixed with `!`, or in a normal terminal).

### YNAB access

Set up the YNAB token and MCP server as in [README.md](./README.md). Register the MCP server only in a local-model client, keep the server's optional TypeSafe suggestions off, and never add it to `.mcp.json` for a cloud assistant (`.mcp.json` is git-ignored).

## Purpose

Use categorized past transactions as reference to suggest (or determine) the right category for uncategorized YNAB transactions. The owner reviews suggestions before anything is written back to YNAB.

## Workflow (local agents only)

1. **History**: fetch categorized transactions with `ynab_get_transactions` (`type: "all"`, `limit` up to 1000, page backwards with `sinceDate`), or read a YNAB export from `personal-data/`. Cache anything you save under `personal-data/`.
2. **Categories**: get valid targets with `ynab_list_categories`. Never suggest hidden, deleted, or internal categories.
3. **Targets**: fetch uncategorized rows with `ynab_get_transactions` (`type: "uncategorized"`). Skip transfers, starting balances, inflows to Ready to Assign, and splits.
4. **Suggest**, deterministic matching first:
   - normalize payee names (strip processor prefixes like `TST*`, `SQ *`, store numbers, city/state suffixes)
   - exact normalized-payee history, then recurring amount/date patterns, then memo keywords
   - use the local LLM only for leftovers history can't decide, with the closest history rows as context
5. **Report**: write `personal-data/suggestions-YYYY-MM-DD.md` with, per transaction: date, payee, amount, suggested category, confidence, and the past transactions used as evidence. Lowest confidence first, so the owner reviews the hard ones.
6. **Apply only after explicit approval** of each batch: call `ynab_update_transaction` with just `transactionId` and `categoryId`. Don't change approval, cleared status, memo, or other fields. Before writing, log each transaction's previous category to `personal-data/undo-YYYY-MM-DD.json`.

`ynab_apply_category_suggestions` needs fingerprints from the TypeSafe suggestion tool, so it is not usable here.

## Data notes

- YNAB stores amounts in milliunits (1/1000 of a dollar). Convert exactly once. The YNAB MCP server already returns dollars.
- The YNAB token lives in a secret manager (see README) and is injected only into the process that needs it.
- **YNAB CSV export amounts are 1000× too small** (exported in scaled milliunit form, not raw). Always multiply by 1,000 when rendering amounts from the CSV. Use `ynab_categorize.py` — it handles this automatically.

## Scripts

- `ynab_categorize.py` — reads `personal-data/ynab_transactions.csv`, filters uncategorized (excludes Starting Balance & Transfer), applies deterministic category suggestions, multiplies CSV amounts by 1,000, and writes an interactive HTML report to `/tmp/ynab_suggestions_v3.html`. Run with: `python3 ynab_categorize.py && npx -y lavish-axi open /tmp/ynab_suggestions_v3.html`
- `npx -y lavish-axi poll /tmp/ynab_suggestions_v3.html` — foreground poll for interactive category overrides.
