# Hermes Agent on Render

Deploy [Hermes Agent](https://github.com/NousResearch/hermes-agent) (the self-improving AI agent from Nous Research) on Render as a single Docker web service with a persistent disk for skills, sessions, and memories.

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy-template/api/github/start?template_repo=hermes-render)

This template pins a specific Hermes release for reproducible deploys, keeps all state on a persistent disk so upgrades are non-destructive, and exposes the Hermes web dashboard at the service URL for browser-based setup.

## Architecture

```
                            ┌──────────────────────────────────────────┐
                            │  Render web service (Docker, plan: standard)
                            │                                          │
   you / external clients   │  ┌────────────────────────────────────┐  │
   ─────────HTTPS──────────►│  │  hermes dashboard (port 10000)     │  │
                            │  │  - /api/status (healthcheck)        │  │
                            │  │  - browser UI: config / keys / logs│  │
                            │  └────────────────────────────────────┘  │
                            │                  │                       │
                            │  ┌────────────────────────────────────┐  │
   Telegram / Discord /  ◄──┤  │  hermes gateway run (foreground)   │  │
   Slack / etc. (outbound)  │  │  - long-poll connections to chat    │  │
                            │  │    platforms                        │  │
                            │  │  - spawns subagents per task        │  │
                            │  └────────────────────────────────────┘  │
                            │                  │                       │
                            │                  ▼                       │
                            │  ┌────────────────────────────────────┐  │
                            │  │  /opt/data (persistent disk, 5 GB) │  │
                            │  │  .env, config.yaml, sessions/,    │  │
                            │  │  skills/, memories/, logs/         │  │
                            │  └────────────────────────────────────┘  │
                            └──────────────────────────────────────────┘
```

A single container runs both Hermes processes. The dashboard ([upstream docs](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/web-dashboard.md)) is a side-process that the upstream entrypoint backgrounds whenever `HERMES_DASHBOARD=1` is set; the gateway is the foreground PID. They share `/opt/data` and a PID namespace, which is required for the dashboard's gateway-liveness checks.

The disk holds everything that should survive a redeploy: API keys (`.env`), config (`config.yaml`), the FTS5 session database, installed skills, Honcho user models, agent memories, cron job definitions, and logs.

## Prerequisites

You need at least one of the following:

