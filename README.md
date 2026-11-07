# Openly Computer Use

Openly is a local platform for building computer-use workflows. Use the guided UI to create steps, choose a configured VM provider, run workflows, inspect logs and recordings, and schedule future runs.

The project is split into three Git submodules:

- [`platform`](platform/) is the local UI and calls the backend.
- [`backend`](backend/) owns PostgreSQL, workflow versions, orchestration, schedules, run history, logs, and recordings.
- [`agents`](agents/) executes workflows in provider sessions and reports results to the backend. It has no database.

Today the supported provider/target combinations are OnKernel Browser and Daytona Windows Computer. Only configured providers are advertised. Other provider drafts are not enabled. Daytona's native recording has been unreliable in previous tests.

## Clone

```sh
git clone --recurse-submodules https://github.com/openlybuilt/computer-use.git
cd computer-use
```

For an existing clone, run `git submodule update --init --recursive`. Each submodule is a separate repository pinned by this parent repository.

## Prerequisites

- Node.js 24 and pnpm 11.25.0.
- A local PostgreSQL database, for example one named `openly`.
- An OpenAI API key and model name.
- A key for at least one supported provider. Real runs can incur provider and model usage charges.

The app currently operates as a local workspace without end-user sign-in. Backend and agents use separate service-to-service tokens. Keep tokens, API keys, placeholder values, recordings, and live-view URLs private.

## Configure

Create two distinct random URL-safe service tokens, each 32–256 characters, and one stable base64-encoded 32-byte placeholder encryption key. For example, run these locally and save the values securely:

```sh
node -e "console.log(require('node:crypto').randomBytes(32).toString('base64url'))"
node -e "console.log(require('node:crypto').randomBytes(32).toString('base64url'))"
node -e "console.log(require('node:crypto').randomBytes(32).toString('base64'))"
```

Create `backend/.env` with your database credentials and the generated values. Replace every placeholder:

```dotenv
DATABASE_URL=postgresql://USER:PASSWORD@127.0.0.1:5432/openly
AGENTS_URL=http://127.0.0.1:3100
AGENTS_API_TOKEN=REPLACE_WITH_FIRST_TOKEN
AGENTS_CALLBACK_TOKEN=REPLACE_WITH_SECOND_TOKEN
PLACEHOLDER_ENCRYPTION_KEY=REPLACE_WITH_BASE64_KEY
```

Create `agents/.env`, using the same two tokens and your own model and provider credentials:

```dotenv
BACKEND_URL=http://127.0.0.1:3000
AGENTS_API_TOKEN=REPLACE_WITH_FIRST_TOKEN
AGENTS_CALLBACK_TOKEN=REPLACE_WITH_SECOND_TOKEN
OPENAI_API_KEY=REPLACE_WITH_OPENAI_KEY
OPENAI_MODEL=REPLACE_WITH_MODEL_NAME
ONKERNEL_API_KEY=REPLACE_IF_USING_ONKERNEL
```

Alternatively or additionally, configure `DAYTONA_API_KEY` for Windows Computer; `DAYTONA_API_URL` is optional. Remove provider lines you do not use rather than leaving placeholder values. The Daytona adapter currently requests the `windows-medium` snapshot; the account and endpoint must support that environment.

Create `platform/.env.local`:

```dotenv
BACKEND_URL=http://127.0.0.1:3000
UI_ORIGIN=http://127.0.0.1:3001
```

`UI_ORIGIN`, when set, must match the browser's exact origin. Do not commit these files. Preserve the placeholder encryption key across restarts; changing it prevents previously saved values from being decrypted.

## Start the services

Use a separate terminal for each service. The backend applies its migrations on startup, so use a dedicated local database. A migration from the former signed-in schema removes legacy signed-in workspace data.

```sh
cd backend
pnpm install --frozen-lockfile
pnpm build
node --env-file=.env dist/server.js
```

```sh
cd agents
pnpm install --frozen-lockfile
pnpm build
node --env-file=.env dist/server.js
```

```sh
cd platform
pnpm install --frozen-lockfile
pnpm dev --port 3001
```

Open `http://127.0.0.1:3001`. The UI checks that the backend and agents are available. At least one VM provider must be configured for the services to report an available execution target, even though the guide's example recording does not start a VM.

## Learn more

- [Agents service and VM provider extension guide](https://github.com/openlybuilt/agents/blob/main/docs/vm-providers.md)
- [Backend](https://github.com/openlybuilt/backend)
- [Platform](https://github.com/openlybuilt/platform)
- [Build and debug workflows as a coding agent](docs/agent-workflows.md)

## License

[MIT](LICENSE).
