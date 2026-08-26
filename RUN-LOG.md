# Qwen 3.8 27B Q6 Quant Shootout — Full Run Log

## Results at a glance

| Game | Atomic AD-Q6_K | Bartowski Q6_K | Unsloth UD-Q6_K_L |
|---|---|---|---|
| Pool | ✅ works — 118,069 tok, 17.7 min, 110.91 t/s | ✅ works — 100,296 tok, 22.7 min, 73.73 t/s | ❌ no working build — 84,529 tok, 21.8 min, 64.69 t/s; table and pockets render, simulation never advances |
| Air hockey | ✅ works — 86,564 tok, 12.2 min, 118.69 t/s | ⚠️ table renders, puck+mallets invisible — 71,480 tok, 14.7 min, 80.84 t/s | ✅ works — 70,734 tok, 17.8 min, 66.11 t/s |
| Foosball | ✅ works — 110,992 tok, 16.7 min, 110.64 t/s | ✅ works — 53,326 tok, 11.6 min, 76.54 t/s | ✅ works — 71,538 tok, 16.4 min, 72.49 t/s |
| Bowling | ✅ works — 77,464 tok, 10.8 min, 119.95 t/s | ✅ works — 84,292 tok, 16.4 min, 85.68 t/s | ✅ works, score HUD frozen — 89,665 tok, 21.9 min, 68.13 t/s |

## Contestants

| | Atomic Chat | Bartowski | Unsloth |
|---|---|---|---|
| HF repo | `AtomicChat/Qwen3.8-27B-GGUF` | `bartowski/Qwen3.8-27B-GGUF` | `unsloth/Qwen3.8-27B-GGUF` |
| File | `Qwen3.8-27B-AD-Q6_K.gguf` | `Qwen3.8-27B-Q6_K.gguf` | `Qwen3.8-27B-UD-Q6_K_L.gguf` |
| Size | 23.29 GiB | 21.85 GiB | 22.53 GiB |
| Run in | this session | this session | this session |

## Backend

- **llama.cpp**, built from source on each box: `master` + **PR #27342** (`draft-dflash` speculative-decoding support, unmerged at run time), fetched via `git fetch origin pull/27342/head` on 2026-08-25. CUDA build (`-DGGML_CUDA=ON`, Release, single-arch for the box's compute capability, targets `llama-server` + `llama-bench`).
  - *Limitation:* the exact commit hash was written to `/workspace/BUILD_COMMIT` on each box but not copied off before teardown. Both boxes cloned within minutes of each other, so both ran the same PR head.
- **Drafter (speculative decoding):** DFlash2 — `z-lab/Qwen3.8-27B-DFlash2-GGUF`, file `Qwen3.8-27B-DFlash2-Q8_0.gguf`, via `--spec-type draft-dflash --spec-draft-n-max 7`. Identical drafter file on both boxes.
- **Exact server invocation** (from `boxes/bootstrap_*.sh`; only the model file differs between boxes):

```
llama-server -m <quant>.gguf --host 0.0.0.0 --port 4101 -ngl 999 -c 131072 \
  -fa on --jinja --parallel 1 --metrics --no-warmup --alias qwen38-27b-<role> \
  -md Qwen3.8-27B-DFlash2-Q8_0.gguf --spec-type draft-dflash --spec-draft-n-max 7
```

Full offload (`-ngl 999`), 262k context, flash attention on, chat template from the GGUF via `--jinja`, single slot.

## Hardware

Each quant ran on its own rented **RTX PRO 6000 Workstation (96 GB)** — one box per quant, same GPU model.

## Request, sampling, transport

Request body (identical for every run; saved in `outputs/bartowski_*_req.json`, `outputs/.unsloth_pool_body.json`):

```json
{
  "model": "qwen38-27b-<role>",
  "messages": [{ "role": "user", "content": "<game prompt verbatim>" }],
  "max_tokens": 125000,
  "chat_template_kwargs": { "reasoning_effort": "xhigh" }
}
```