- **An LLM provider API key.** [OpenRouter](https://openrouter.ai/keys) is the easiest because it routes to most providers behind a single key. Direct keys for Anthropic, OpenAI, Google, or Hugging Face also work.
- **A Render account.** The free plan can't run this image; you need at least the `standard` plan ($25/month at time of writing) for memory headroom.

Optional, depending on which channels you want Hermes to listen on:

- **Telegram bot token** from [@BotFather](https://t.me/BotFather), plus your Telegram user ID from [@userinfobot](https://t.me/userinfobot).
- **Discord bot token** from [discord.com/developers/applications](https://discord.com/developers/applications) (enable the Message Content Intent).
- **Slack bot + app-level tokens** from [api.slack.com/apps](https://api.slack.com/apps) (Socket Mode requires both `xoxb-...` and `xapp-...`).

You don't need any of these to deploy. You can fill them in via the Render Dashboard after the service is up.

## Deploy

### Option 1: Deploy button

1. Click the **Deploy to Render** button above.
2. Pick a workspace and a service name.
3. Render reads `render.yaml`, generates a value for `HERMES_GATEWAY_TOKEN`, and creates the service. All other env vars start blank.
4. The first deploy pulls the upstream Docker image (~2.6 GB compressed). Expect ~3 to 5 minutes for the image pull, then ~1 minute for the gateway to boot.

### Option 2: Manual Blueprint sync

1. Fork [render-examples/hermes-render](https://github.com/render-examples/hermes-render).
2. In the Render Dashboard, go to **Blueprints** → **New Blueprint Instance** and point at your fork.
3. Confirm and apply.

### Lock down the URL before configuring

The Hermes dashboard has no built-in authentication. Anyone who knows the service URL can read and write your API keys. Before you visit the dashboard for the first time, restrict access via Render's per-service IP allowlist:

1. In the Render Dashboard, open the `hermes` service.
2. Go to **Settings** → **Networking** → **IP Allow**.
3. Add your current public IP (Render shows it inline).
4. Save.

You can remove the allowlist later if you want a fully public dashboard, but read the **Security** section first.

## Post-deploy setup

Once the service is healthy (the **Events** tab shows "Deploy live"), open the URL Render assigned (it ends in `.onrender.com`). You'll see the Hermes dashboard.

The Blueprint deliberately keeps the env-var surface tiny. All provider keys, tool keys, and chat platform tokens are set from the dashboard, not from `render.yaml`. The dashboard writes everything to `/opt/data/.env`, which lives on the persistent disk and survives redeploys.

Walk through these tabs in order:

1. **API Keys**. Paste a key for at least one LLM provider. Pick one:
   - `OPENROUTER_API_KEY` from [openrouter.ai/keys](https://openrouter.ai/keys) routes to most providers behind a single key
   - `ANTHROPIC_API_KEY` from [console.anthropic.com](https://console.anthropic.com) for Claude models direct
   - `OPENAI_API_KEY`, `GOOGLE_API_KEY`, `HF_TOKEN`, etc. for the others
2. **Config**. Set the `model` field at the top of the list. The upstream image's default is `anthropic/claude-opus-4.6`, which works as soon as you've set `ANTHROPIC_API_KEY`. Otherwise pick a model your provider supports (for example, `anthropic/claude-sonnet-4.6` for Anthropic, or any OpenRouter model ID like `openai/gpt-5.5`).
3. **Status**. Confirm the gateway is running and the model is reachable. The "Connected platforms" list will be empty until you add a chat platform.
4. **API Keys** again, optionally. If you want a chat gateway, add the matching tokens: `TELEGRAM_BOT_TOKEN`, `DISCORD_BOT_TOKEN`, `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN`, etc. Use the **Restart gateway** button on the Status tab so the new tokens are picked up.

If you'd rather set keys from the Render Dashboard's **Environment** tab (handy for CI or secrets-manager workflows), that path also works: Render env vars override `/opt/data/.env` at process start. Pick one path and stick with it to avoid drift.

### Where the "gateway token" fits

The Blueprint generates a `HERMES_GATEWAY_TOKEN` for you. Today, upstream Hermes doesn't read this variable directly at runtime: it's a placeholder for the OpenAI-compatible API server's bearer key. If you opt into the API server (set `API_SERVER_ENABLED=true` from the dashboard's **API Keys** tab, then paste this token into `API_SERVER_KEY`), external HTTP clients can authenticate against `/v1/chat/completions` using `Authorization: Bearer <that value>`.

## Chatting with the agent

The simplest way to talk to your deployed Hermes is the dashboard's **Chat** tab. The Blueprint sets `HERMES_DASHBOARD_TUI=1`, which makes the upstream dashboard expose the full TUI in the browser over a server-side PTY plus xterm.js. Slash commands, model picker, tool-call cards, streaming, sessions: everything works the same as a local terminal.

If you'd rather stay on the command line, two paths work, both because the in-container `hermes` is the same binary as the local CLI:

- **One-shot prompts via Render Shell or SSH.** The browser shell on Render does not allocate a TTY for `runtime: image` services. The interactive REPL (`hermes` with no args) will print a banner and quit immediately with `Warning: Input is not a terminal (fd=0)`. Use the non-interactive form instead:

  ```bash
  /opt/hermes/.venv/bin/hermes chat -q "summarize today's logs"
  ```

  This runs one turn, prints the result, and exits cleanly. You can chain it with `--resume <session-id>` to continue an existing conversation.

- **Real terminal via the Render CLI.** From your local machine:

  ```bash
  render ssh <service-id>
  /opt/hermes/.venv/bin/hermes
  ```

  `render ssh` allocates a PTY, so the interactive REPL works.

The chat tab in the dashboard is still the cleanest UX. Use the CLI fallbacks when you're scripting or already in a terminal context.

## Cost expectations

Costs assume Render's published prices in May 2026 and don't include data egress, which is unmetered for typical Hermes traffic.

| Component                     | Plan                              | Cost            |
|-------------------------------|-----------------------------------|-----------------|
| Web service (`runtime: image`) | `standard` (2 GB / 1 CPU)         | $25/month       |
| Persistent disk (`/opt/data`)  | 5 GB SSD                          | $1.25/month     |
| **Subtotal (this template)**   |                                   | **$26.25/month**|

If you do a lot of Playwright browsing or run several subagents in parallel, bump the plan to `pro` (4 GB / 2 CPU, $85/month). The starter plan (512 MB) cannot hold the Hermes image and is not supported.

LLM costs are separate and depend entirely on your provider and usage. OpenRouter and Anthropic both report usage in their respective dashboards; Hermes also surfaces per-model usage on its **Analytics** page.

## Updating Hermes

Bump the image tag in `render.yaml`:

```yaml
image:
  url: docker.io/nousresearch/hermes-agent:v2026.5.7  # change this
```

Commit and push. Render won't auto-deploy (the Blueprint sets `autoDeployTrigger: off`); trigger a manual deploy from the Dashboard or via the [Render CLI](https://render.com/docs/cli):

```bash
render deploys create <service-id>
```

Your `/opt/data` disk is untouched across image upgrades. The upstream entrypoint runs a manifest-based `skills_sync.py` on each boot, which preserves edits to bundled skills.

Hermes ships fast: roughly weekly tagged releases, each with around 180 commits. Check [the upstream releases page](https://github.com/NousResearch/hermes-agent/releases) before bumping.

## Troubleshooting

### Logs

Render keeps logs in the **Logs** tab of your service. Filter by stream:

- The dashboard side-process prefixes its lines with `[dashboard]`.
- Gateway and agent logs are unprefixed.
- For deeper inspection, log files also live on disk at `/opt/data/logs/` (`agent.log`, `errors.log`, `gateway.log`).

You can tail them from the dashboard's **Logs** tab too, or via SSH (next section).

### Shell access

Render gives you SSH into the container. From the service's overview page, click **Shell** (browser PTY) or copy the SSH command from **Settings**.

```bash
# Inspect the data volume.
ls /opt/data
cat /opt/data/.env

# Run the Hermes CLI directly.
/opt/hermes/.venv/bin/hermes status
/opt/hermes/.venv/bin/hermes config get model.default
```

The container runs as the `hermes` user (UID 10000), not root.

### Service won't start

Check the **Events** tab for the deploy that failed, then the **Logs** tab around that timestamp.

| Symptom                                              | Likely cause                                                                 |
|------------------------------------------------------|------------------------------------------------------------------------------|
| `Refusing to start: binding to 0.0.0.0 requires API_SERVER_KEY` | You set `API_SERVER_ENABLED=true` and `API_SERVER_HOST=0.0.0.0` without an `API_SERVER_KEY`. Set the key or flip back to `127.0.0.1`. |
| Health check fails on `/api/status`                  | `HERMES_DASHBOARD` is unset or the dashboard crashed. Check `[dashboard]` lines for a Python traceback. |
| Container OOM-killed                                 | Bump plan to `pro`. Playwright/Chromium is the usual culprit.                 |
| `Permission denied` on `/opt/data/...`               | The disk was attached after a deploy that ran as a different UID. Restart the service; the entrypoint chowns `/opt/data` on boot when run as root. |
| `Warning: Input is not a terminal (fd=0)` then `Goodbye!` when running `hermes` | Render's browser shell pipes stdin instead of allocating a PTY. Chat from the dashboard's **Chat** tab, or use `hermes chat -q "..."`, or `render ssh <service-id>` from a local terminal. |
| `Goodbye! ⚕` in the deploy logs followed by 502s on the URL | The container is missing the `dockerCommand` override. On Render, `dockerCommand` replaces both the image's `CMD` and its `ENTRYPOINT`, so you need to invoke the upstream entrypoint chain explicitly: `/usr/bin/tini -g -- /opt/hermes/docker/entrypoint.sh gateway run`. Without it, the entrypoint launches the interactive REPL as PID 1 and exits on Render's non-TTY stdin. |
| `Refusing to run the Hermes gateway as root` | Same root cause as above. `dockerCommand` bypassed the entrypoint, so `gosu` never dropped privileges. The full entrypoint chain in `dockerCommand` fixes both. |
| `tirith security scanner enabled but not available`  | Harmless. Tirith is an optional Rust-based command scanner; without it, Hermes uses pattern matching. Ignore unless you specifically want native scanning. |

### Changing env vars

Set, change, or delete env vars under the service's **Environment** tab. Render restarts the container after a save. Hermes also exposes a `/reload` slash command for in-session reloads if you've already started chatting from the CLI; it's not relevant for the gateway, which restarts cleanly.

### Forcing a clean rebuild

If the Hermes data directory gets into a bad state (corrupt session DB, partial skill install), wipe it:

1. SSH in.
2. `mv /opt/data /opt/data.bak && exit`.
3. Restart the service from the Render Dashboard. The entrypoint recreates the directory tree and reseeds defaults.

Or restore the most recent automatic disk snapshot from the **Disks** page.

## Security

Hermes' web dashboard has no authentication of its own. The session token it injects into the SPA HTML is regenerated on every server start; anyone who can reach the URL gets it. Upstream's own docs are blunt about this:

> If you bind to `0.0.0.0`, anyone on your network can view and modify your credentials. The dashboard has no authentication of its own.

This template binds the dashboard to `0.0.0.0:10000` because Render web services need a public listener. The recommended posture is therefore:

1. Put the service behind Render's IP allowlist (above) before visiting it the first time.
2. Add a reverse proxy in front (Cloudflare Access, Tailscale, basic-auth via a Render private service sidecar) if you want long-term remote access.
3. Or keep the IP allowlist permanent and visit the dashboard only from trusted networks.

The OpenAI-compatible API server (`API_SERVER_ENABLED=true`) is a separate concern: that one is properly authenticated by `API_SERVER_KEY` and is safe to expose with a long random key, but it isn't routed to the public URL by this Blueprint.

For broader Hermes security guidance see the [upstream security doc](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/security.md).

## What this template does and doesn't do

What it does:

- Pins a specific upstream image (`v2026.5.7`) for reproducible deploys.
- Runs the Hermes gateway and dashboard inside one container, the way upstream supports.
- Mounts a persistent disk at the upstream-default `HERMES_HOME` path.
- Generates a `HERMES_GATEWAY_TOKEN` and lists every env var with `sync: false` so secrets never sync from the repo.
- Sets a healthcheck that probes the dashboard.

What it deliberately doesn't do:

- It doesn't try to add authentication on top of the dashboard. Use the IP allowlist or a reverse proxy.
- It doesn't enable the OpenAI-compatible API server. Flip `API_SERVER_ENABLED=true` and supply `API_SERVER_KEY` if you need it.
- It doesn't ship a default model. Hermes' upstream default is set in `config.yaml`, which lives on disk and is owner-configurable from the dashboard.
- It doesn't configure browser automation tweaks (`--shm-size`, GPU access). Those need an instance type with more RAM, not extra Render config.

## License

This template is MIT licensed (see [`LICENSE`](./LICENSE)). Hermes Agent itself is also MIT licensed; see [the upstream LICENSE](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE).
