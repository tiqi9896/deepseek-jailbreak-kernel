# Changelog

---

## 1.1.1 — 2026-11-07

**Two protocol bugs that made Claude Code and Codex CLI unusable, and one
factual correction to the v1.1.0 notes.**

### The protocol shape has to follow the client

`provider_of()` always returns `"deepseek"` (this proxy serves DeepSeek only),
so the old routing rewrote **every** DeepSeek request into the OpenAI shape
and sent it to `/chat/completions` — then forwarded the OpenAI-shaped reply
verbatim:

```
POST /anthropic/v1/messages  ->  https://api.deepseek.com/chat/completions
POST /v1/responses           ->  https://api.deepseek.com/chat/completions
```

Consequences:

* **Claude Code** (Anthropic Messages) received `{"choices":[...]}` instead of
  `{"content":[{"type":"text",...}]}` — it cannot parse the reply.
* **Codex CLI** (OpenAI Responses) received a Chat Completions body, and its
  `input` array was never translated at all.
* The Anthropic request's top-level `system` was being sent to an OpenAI
  endpoint.

DeepSeek publishes **native endpoints in all three shapes** (recorded in this
machine's codex model catalogue, verified against the live API on 2026-09-20):

| client protocol | DeepSeek native endpoint |
|---|---|
| OpenAI Chat Completions | `https://api.deepseek.com` + `/chat/completions` |
| Anthropic Messages | `https://api.deepseek.com/anthropic` + `/v1/messages` |
| OpenAI Responses | `https://api.deepseek.com/v1` + `/responses` |

Each protocol now goes to its own same-shaped endpoint, so the reply comes
back in the same shape and **no shape conversion is needed at all**. Two new
options: `--deepseek-anthropic-upstream`, `--deepseek-responses-upstream`.

Each shape also gets its own injector, because `system` lives in a different
place in each one — and DeepSeek never gets the authorization block (measured
negative there: `deepseek-flash` 8/10 → 7/10):

| shape | where the kernel goes |
|---|---|
| OpenAI Chat Completions | a `role:"system"` message in `messages[]` |
| Anthropic Messages | the top-level `system` field (a `role:"system"` message would be rejected) |
| OpenAI Responses | the top-level `instructions` field |

### Multi-round escalation was dropping the kernel

`_maybe_multiround` read the kernel back out of `body["system"]` — a field
that only exists in the Anthropic shape. On the OpenAI shape the kernel lives
in a `role:"system"` message, and on the Responses shape in `instructions`, so
the kernel came back **empty** and the escalated rounds were sent **without
it** (`payload["messages"]` was overwritten with the bare history).

The kernel is now recovered per shape (`_kernel_of`) and re-attached to every
round (`_history_for` also converts the history into the target shape).

### Correction to the v1.1.0 notes

The v1.1.0 notes claimed `deepseek-flash` was "a Hermes-internal tier alias
that does not exist on DeepSeek's public API". **That was wrong** — it is a
real DeepSeek model id, as this repo's own measurement data and its own
`/v1/models` listing show. The defect was that the allowlist named **exactly
one** id, not that the id was fake. Every model id in the examples has been
corrected, and the root-cause text rewritten.

### Verification

* `proxy/proxy.py --self-test` — **125 checks, 0 failures** (was 116), including
  the new assertions that the Anthropic request lands on `/v1/messages` with
  the kernel in the top-level `system` and no `role:"system"` message, and that
  the Responses request lands on `/responses` with the kernel in `instructions`.
* Reverting the routing makes those assertions fail (2 FAIL) — the regression
  test has teeth.

## 1.1.0 — 2026-11-07

**This release fixes the reason the kernel appeared to do nothing.**
Reports said the repo was handed to a dozen different agents (DSH, Hermes,
openclaw, …) and none of them got the kernel to take effect. The cause was
measured, not guessed.

### Root cause — the default allowlist named exactly one model id

The default allowlist named a single model id.

DeepSeek's real model ids are `deepseek-flash` and `deepseek-v4-pro` — this
repo's own measurement data (`data/DS-E2E-FLASH-MT32000-20261101.json`, real
requests on 2026-11-01) and its own `/v1/models` listing both use them. **The
id was never fake; the allowlist was simply too narrow.**

So for any agent configured with a *different* DeepSeek model id, the proxy
still routed the request to DeepSeek — the name contains `deepseek` — but
**never injected the kernel** and never raised the output floor. Nothing
errored: `/health` was 200, the self-test was green, the answer came back
normally. The behaviour was identical to installing nothing.

* The default now admits **every `deepseek*` model id**, so a newly released
  DeepSeek model works with no config change, and `deepseek-v4-pro` stays
  excluded (the original design decision, still deliberate).