- `xhigh` is the highest reasoning rung this model's chat template accepts.
- **Sampling:** no sampler parameters were set in any request — generation ran on the llama-server build's defaults, with no fixed seed (generation is non-greedy, so results are not reproducible run to run). *Limitation:* the effective default values on that build were not archived before teardown.
- **Transport:** all successful generations were carried over SSE streaming and reassembled into a standard completion object (recorded in the responses' `_transport` field). Reason: vast's proxy kills idle non-streamed connections at ~3 minutes. Streaming does not alter generation.

## Prompts

| File | Bytes | sha256 | Prompt tokens (server-side `prompt_n`) |
|---|---|---|---|
| `01_pool.txt` | 12,289 | `660f1e6e11b58b518df93ff8aba1f670ffdd7ed5619f3932068196879e774360` | 2,841 or 2,883 depending on prefix-cache state (see note below) |
| `02_air_hockey.txt` | 17,490 | `d0576b2735f688aa4a7beaeec3534029fee5342884901e375919d1c490a1794f` | 4,026 (both) |
| `03_foosball.txt` | 17,960 | `f8aeed753da6c92c238c8521074291616ffbdc80c739e3c7656c981eb55f2667` | 4,120 (both) |
| `04_bowling.txt` | 18,486 | `dda492ccf66886a4c3034e38fd3aa81ec25d124d511b415d220ab90c884acde7` | 4,195 (both) |

The 42-token `prompt_n` spread on the pool prompt is explained by prefix caching: `prompt_n` counts only *non-cached* prompt tokens. A server's very first request processes all 2,883 (`cache_n=0`); later requests reuse a 42-token cached prefix and report 2,841 (`cache_n=42`). Total prompt tokens are identical on every run.

## Per-run log

All server-side numbers from the `timings` block llama-server returns with each completion (`predicted_n`, `predicted_ms`, `draft_n`, `draft_n_accepted`). t/s = `predicted_per_second` (decode only). "Reasoning/content" are character counts of `reasoning_content` vs `content` in the response.

### Atomic AD-Q6_K

| Game | Tokens | Decode time | t/s | Draft accepted | Finish | Reasoning/content chars | Syntax gate | Result |
|---|---|---|---|---|---|---|---|---|
| pool | 118,069 | 1064.6 s | 110.91 | 83,061/245,056 = 33.9% | stop | 284,477 / 33,136 | PASS | Rack, break, balls pocketed |
| air_hockey | 86,564 | 729.3 s | 118.69 | 62,267/170,079 = 36.6% | stop | 190,228 / 25,699 | PASS | Puck, mallets, scoring |
| foosball | 110,992 | 1003.2 s | 110.64 | 78,080/230,377 = 33.9% | stop | 263,919 / 35,823 | PASS | Rods, players, ball physics |
| bowling | 77,464 | 645.8 s | 119.95 | 55,895/150,983 = 37.0% | stop | 157,934 / 28,035 | PASS | Ball rolls, pins topple |

393,089 tokens in 57.4 min of decode, weighted mean **114.17 t/s**, draft acceptance 35.1%, 4/4 clean stops.

### Bartowski Q6_K

| Game | Tokens | Decode time | t/s | Draft accepted | Finish | Reasoning/content chars | Syntax gate | Result |
|---|---|---|---|---|---|---|---|---|
| pool | 100,296 | 1360.3 s | 73.73 | 68,194/224,714 = 30.3% | stop | 254,269 / 29,406 | PASS | Rack, break, balls pocketed |
| air_hockey | 71,480 | 884.2 s | 80.84 | 50,373/147,749 = 34.1% | stop | 166,779 / 24,291 | PASS | No working build — table draws, puck and mallets never appear |
| foosball | 53,326 | 696.7 s | 76.54 | 36,571/117,285 = 31.2% | stop | 122,026 / 28,487 | PASS | Rods, players, ball physics |
| bowling | 84,292 | 983.8 s | 85.68 | 61,133/162,113 = 37.7% | stop | 175,691 / 22,839 | PASS | Ball rolls, pins topple, cam per spec |

### Unsloth UD-Q6_K_L

Pool ran at a 150,000 max_tokens cap; the other three at 125,000.

| Game | Tokens | Decode time | t/s | Draft accepted | Finish | Reasoning/content chars | Syntax gate | Result |
|---|---|---|---|---|---|---|---|---|
| pool | 84,529 | 1306.7 s | 64.69 | 58,915/179,298 = 32.9% | stop | 197,802 / 26,506 | PASS | No working build — table, pockets and aim line draw, but the simulation never advances and only ~3 balls exist instead of a 15-ball rack |
| air_hockey | 70,734 | 1069.9 s | 66.11 | 49,611/147,861 = 33.6% | stop | 166,658 / 23,766 | PASS | Puck, mallets, scoring |
| foosball | 71,538 | 986.9 s | 72.49 | 52,078/136,220 = 38.2% | stop | 146,329 / 23,871 | PASS | Rods, players, ball physics |
| bowling | 89,665 | 1316.1 s | 68.13 | 64,042/179,354 = 35.7% | stop | 191,380 / 26,757 | PASS | Ball rolls, pins scatter, rack resets; FRAME/TOTAL HUD counters stay frozen while PINS updates |
