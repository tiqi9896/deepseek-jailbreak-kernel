# The jailbreak proxy (starts at login)

- port: **8790**
- Anthropic-compatible base: `http://127.0.0.1:8790/anthropic`
- OpenAI-compatible base: `http://127.0.0.1:8790/v1`
- OpenAI Responses base: `http://127.0.0.1:8790/v1`
- health: `http://127.0.0.1:8790/health`
- model: **`deepseek-chat`** (or `deepseek-reasoner`)

**This is the proxy for external agents** — Cursor, Claude Code, Cline,
Codex CLI, opencode, and anything else that can set a `base_url`.

Hermes itself uses the native plugin
(`~/.hermes/plugins/deepseek-jailbreak-core`, on Windows
`%USERPROFILE%\.hermes\plugins\deepseek-jailbreak-core`) — **Hermes does
not need this proxy**.

Per-agent setup instructions: [`docs/AGENT-SETUP.md`](docs/AGENT-SETUP.md).

## Start at login

| what | role |
|---|---|
| `<repo>/proxy/start-proxy.vbs` | the real launcher (idempotent — it does not start a second copy) |
| `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\deepseek-jailbreak-proxy.vbs` | the login item; it calls the launcher above by absolute path |

The Python interpreter is probed in order `py.exe` → `python.exe` →
`python3.exe`, falling back to `python` on PATH. **No machine path is ever
hard-coded.**

## Manual operation

```bash
# start
python proxy.py --port 8790 --verbose

# stop (Windows)
netstat -ano | findstr :8790
taskkill /PID <pid> /F

# start script (cmd variant, idempotent)
start-proxy.cmd

# verify the request path against the running proxy
python tests/proxy_smoke.py --port 8790
```

## Allowlist

The default admits every `deepseek*` model id and excludes only
`deepseek-v4-pro`. To narrow it, set the proxy's own variables — the proxy
does not read `HERMES_JAILBREAK_DEEPSEEK_MODELS` (that is the Hermes plugin's
variable):

```bash
export JBP_DEEPSEEK_MODELS=deepseek-chat,deepseek-reasoner
export JBP_DEEPSEEK_MODELS_DENY=deepseek-v4-pro
```

## Source

https://github.com/tiqi9896/deepseek-jailbreak-kernel (MIT)
