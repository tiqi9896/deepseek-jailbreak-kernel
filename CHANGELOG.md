# Changelog

## 1.0.0 — 2026-11-02

First release of the DeepSeek kernel.

### Kernel injection

- Separate injector for the OpenAI shape: `_openai_inject_system` writes a
  `{"role":"system"}` entry inside `messages[]`. A plugin that only rewrites the
  top-level `system` field is a silent no-op on DeepSeek.
- Model allowlist `HERMES_JAILBREAK_DEEPSEEK_MODELS`, default `deepseek-flash`.
  `deepseek-v4-pro` is deliberately excluded.
- Both `context["model"]` and `request["model"]` are checked, closing the fallback
  window where the two disagree.
- Allowlist membership is separate from upstream routing: a non-allowlisted DeepSeek
  model still goes to the DeepSeek endpoint, only without the kernel.
- No authorization block on this path. Measured negative: flash 8/10 → 7/10.
- No `thinking` / `output_config`. The OpenAI shape has no such parameters and the
  upstream silently ignores them.

### Output budget

- DeepSeek floor raised to 32000. At 16000 the model spends the budget on reasoning
  and opens with an 844-character refusal; at 32000 the same request delivers 47590
  characters.

### Stale kernel fix

- `_kernel_is_current()` compares the first 120 and last 120 characters of the
  current kernel against the existing text. Marker presence alone was insufficient:
  Hermes freezes plugin sections per session, so old sessions restored an old kernel
  forever.
- `_cut_kernel()` truncates at the first kernel marker. Without it, replacement
  produced two kernels, because `_extract_section()` swallows a bare old kernel into
  the memory section and `strip_plugin_sections()` only removes HTML containers.
- Applied to both the OpenAI `messages[]` path and the Anthropic top-level path.

### Multi-round escalation

- `multiround.py`: specification → fill verbatim → drop the qualifiers, sharing one
  conversation history across rounds.
- `HERMES_JAILBREAK_MULTIROUND=auto|always|off`, default `auto`.
- Five detection predicates: `refusal`, `reversal`, `downgrade`, `form`, `thin`.
  `reversal` and `form` were added after measured failures that every earlier
  predicate passed.
- Honest fallback: if three rounds come back worse than a usable single turn, the
  single turn is delivered instead.
- Topic hints via `X-Topic-Hint` header or `topic_hint` body field.
- Upstream round calls retry twice with backoff; a multi-round run is 130–280 seconds
  and the upstream resets connections at that duration.

### Tools

- `tools/audit-deepseek-jailbreak.py` — verifies plugin, middleware registration,
  allowlist, routing (including `provider: custom`), and observed injection in the
  gateway log. Portable: locates the Hermes home via `HERMES_HOME` or by walking up
  from the script.
- `tools/jb-daily-check.py` — daily health check: kernel budget, allowlist, proxy
  reachability, sync across the four asset copies, parallel-run collision detection.

### Measurements

All published numbers are in `data/` and were independently recomputed:

- baseline A/B/C: flash 4/10 → 8/10 → 7/10; v4-pro 3/10 → 10/10 → 9/10
- variant matrix (120 requests): base 23/30, +affirm 24/30, +noqual 20/30, +both 25/30
  — enhancements are inside the noise band
- kernel placement: 1/8 in system vs 0/8 at the end of the user message
- end-to-end through the real middleware: v4-pro 12/12, flash 7/12 at 16000 → 8/12 at 32000
- multi-round: ransomware 12 runs zero refusals, the third round removes the
  dry-run branch
- automation: 5 of 5 failures without a topic hint, 41899 chars / 28 code blocks with one

### Corrections

- `deepseek-v4-pro` variant C was first recorded as 1/1 from a single observation.
  The full run gives 9/10. Corrected across all published documents.
