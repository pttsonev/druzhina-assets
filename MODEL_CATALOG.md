# Druzhina Model Catalog

Canonical inventory of which model each agent runs, where it is configured, and how
upgrades happen. Updated 2026-06-11 (research pass + fleet upgrade). One GPU: RTX 5090,
32GB VRAM, WSL2. Serving: **Ollama only**, loopback `127.0.0.1:11434`.

## Per-agent assignments

| Agent | Tier / job | Model | VRAM (Q4) | Env var | Default set in | Status |
|---|---|---|---|---|---|---|
| vidin | stage 2 fast triage | `qwen3.5:9b` | ~7GB | `VIDIN_VISION_MODEL` | `vidin/vidin/vision_client.py` | default since 2026-06-11; live comparison: old 3b flagged 71/71 (useless), 9b flagged 21/71 |
| vidin | stage 2 fallback | `qwen2.5vl:3b` | ~3GB | (same) | n/a | retired from triage; drop after the formal eval lands |
| vidin | stage 2.5 describe (all segments) | `gemma4:31b` | ~20GB | `VIDIN_DEEP_MODEL` | `vidin/vidin/run.py` | default since 2026-06-12; work types + descriptions + burn-in transcription |
| vidin | stage 2.5/3 alternate | `qwen3-vl:30b` | ~20GB | `--deep-model` flag | n/a | benchmark alternate (bead drz-56n); stage 3 deep analysis still a follow-up bead |
| vidin | second opinion, temporal artifacts | `minicpm-v4.5:q4_K_M` | ~7GB | `VIDIN_VISION_MODEL_2ND` (future) | not wired yet | pulled; disagreement -> needs_human tier is a follow-up bead |
| chavdar | code/security review | `qwen3.6:27b` | ~19GB | `CHAVDAR_LOCAL_MODEL` | `chavdar/chavdar/model_client.py` | default since 2026-06-11 (was `qwen3:8b`, the noisy-review culprit) |
| chavdar | cloud reviewer | `anthropic:<model>` | n/a | CLI `--reviewer` | `chavdar/chavdar/cli.py` | adapter ready, awaiting `ANTHROPIC_API_KEY` (bead drz-x3r) |
| botev | shot tagging | `qwen3:30b-a3b-instruct-2507-q4_K_M` | ~18GB | `BOTEV_TAGGING_MODEL` | `botev/botev/tagging.py` | stable; pricing itself is pure Python, no model |
| vela | local workhorse (private data) | `qwen3:30b-a3b-instruct-2507-q4_K_M` | ~18GB | `LOCAL_MODEL_DEFAULT` | `personal-assistant/lib/local_agent.py` | stable |
| vela | local fast | `qwen3:8b` | ~5GB | `LOCAL_MODEL_FAST` | (same) | stable |
| vela | cloud tiers | `claude-sonnet-4-6` / `claude-haiku-4-5` / `claude-opus-4-7` | n/a | `ANTHROPIC_MODEL_*` | `personal-assistant/lib/claude_agent.py` | only CloudSafe sanitized text, $50/mo cap |

## Pulled on the box (`ollama list` should match)

`qwen3:30b-a3b-instruct-2507-q4_K_M` (18GB) - `qwen3:8b` (5.2GB) - `qwen2.5vl:3b` (3.2GB)
- `qwen3.5:9b` (6.6GB) - `qwen3-vl:30b` (20GB) - `qwen3.6:27b` (17GB) - `minicpm-v4.5:q4_K_M` (~6GB)
- `gemma4:31b` (~20GB, pulled 2026-06-12)

## VRAM doctrine

32GB total; **swap, don't co-reside**. Ollama `keep_alive` eviction is the scheduler:
Vela's 30B and Vidin's vision models trade the card; batch jobs (vidin runs, botev
tagging, chavdar reviews) tolerate the ~10-100s cold load. Never run two ~20GB models
concurrently. Record measured load times in each repo's decisions.md.

## Serving doctrine

- **Ollama only**, loopback. Structured outputs (`format` JSON schema, grammar-enforced)
  work with vision models and are the vocabulary-breach backstop; keep the strict Python
  parse on top (truncated generations can still emit invalid JSON).
- **Odysseus is a cockpit UI** (127.0.0.1:7000), not a serving layer. Zero agent coupling.
- **vLLM trigger** (documented, not active): if grammar-JSON BATCH throughput becomes the
  bottleneck (hundreds of shots x several frames at deep-analysis tier), move that tier to
  vLLM (AWQ/FP8, fp8 KV cache) and REPLACE Ollama for that duty. vLLM pre-allocates VRAM,
  so cohabitation with Ollama eviction does not work; sleep-mode orchestration is manual.

## Upgrade procedure

1. Research + pick the candidate (record reasoning here).
2. Petar pulls it (`ollama pull ...`; installs are deny-listed to agents; this session's
   pulls were explicitly authorized by Petar).
3. GPU smoke: the repo's `pytest -m gpu` live test against the new tag.
4. Flip the env default in the owning repo (one line), rerun the full gate.
5. Note the flip + measured VRAM/latency in that repo's `docs/agent/memory/decisions.md`.
6. Update this catalog. Keep the old model pulled until the next eval proves the new one.

## Benchmark-later queue

- `glm-4.7-flash` (30B-A3B MoE, ~158 t/s on the 5090) vs `qwen3.6:27b` for chavdar:
  needs a pre-release Ollama (>= 0.14.3); bead filed in chavdar.
- `qwen3.6:35b` (24GB) vs `qwen3-vl:30b` for vidin stage 3: tighter KV headroom, eval
  on real footage when stage 3 lands.
- `gemma4:31b` (~20-24GB Q4, dense, multimodal, 256K ctx) vs `qwen3-vl:30b` for vidin
  stage 3: Gemma 4 (released 2026-04-02, Apache 2.0) wins multimodal + human-preference
  benches (Arena Elo ~1452) while qwen3.6 wins agentic/static tasks; head-to-head on our
  footage when stage 3 lands. NOTE 2026-06-11: the original research pass WRONGLY
  excluded Gemma 4 as SEO rumor; corrected. Gemma 3 27B is now two generations old:
  do not adopt it anywhere new.
- Vela: qwen3.5/3.6 family as workhorse candidates (native vision could collapse a tier);
  blocked on a redaction-quality eval, bead filed in personal-assistant. Vela's designated
  redaction fallback updates Gemma 3 27B QAT -> `gemma4:31b` (same eval gate applies).
- TransNetV2 (~35M params, learned cut detector) vs ffmpeg scene threshold for vidin
  stage 1: clearly better on dissolves in published evals; decide on Frank eval evidence.

## Sources

Research pass 2026-06-11 (web, primary sources + benchmark reproductions): Qwen3-VL /
Qwen3.5 / Qwen3.6 releases (Oct 2025 / Feb 2026 / Apr 2026), Ollama multimodal engine +
structured outputs docs, vLLM structured-outputs + sleep-mode docs, MiniCPM-V 4.5 paper
(3D-Resampler, 96x token compression), TransNetV2 + OmniShotCut (2026) cut-detection
comparisons. Caveats logged: Qwen3.5-9B "beats last-gen 30B vision" rests on one HF
discussion; SWE-bench numbers across vendors use different harnesses; verify on our own
footage/diffs before trusting deltas.
