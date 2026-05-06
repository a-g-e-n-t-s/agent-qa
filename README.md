# agent-qa
> QA-oriented KADI agent that performs semantic validation and heuristic scoring for task reviews.

Overview
--------
agent-qa is a KADI (kadi-agent) that subscribes to review tasks and performs semantic validation using an LLM provider (Model Manager or Anthropic) with a heuristic fallback. It uses agents-library's BaseAgent to connect to a KADI broker, exposes memory-backed pattern recall, and registers a validation handler to score and review tasks published to the broker.

Quick Start
-----------
1. Install dependencies:
- `npm install`

2. Install the agent into KADI (local KADI CLI):
- `kadi install`

3. Start the agent:
- `kadi run start`

Quick development / local run
- Check prerequisites (script verifies node_modules): `npm run preflight`
- Run in watch-mode (ts -> node via tsx): `npm run dev`
- Build TypeScript: `npm run build` or `npm run setup`
- Start compiled output: `npm run start`

Tools
-----
| Tool | Description |
|------|-------------|
| broker (KADI WS) | WebSocket broker(s) used for messaging. Default remote broker: `wss://broker.dadavidtseng.com/kadi` (configured in agent.json and config.toml). Local broker can be configured via `[broker.local]` in config.toml and overridden with env vars `KADI_BROKER_URL_LOCAL` / `KADI_BROKER_URL_REMOTE`. |
| model-manager | Primary provider option when configured in config.toml (`provider.PRIMARY = "model-manager"`) and when `MODEL_MANAGER_BASE_URL` / `MODEL_MANAGER_API_KEY` are available (from env or vault). |
| anthropic | Fallback or primary provider when configured and `ANTHROPIC_API_KEY` is available. Uses `@anthropic-ai/sdk`. |
| secret-ability + other abilities | Declared abilities in `agent.json` (see Abilities below). The agent expects `kadi secret` or vault-delivered credentials for provider and arcadedb secrets in production deployments. |
| validation handler (`src/handlers/validation.js`) | Registers to `task.review_requested` events and scores/reviews tasks. Uses `baseAgent.providerManager` (LLM) and `baseAgent.memoryService` (past pattern recall) when available. |
| memory service (BaseAgent.memoryService) | Persistent memory-backed service. Default data path: `./data/memory` (configurable via config.toml or MEMORY_DATA_PATH env var). |
| provider manager (BaseAgent.providerManager) | Abstracted provider layer that routes requests to Model Manager or Anthropic depending on configuration (set in config.toml and credential availability). |
| arcadedb (optional) | Optional remote DB used in deploy configuration for logs/analytics (credentials delivered from vaults in agent.json deploy). |

Configuration
-------------
Agent configuration is read from config.toml (via agents-library.readConfig()) and may be supplemented/overridden by environment variables or secrets delivered from a vault (agents-library.loadVaultCredentials()). Environment variables take precedence over vault/config values.

Primary configuration sources and envs
- config.toml — primary configuration file (see `config.toml` in repo).
- `KADI_BROKER_URL_LOCAL` / `KADI_BROKER_URL_REMOTE` — override broker URLs defined in config.toml.
- `ANTHROPIC_API_KEY` — Anthropic API key (env takes precedence over vault).
- `MODEL_MANAGER_BASE_URL` — Model Manager base URL (env or vault).
- `MODEL_MANAGER_API_KEY` — Model Manager API key (env or vault).
- `MEMORY_DATA_PATH` — Filesystem path for memory persistence. Default: `./data/memory`.
- Vault-delivered secrets — loadable via agents-library; agent.json deploy also lists required vaults (anthropic, model-manager, arcadedb).

Provider selection behavior (implemented in src/index.ts -> buildProviderConfig)
- Primary/fallback provider names are read from config.toml (`[provider] PRIMARY` / `FALLBACK`).
- Credentials for providers are resolved from environment variables first, then from loaded vault secrets:
  - `ANTHROPIC_API_KEY`
  - `MODEL_MANAGER_BASE_URL` and `MODEL_MANAGER_API_KEY`
- If a configured primary provider has credentials available, it will be used as primary. If a fallback provider is configured and has credentials, it will be registered as fallback.
- If no provider credentials are available the agent will run in heuristic-only scoring mode (LLM semantic review disabled) and will log a warning.

Files and important config locations
- Agent metadata: `agent.json` (name, version, scripts, build config, abilities, brokers)
- Runtime configuration: `config.toml` (broker URLs, networks, provider choices, memory path, logging)
- Entrypoint: `src/index.ts`
- Compiled entrypoint: `dist/index.js`
- Validation handler: `src/handlers/validation.js`
- Memory default path: `./data/memory` (override with `MEMORY_DATA_PATH` or config.toml)
- Build steps (in `agent.json`): includes `kadi install kadi-secret` and `kadi install` as part of production build

Architecture
------------
High-level components and data flow:
1. BaseAgent
   - Implemented by agents-library and constructed in `src/index.ts`.
   - Configured with fields: `agentId` (from config.toml), `agentRole` (`programmer` in code), `version` (from config.toml), broker(s), provider settings, and memory data path.
   - Exposes `client`, `providerManager`, and `memoryService`.
