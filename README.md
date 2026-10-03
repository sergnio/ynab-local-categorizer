# YNAB Categorizer (local LLMs)
Suggests categories for uncategorized YNAB transactions using your own categorization history, with local LLMs only. Your financial data never goes to a cloud model.

Feel free to fork this and make it your own.

## How it works

1. A local model reads your already-categorized transactions as reference.
2. For each uncategorized transaction it suggests a category, with a confidence and the past transactions that support it.
3. You review the suggestions. Nothing is written to YNAB until you approve each batch.

There are no scripts yet: a local agent does the work by following [AGENTS.md](./AGENTS.md).

## Requirements

- A local LLM runtime, e.g. [Ollama](https://ollama.com) or [LM Studio](https://lmstudio.ai)
- An MCP-capable agent client that runs that local model (e.g. LM Studio, or [Goose](https://block.github.io/goose/) with Ollama)
- Node.js, for the YNAB MCP server
- A YNAB account

## Setup

1. **Fork or clone** this repo, then create the data folder:
   ```sh
   mkdir personal-data
   ```
   Everything personal (exports, caches, suggestion reports) goes in `personal-data/`, which is git-ignored.
2. **Get a YNAB Personal Access Token**: YNAB > Account Settings > Developer Settings > New Token ([docs](https://api.ynab.com/#personal-access-tokens)).
3. **Store the token** in a secret manager instead of a plain env var or config file. Recommended: [Automic Vault](https://www.automicvault.com):
   ```sh
   av save YNAB_API_TOKEN
   ```
4. **Install [ynab-mcp-server](https://github.com/calebl/ynab-mcp-server)**:
   ```sh
   git clone https://github.com/calebl/ynab-mcp-server.git
   cd ynab-mcp-server && npm install && npm run build
   ```
   Launch it through the vault so the token only exists in the server's process, e.g. a `run-with-vault.sh` kept outside this repo:
   ```sh
   #!/bin/sh
   exec av inject +YNAB_API_TOKEN -- node /path/to/ynab-mcp-server/dist/index.js
   ```
   Optionally set `YNAB_PLAN_ID` in that script so tools don't need a plan ID each call (find it with the `ynab_list_plans` tool).
5. **Register the MCP server in your local-model client only**, using `run-with-vault.sh` as the command. Don't connect it to cloud assistants, and leave its optional TypeSafe suggestions off (they send data to a third party).
6. **Lock out cloud agents** if you also use Claude Code or Codex on this machine. `.claude/settings.json` is already committed. For global Claude rules and Codex, follow [Enforcement setup in AGENTS.md](./AGENTS.md#enforcement-setup-for-a-fresh-clone-or-fork) and run its fake-data verification.

## Usage

Open your local-model client in this repo and ask it to categorize your uncategorized YNAB transactions. It follows [AGENTS.md](./AGENTS.md): pulls history and uncategorized transactions through the MCP server, writes a suggestions report to `personal-data/`, and asks you to approve before updating anything in YNAB.

Instead of the MCP server, you can also export your plan from the YNAB web app and drop the export in `personal-data/`.