* Setting an allowlist still narrows it, exactly as before.
* New deny list: `HERMES_JAILBREAK_DEEPSEEK_MODELS_DENY` /
  `JBP_DEEPSEEK_MODELS_DENY`. The deny list wins over the allow list.

### Second root cause — the proxy inherited Hermes' allowlist

`_env_file()` prefers `HERMES_HOME/.env`. On a machine that also runs Hermes,
the proxy for external agents therefore **inherited**
`HERMES_JAILBREAK_DEEPSEEK_MODELS=deepseek-flash` and the problem returned.

The proxy and the plugin now use **separate variables**:

| component | allowlist | deny list |
|---|---|---|
| Hermes plugin | `HERMES_JAILBREAK_DEEPSEEK_MODELS` | `HERMES_JAILBREAK_DEEPSEEK_MODELS_DENY` |
| Proxy for external agents | `JBP_DEEPSEEK_MODELS` | `JBP_DEEPSEEK_MODELS_DENY` |

### Fixed — `/jailbreak` raised NameError

`set_effort` / `effective_effort` were called by the `/jailbreak` command and
by `status()` but **were never defined**, so any `/jailbreak` invocation died
with `NameError: name 'effective_effort' is not defined`. DeepSeek's OpenAI
shape has no effort parameter at all — thinking tier is chosen by model id, and
what actually matters for DeepSeek is the 32000 output floor — so the two
functions are now documented no-ops that record and echo the setting.

### Fixed — the audit tool crashed when the plugin was missing

`tools/audit-deepseek-jailbreak.py` raised
`UnboundLocalError: cannot access local variable 'plug'` exactly when the
plugin was not installed — i.e. when you most need the audit. It now reports a
clean failure instead of a traceback, and checks the **real public model ids**
rather than the alias.

### Fixed — the self-test never made a real request

The self-test only exercised the injection functions and never checked
model-name routing, which is why the defect above was invisible. It now runs a
**real HTTP round trip** against a local stub upstream — no network, no API key
— across all three protocol shapes, and asserts:

* a response comes back (not a dropped connection)
* the kernel reaches the upstream, with the 32000 output floor
* the auth headers are forwarded
* a missing key yields 401, not a crash
* **`deepseek-flash` and `deepseek-next` do get the kernel**
* a denied model (`deepseek-v4-pro`) does not

Reverting the allowlist default makes five of these assertions fail — the
regression test has teeth.

`--self-test` now ends with a bilingual `ALL PASSED / 结果: 全部通过` line,
printed only on real success.

### Fixed — machine-specific paths and missing installers

* `watch-proxy.py`, `watch-proxy.cmd`, `start-proxy.cmd`, `start-proxy.vbs` no
  longer hard-code a Python path; they probe `py.exe` / `python.exe` /
  `python3.exe`.
* `AGENTS.md` and `proxy/README.md` no longer contain this machine's paths.
* `AGENTS.md` Step 1 contained the literal placeholder
  `https://github.com/<owner>/deepseek-jailbreak-kernel.git`, which cannot be
  cloned. Replaced with the real URL.
* `AGENTS.md` Gate 3 promised `133 通过`, but the plugin has 22 assertions — a
  correct run looked like a failure. The count now matches reality.
* `install.ps1` is saved as UTF-8 **with BOM**: Windows PowerShell 5.1 reads
  `.ps1` files in the local ANSI codepage otherwise, which mangles the
  non-ASCII characters and breaks the parser (`ParserError`).
* `install.sh` is POSIX-portable (Linux / macOS / WSL / Git Bash) and no longer
  dies on `set -u` when `LOCALAPPDATA` is unset.
* `install.sh` / `install.ps1` copy the plugin into
  `$HERMES_HOME/plugins/deepseek-jailbreak-core`, run **both** self-tests and
  fail loudly if either does not pass.
* New `tests/proxy_smoke.py` — sends a real request to a **running** proxy and
  tells apart the failures that look identical to the user: connection refused
  (not running), `RemoteDisconnected` (proxy crashed handling the request),
  401 (key missing), 5xx (upstream/proxy error).

### Verification

* `plugin/selftest.py` — 22 assertions, 0 failures.
* `proxy/proxy.py --self-test` — 125 checks, 0 failures, including the real
  HTTP round trip and the model-name routing assertions.
* `tools/audit-deepseek-jailbreak.py` — 13 checks, 0 failures; and it no longer
  crashes when the plugin is absent.
* `install.sh` verified under Git Bash (Windows) and under WSL (Linux).
* `install.ps1` parses under Windows PowerShell 5.1 and PowerShell 7.

