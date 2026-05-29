# CLAUDE.md — hermes-agent (fork)

You're working on the **Venture-Formations fork of NousResearch/hermes-agent**.
We carry a small patch on top of upstream releases until the patch
lands upstream.

## What this repo is

A fork of [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent) — the open-source Hermes
Agent runtime. Lives at `Venture-Formations/hermes-agent`. The
Venture-Formations deployment template clones this fork (instead of
upstream) at Docker build time, controlled by an `ARG HERMES_REPO` in
`Venture-Formations/hermes-template/Dockerfile`.

## Why we forked

Two-line root cause:

1. Upstream's `chat_completions` transport never sent
   `parallel_tool_calls=True` on outgoing requests, so providers
   (OpenRouter, DeepSeek, etc.) defaulted to one tool dispatch per
   response. Production logs showed 84% of completions finishing with
   `tool_calls`, median 5+ model calls per user task.
2. The upstream `Codex` transport (`agent/transports/codex.py:153`)
   already set the flag; the patch brings `chat_completions` to parity.

## Our changes — 48 lines across 4 files

| File | What changed | Why |
|---|---|---|
| `providers/base.py` | Added `supports_parallel_tool_calls: bool = True` on `ProviderProfile` | Default-on with per-profile opt-out for any provider whose upstream rejects the field (e.g. Anthropic-via-OpenAI-compat shims) |
| `agent/transports/chat_completions.py` | Set `parallel_tool_calls=True` in both `_build_kwargs_from_profile` (profile-gated) and `build_kwargs` legacy path (gated on `is_openrouter / is_nous / is_github_models / is_nvidia_nim / is_qwen_portal`) | The actual fix |
| `agent/prompt_builder.py` | Added `PARALLEL_TOOL_GUIDANCE` constant and `PARALLEL_TOOL_MODELS = ("deepseek", "qwen", "glm")` tuple | The API permits batched dispatch but the model still chooses how many to emit; this nudges non-Gemini batch-friendly models. Gemini/Gemma already get equivalent guidance via `GOOGLE_MODEL_OPERATIONAL_GUIDANCE`. |
| `agent/system_prompt.py` | Injects `PARALLEL_TOOL_GUIDANCE` for non-Gemini batchable models, adjacent to where `GOOGLE_MODEL_OPERATIONAL_GUIDANCE` is appended | Wires the guidance into Hermes's system prompt assembly |

The pattern mirrors `agent/transports/codex.py:153` — same precedent,
same flag, just on a different transport.

## Branch topology

```
main                                  ← tracks upstream NousResearch/hermes-agent
fix/parallel-tool-calls-openrouter   ← 2 commits on top of upstream/main
                                        (used to draft the upstream PR)
fix/parallel-tool-calls-v2026.5.16   ← 2 commits on top of tag v2026.5.16
fix/parallel-tool-calls-v2026.5.28   ← 2 commits on top of tag v2026.5.28 (v0.15.0)
fix/parallel-tool-calls-v2026.5.29   ← 2 commits on top of tag v2026.5.29 (v0.15.1)
fix/parallel-tool-calls-v2026.5.29.2 ← 2 commits on top of tag v2026.5.29.2 (v0.15.2)
                                        ← CURRENTLY DEPLOYED (per template Dockerfile)
```

The deployed branch is whichever one the template Dockerfile's
`ARG HERMES_REF` points at. Check
`Venture-Formations/hermes-template/Dockerfile` to confirm.

## Upgrade procedure (when upstream ships a new tag)

1. Sync remotes:
   ```
   git fetch upstream --tags
   ```
2. Create a new fix branch from the new tag:
   ```
   git checkout -b fix/parallel-tool-calls-v<NEW_TAG> v<NEW_TAG>
   ```
3. Cherry-pick our two commits (look them up from any existing
   `fix/parallel-tool-calls-v*` branch):
   ```
   git cherry-pick <transport-commit-sha>
   git cherry-pick <prompt-commit-sha>
   ```
4. Resolve any conflicts. The most likely:
   - `agent/transports/chat_completions.py` — upstream refactored
     `build_kwargs`. Re-apply the `parallel_tool_calls=True` line at
     the matching insertion point.
   - `agent/prompt_builder.py` / `agent/system_prompt.py` —
     `GOOGLE_MODEL_OPERATIONAL_GUIDANCE` moved or was renamed.
     Re-anchor our injection to whatever its current location is.
   - `providers/base.py` — `ProviderProfile` gained or lost fields.
     Our `supports_parallel_tool_calls` field is independent; just
     re-add it.
5. Verify the patch compiles:
   ```
   python3 -m py_compile providers/base.py agent/transports/chat_completions.py agent/prompt_builder.py agent/system_prompt.py
   ```
6. Push:
   ```
   git push -u origin fix/parallel-tool-calls-v<NEW_TAG>
   ```
7. Update the deploy:
   ```
   cd ../hermes-template
   git checkout deploy/venture-formations-fork
   # edit Dockerfile: HERMES_REF=fix/parallel-tool-calls-v<NEW_TAG>
   git add Dockerfile && git commit && git push
   ```
   Railway auto-rebuilds.

## What's safe vs unsafe to modify

✅ **Modify freely:**
- Our `fix/parallel-tool-calls-v*` branches — they exist to carry our
  patches.

⚠️ **Don't modify `main`:**
- `main` should always track upstream `NousResearch/hermes-agent:main`
  so we can `git fetch upstream` and create new fix branches cleanly.

❌ **Don't open PRs to upstream from this repo unless explicitly
   asked.** The user reviewed an upstream PR draft and decided to hold
   it pending behavioural validation. See workspace `CHANGELOG.md`
   entry for `2026-05-28 — Hermes Agent patch`.

## How to verify the patch is live in the deployed container

```bash
railway service files download \
  /opt/hermes-agent/agent/transports/chat_completions.py /tmp/check.py
grep -n parallel_tool_calls /tmp/check.py
# Should show two lines setting the kwarg to True.
```

The workspace's `verify-upgrade.sh` runs this check on every nightly
upgrade tick.

## Upstream notes

- Upstream repo: `NousResearch/hermes-agent`
- Maintainer activity is heavy — `teknium1` reviews and merges most
  `agent/transports/` PRs.
- Release cadence: ~weekly, tagged `vYYYY.M.D`.
- Contributing guide: `https://hermes-agent.nousresearch.com/docs/developer-guide/contributing`
- Title convention: `fix(agent):` / `feat(agent):` (Conventional Commits)
- Tests live at `tests/agent/transports/test_chat_completions.py` and
  `tests/providers/test_provider_profiles.py`

## For full deployment context

See `Venture-Formations/hermes-workspace`:

- `CLAUDE.md` — deployment-wide overview
- `CHANGELOG.md` — every change since gbrain install
- `OPERATIONS_LOG.md` — infra history
