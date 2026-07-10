# RESULTS 2026 — Modernizing ReAct for `gpt-4o-mini`

*Companion notes to this fork of [ysymyth/ReAct](https://github.com/ysymyth/ReAct).
See the fork notice at the top of the [README](README.md). Last updated 2026-07-10.*

## TL;DR

- Ported the 2022 ReAct HotpotQA/FEVER prompting code — written for OpenAI
  `text-davinci-002` on the legacy Completions API — to run today on
  **`gpt-4o-mini`** via the modern OpenAI **v1 SDK**.
- Fixed **five latent bugs** that only surfaced once the code hit live services
  and a modern model again. Every changed line is tagged `# ! -`.
- **HotpotQA (500 random dev, EM):** paper's davinci-002 **30.4%** → our
  gpt-4o-mini **35.4%** (separate-lines prompt). A "commit-nudge" variant is
  **pending** (see §6).
- **Most interesting finding:** gpt-4o-mini *hedges*. It fails to commit an
  answer **~29%** of the time, vs **~1.8%** for davinci — a
  base-vs-instruction-tuned **behavior** gap, not a flaw in ReAct.

> ⚠️ **Framing.** Beating the paper's ~30% is **model progress, not method
> progress.** We run a 2024 model against a 2022 baseline; a faithful
> head-to-head is impossible because `text-davinci-002` was retired
> (2024-01-04). Read these as *"ReAct with gpt-4o-mini + light prompt tuning,"*
> not *"we beat the ReAct paper."*

---

## 1. What this fork is

An independently-maintained fork that makes the original ReAct notebooks
*runnable in 2026*. Scope is deliberately narrow: modernize the API/model,
fix the bugs that block a run, and document the differences. The original
research logic (`wikienv.py`, `wrappers.py`, `prompts/`, the `webthink` loop)
is otherwise untouched. Original work © 2023 Shunyu Yao, MIT (retained).

## 2. Environment

| Item | Value |
|---|---|
| OS / Python | macOS, Python **3.14** (Homebrew), project `.venv` |
| Key deps | `openai` (v1), `notebook`, `requests`, `numpy` 2.x, `beautifulsoup4`, `gym` 0.26.2 |
| Notebooks | `hotpotqa.ipynb`, `FEVER.ipynb` (Wikipedia/`webthink`); `alfworld`/`webshop` not modernized |

**On `gym`:** it's unmaintained and prints a "does not support NumPy 2.0"
notice on import. Harmless here — ReAct only uses `gym.Env` / `gym.spaces.Space`
base classes, and the env passes *text*, not NumPy arrays. **Not** migrated to
`gymnasium`: ReAct uses the old 4-tuple `step()` API and gymnasium is 5-tuple,
so it is *not* a drop-in and would require rewriting `wikienv.py`/`wrappers.py`.

## 3. The migration: `text-davinci-002` → `gpt-4o-mini`

| | Original (2022) | This fork |
|---|---|---|
| SDK | `openai` v0.x | `openai` **v1** (`from openai import OpenAI`) |
| Auth | `openai.api_key = os.environ[...]` | `client = OpenAI()` (reads env) |
| Endpoint | `openai.Completion.create` | `client.chat.completions.create` |
| Model | `text-davinci-002` (**retired 2024-01-04**) | `gpt-4o-mini` |
| Input | `prompt="..."` (single string) | `messages=[{role, content}, ...]` |
| Output | `resp["choices"][0]["text"]` | `resp.choices[0].message.content` |

*(commits `17b186a`, `2d35e1e`)*

### Base vs. instruction-tuned — why this isn't a trivial swap

`text-davinci-002` is a **base completion** model: it continues text. ReAct
exploits that directly — the prompt is a half-finished Thought/Action/Observation
transcript and the model just continues it in-format, stopping at
`stop=["\nObservation N:"]`.

`gpt-4o-mini` is **instruction-tuned + RLHF**: it answers requests, narrates,
and hedges. That reshaped distribution is the root of nearly every behavioral
difference below. We add a `SYSTEM` message telling it to behave like a
completion engine, and a `stop` guard, to claw back base-like behavior — but,
as §5 shows, prompting only gets you part of the way.

## 4. Bug chronology — the "salvage" arc

None of these are gpt-4o-mini's fault per se; they're **latent bugs in code
that hadn't been run against live services since 2022**. All fixes are minimal
and `# ! -` marked.

| # | Symptom | Root cause | Fix | Commit |
|---|---|---|---|---|
| 1 | `model_not_found` / SDK has no `Completion` | model retired + v0 API removed | chat endpoint + `gpt-4o-mini` | `17b186a` |
| 2 | **Every `Observation` blank** | Wikipedia **403s the default `python-requests` User-Agent** | send a descriptive User-Agent | `9a82706` |
| 3 | `UnicodeDecodeError: truncated \UXXXXXXXX` | `clean_str`'s `unicode-escape` round-trip chokes on real page text containing backslash sequences | wrap + fall back to raw text | `fbb8df3` |
| 4 | `NameError: requests is not defined` | `step()`'s retry does `except requests.exceptions.Timeout` but the notebook never imported `requests` — so *any* error became a NameError | `import requests` | `fbb8df3` |
| 5 | `IndexError: string index out of range` | chat model put the action *inline* in the thought → `split("\nAction N:")` failed → fallback returned an **empty action** → `action[0]` crashed | guard empty action → `finish[]` | `7e1bd1b` |
| — | one bad example killed all 500 | no per-example isolation | wrap `webthink` in try/except; log + score `em=0` + continue | `6da7361` |

Housekeeping: `nbstripout` git filter strips outputs + volatile metadata so runs
don't churn the repo (`05ec15f`); full `# ! -` marking pass (`5be5305`).

## 5. Base-vs-chat behavioral findings

### 5.1 Format drift → the `ohh...` fallbacks

Our first `SYSTEM` said *"output only the next single line."* But `webthink`
expects **two** lines back (`Thought N:` **then** `Action N:`) and splits on
`"\nAction N: "`. The "single line" instruction made gpt-4o-mini cram both onto
one line — `"…to answer the question. Action 3: Search[…]"` — so the split
failed and dropped into the noisy (and crash-prone) `ohh...` fallback.

**Fix:** instruct *Thought and Action on separate lines* (`c7c0c88`). This is a
pure chat-model artifact — davinci followed the few-shot format natively.

### 5.2 Hedging → empty answers (the headline finding)

`webthink` caps each question at 7 steps; if the model never calls
`Finish[answer]`, it's force-finished with an empty answer (scored 0).

| Model | Task | Empty / "gave up" rate |
|---|---|---|
| `text-davinci-002` | FEVER (500, measured from the committed original run) | **1.8%** (9/500) |
| `gpt-4o-mini` | HotpotQA (500, this fork, separate-lines prompt) | **29%** (145/500) |

gpt-4o-mini gives up **~16× more often.** Cause: **RLHF rewards not asserting
things you're unsure of** — great for a chatbot, counterproductive for a
benchmark that scores 0 for silence. davinci, a base model tuned to the
few-shot format, committed fast (most FEVER episodes finished in `steps=2`).

The step distribution is **bimodal** — it either commits fast or times out,
almost nothing in between:

```
steps: Counter({3: 157, 8: 145, 4: 79, 2: 45, 5: 44, 6: 22, 7: 8})
                         ^^^^^^^ all 145 empties are steps=8 (hit the ceiling)
```

**Mitigation (pending eval):** a `SYSTEM` line instructing a best-guess
`Finish[...]` before running low on steps (`2f26aac`). Converting an empty
(already 0) into a guess is a strictly non-negative move for EM.

## 6. Results

HotpotQA, 500 random dev examples (seed 233), exact-match:

| Configuration | Model | EM | Empty rate |
|---|---|---:|---:|
| Paper (ReAct) | PaLM-540B | 29.4% | — |
| Paper / repo (ReAct) | `text-davinci-002` | 30.4% | ~low¹ |
| This fork, all crash-fixes + separate-lines prompt | `gpt-4o-mini` | **35.4%** | **29%** |
| This fork, + commit-nudge prompt (`2f26aac`) | `gpt-4o-mini` | **_pending_** | **_pending_** |

¹ Not directly measured on HotpotQA (the original notebook shipped without
outputs); davinci's FEVER empty rate was **1.8%**, so its HotpotQA empties were
almost certainly low too — its ~30% was mostly *committed-but-wrong*, not
*never-answered*.

**Error taxonomy (35.4% run):** two distinct failure modes —
*gave up* (empty, ~29%) vs *answered-but-wrong* (e.g. `Finish[torpedo boats and
submarines]` vs gt `torpedoes` → EM 0). The commit-nudge targets the first bucket;
whether EM actually rises tells us whether those timeouts were **hedging on
answers it had** (EM jumps) or **genuine retrieval failures** (EM stalls, empties
still drop).

<!-- FILL AFTER THE COMMIT-NUDGE RUN:
     - EM: ____   empty: ____ / 500
     - Did steps 2–4 stay healthy? (guards against premature commitment)
     - Interpretation: hedging-recovered vs retrieval-bound
-->

## 7. Reproduce

```bash
brew tap raocow/tap && brew install ezenv    # optional: auto-activates the venv on cd
cd ReAct && python3 -m venv .venv && source .venv/bin/activate
pip install openai notebook requests numpy beautifulsoup4 gym
export OPENAI_API_KEY="sk-..."
jupyter notebook hotpotqa.ipynb              # Kernel → Restart & Run All
```

Metrics after a run:

```python
from collections import Counter
print("EM:", sum(rs) / len(rs))
print(Counter(i['steps'] for i in infos))
print("empty:", sum(1 for i in infos if not i.get('answer')))
```

## Appendix — change log

| Commit | Change |
|---|---|
| `17b186a` | migrate `llm()` to OpenAI v1 SDK + `gpt-4o-mini` |
| `2d35e1e` | completion-style SYSTEM + `Observation` stop guard |
| `9a82706` | Wikipedia User-Agent (fixes blank observations) |
| `7e1bd1b` | guard empty action (fixes `IndexError`) |
| `05ec15f` | strip volatile notebook metadata |
| `c7c0c88` | SYSTEM: Thought/Action on separate lines |
| `fbb8df3` | harden `clean_str`; add missing `requests` import |
| `6da7361` | per-example loop guard |
| `5be5305` | complete `# ! -` marking pass |
| `2f26aac` | SYSTEM: commit a best-guess `Finish[]` instead of timing out |