2. Broker (WebSocket)
   - The agent connects to at least one configured broker (local or remote) — URLs come from config.toml and may be overridden with `KADI_BROKER_URL_LOCAL` / `KADI_BROKER_URL_REMOTE`.
   - Subscribes/publishes KADI messages via BaseAgent.client.
3. Validation Handler (src/handlers/validation.js)
   - Subscribes to `task.review_requested` messages.
   - On review request:
     - If `providerManager` is available, performs LLM-based semantic review using configured primary/fallback providers.
     - Uses `memoryService` to fetch past patterns and context for better recall.
     - Falls back to heuristic-only scoring if no LLM provider is configured/available.
   - Publishes review results back to the broker.
4. Provider Manager
   - Routes LLM calls to configured providers (Model Manager preferred if configured in config.toml).
5. Memory Service
   - Stores and retrieves past patterns to improve validation decisions and enable recall.

Key runtime behaviors
- Reads config.toml via agents-library.readConfig(); at least one broker (local or remote) must be configured or the agent will throw on startup.
- Loads secrets via agents-library.loadVaultCredentials(); environment variables override vault values.
- Logs startup summary including broker URLs, networks and LLM provider summary (primary/fallback and model names when present).
- On fatal startup errors the process exits with code 1 after logging the error.

Development
-----------
Scripts (from agent.json / package.json)
- `npm run preflight` — Checks that dependencies are installed.
- `npm run setup` — Invokes the TypeScript build (alias to `npm run build`).
- `npm run start` — `node dist/index.js` (run compiled JS).
- `npm run dev` — `tsx watch src/index.ts` (fast local development with watch).
- `npm run build` — `npx tsc` (TypeScript build).
- `npm run type-check` — `npx tsc --noEmit` (type-only checking).
- `npm run lint` — `npx eslint src --ext .ts`
- `npm run test` — `npx vitest`
- `npm run clean` — Removes node_modules/dist and related build artifacts.

Building for production
- agent.json build steps include:
  - `npm ci --include=dev`
  - `kadi install kadi-secret`
  - `kadi install`
  - `npx tsc`
  - `npm prune --omit=dev`
- Build environment sets `NODE_ENV=production` in the image configuration.

Dependencies (high-level)
- Runtime: `@anthropic-ai/sdk`, `agents-library`, `zod`, `@kadi.build/core`
- Dev: TypeScript, ESLint, Vitest, tsx

Recommended workflow
1. Populate credentials (env preferred). Examples:
   - `export KADI_BROKER_URL_LOCAL="ws://localhost:8080/kadi"`
   - `export KADI_BROKER_URL_REMOTE="wss://broker.dadavidtseng.com/kadi"`
   - `export MODEL_MANAGER_BASE_URL="https://model-manager.example"`
   - `export MODEL_MANAGER_API_KEY="mm-xxxx"`
   - or `export ANTHROPIC_API_KEY="anthropic-xxxx"`
2. Run `npm run preflight` to ensure deps installed.
3. For development: `npm run dev`
4. For production: `npm run build` then deploy using `kadi` or your container tooling (agent.json includes a deploy example that retrieves secrets via `kadi secret` before starting).

Notes and troubleshooting
- If no provider credentials are supplied the agent will run but only perform heuristic validation. Check logs for the LLM-provider warning and the heuristic fallback behavior.
- Broker configuration is required in `config.toml` (at least one of `[broker.local]` or `[broker.remote]`).
- Vault secrets (anthropic, model-manager, arcadedb) can be delivered by `kadi secret` or other vault integrations — environment variables override vault values.
- Ensure `agent.json` brokers field or `config.toml` broker entries match your running KADI broker or set `KADI_BROKER_URL_LOCAL` / `KADI_BROKER_URL_REMOTE` accordingly.

Contact / Contributing
----------------------
- See repository root for contribution guidelines and code of conduct.
- For issues specific to agent behavior, include runtime logs and the environment variable / config.toml used to start the agent.

License
-------
- See repository root for license details.

## Quick Start

```bash
cd agent-qa
npm install
kadi install
kadi run start
```

## Tools

See "Tools" table above — broker, model-manager, anthropic, validation handler, memory service, arcadedb.

## Configuration

### agent.json

| Field | Value |
|-------|-------|
| **Version** | 0.3.5 |
| **Type** | agent |

### Abilities

- `secret-ability`
- `ability-file-local`
- `ability-eval`
- `ability-vision`
- `ability-file-remote`
- `ability-log` (^0.1.5)

### Brokers

- **remote**: `wss://broker.dadavidtseng.com/kadi`
- Local broker configuration is possible via `config.toml` and can be overridden with `KADI_BROKER_URL_LOCAL` / `KADI_BROKER_URL_REMOTE`.

## Architecture

See "Architecture" section above for the high-level design and data flow.

## Development

```bash
npm install
npm run build
kadi run start
```

---