# qwen3.5-122b-a10b

`qwen3.5-122b-a10b` — (model)

## Decode throughput
Honest measure — **greedy** (temp 0), **prompt cache disabled**, a **fresh generation per
request**; decode tok/s from llama.cpp `timings`. (1 run(s) × 180 tokens.)

| Workload | Decode tok/s |
|---|---|
| Free-form prose | _n/a_ |
| Code generation | _n/a_ |
| JSON / structured | _n/a_ |
| Chat / dialogue | _n/a_ |
| Math / reasoning | _n/a_ |
| Translation (multilingual) | _n/a_ |
| Summarization | ~33.9 |

**Average decode ≈ 33.9 tok/s · Prefill ≈ 34.7 tok/s.** Status: ⚠️ partial (1/7 — see notes)

## Serving configuration
| Param | Value |
|---|---|
| Backend | llama.cpp |
| Quant | UD-IQ4_XS (GGUF) |
| Engine build | b9733-f449e0553 |
| GPU(s) | RTX 5090 32G + 5080 16G (48G) (consumer — no ECC) |
| GPU cards used | ? |
| GPU layers | -ngl ? |
| MoE expert offload | --n-cpu-moe —  |
| Tensor split | — |
| KV cache | fp16 |
| Context | 8192 |
| Flash-attn | — |
| Port / node | :? on ? |

## Notes
- Only 1/7 workloads returned — the rest errored (server swapped/restarted or timed out during the run). Re-run in a quiet window.
- ECC: N/A (consumer GPU). Model path: `/data/models/qwen3.5-122b-MTP/UD-IQ4_XS/Qwen3.5-122B-A10B-UD-IQ4_XS-00001-of-00003.gguf`.

_Measured 2026-06-27 · `bench_and_report.py` · config from mctl registry + /props._
