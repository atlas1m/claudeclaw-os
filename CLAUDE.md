# ClaudeClaw — Project Notes

This file is auto-loaded by the Claude Code SDK whenever an agent runs from this repo as cwd. Personal Boss persona lives in `~/.claudeclaw/CLAUDE.md` (the SDK also loads that as `systemPrompt`). Project-agent personas live in `~/.claudeclaw/agents/<id>/CLAUDE.md`. **Keep this file scoped to engineering facts about the codebase, not personality, not orchestration decisions.**

## What this codebase is

ClaudeClaw is a Telegram bridge that runs the `claude` CLI on this machine and pipes results back to Telegram. The local fork `atlas1m/claudeclaw-os` diverges from upstream `earlyaidopters/claudeclaw-os` for personal customisations.

## Our agent topology

There is exactly one default-installed agent: `main` (alias: Boss). It runs under `launchctl` as `com.claudeclaw.app`.

There are NO `research`, `comms`, `content`, `ops`, or other specialist agents. The directories `agents/comms/`, `agents/content/`, `agents/ops/`, `agents/research/` were removed in this fork. Only `agents/_template/` remains, used by `npm run agent:create` as the blank scaffold for new project agents.

When `getAvailableAgents()` returns an empty list at runtime, that is correct — agents are created on demand, one per project.

## How to rebuild

```bash
cd ~/.claudeclaw-app
env -u VIRTUAL_ENV PATH="/Users/maximejean/.nvm/versions/node/v24.11.1/bin:/opt/homebrew/bin:/usr/bin:/bin" npm run build
```

The `env -u VIRTUAL_ENV PATH=...` is mandatory: the user's shell auto-activates a Skipr Python venv whose `bin/` directory is at `/Volumes/Extreme Pro/Jarvis/Skipr/.venv/bin` (path with spaces) which breaks `node-gyp` in `better-sqlite3`. Always strip those entries before npm operations.

## How to restart Boss

```bash
launchctl kickstart -k gui/$(id -u)/com.claudeclaw.app
```

Verify with `launchctl list | grep claudeclaw` (status `0`) and `tail -10 /tmp/claudeclaw.log`.

## How to spawn a project agent

`npm run agent:create` is interactive. The recipe is documented in `~/.claudeclaw/CLAUDE.md` (Boss's instructions). Project agents live under `~/.claudeclaw/agents/<id>/` and need their own launchd plist at `~/Library/LaunchAgents/com.claudeclaw.<id>.plist` to run persistently.

## Sending files via Telegram

Include markers in agent responses:
- `[SEND_FILE:/abs/path/to/file]` for documents
- `[SEND_PHOTO:/abs/path/to/image]` for inline photos
- `[SEND_FILE:/abs/path|caption]` for documents with captions

Always absolute paths. Max 50 MB per Telegram limits.

## Special slash commands handled in code (do NOT redefine)

The set in `src/bot.ts:OWN_COMMANDS` defines bot-owned slash commands (`/start`, `/help`, `/voice`, `/model`, `/agents`, `/delegate`, `/lock`, `/status`, etc.). Any other slash command goes through to Claude Code as a skill invocation. gstack contributes about forty more, all available via `/skill-name`.

## Special user phrases handled in CLAUDE.md (Boss's `~/.claudeclaw/CLAUDE.md`)

`convolife` — context window report.
`checkpoint` — save TLDR to memory before `/newchat`.

## launchd gotcha

The macOS launchd silently exits with code 78 (`EX_CONFIG`) when `StandardOutPath` or `StandardErrorPath` contain spaces. `WorkingDirectory` accepts spaces fine. Keep agent log paths under `/tmp/` or `~/Library/Logs/`. The currently installed plist at `~/Library/LaunchAgents/com.claudeclaw.app.plist` writes to `/tmp/claudeclaw.log`.

## Voice mode

Voice replies for Maxime's chat are persistent across restarts via `VOICE_MODE_DEFAULT=on` in `.env`. This is a fork-local feature: `src/config.ts` exports `VOICE_MODE_DEFAULT`, `src/bot.ts` seeds `voiceEnabledChats` with `ALLOWED_CHAT_ID` at startup.

## Voice ID

ElevenLabs cloned voice ID `LAqExCMuLu67UPvohj2g`. API key in `.env` as `ELEVENLABS_API_KEY`. Subscription is the $6/mo Creator tier.

## Anthropic routing

`ANTHROPIC_BASE_URL=http://100.64.95.60:9090` in `.env` routes through Maxime's claude-proxy on the VPS via Tailscale. Failover across four Claude Max accounts. See `/Volumes/Extreme Pro/Jarvis/claude-proxy/HANDOVER.md` for ops details.

## War Room

Lives in `warroom/`. Python venv at `warroom/.venv/`. Requires `pipecat-ai==0.0.108` plus `google-genai` for Gemini Live. The dashboard exposes `/warroom` for the boardroom UI; the bridge code in `src/agent-voice-bridge.ts` connects each agent's Claude session to the room via WebSocket.

`WARROOM_ENABLED=true` in `.env` to start the subprocess at boot. The launcher in `src/index.ts` does pre-flight `python -c 'import pipecat'` checks and limits respawns to three on crash.
