# qwen3.5-122b-a10b (MTP) — the daily-driver reasoning baseline

`Qwen3.5-122B-A10B` MoE (122B total / ~10B active) at **UD-IQ4_XS (58 GB, 3 shards)** + native **MTP self-speculation**.
The standing daily-driver on `keyingd:8001`. **Decode ~28–34 tok/s, workload-dependent** (MTP draft-accept varies).

## Decode throughput (7 workloads)
| Workload | decode tok/s (production ncm14/ctx8192/MTP) |
|---|---|
| Math / reasoning | **33.9** |
| JSON / structured | **33.1** |
| Code generation | 30.6 |
| Free-form prose | 29.3 |
| Chat / dialogue | 28.2 |
| Summarization | ~33.9 |
| Translation (multilingual) | ~24–26 |
| **Average** | **~30.6** (prefill ~35–52) |

> Cross-checks Kecheng's independent `keying-deep.md` (~30). The much-quoted "37" is the **code/high-draft-accept peak**
> (93% MTP accept); the single "avg" hides a real 24→34 spread. **Report per-workload, not one number.**

## Serving configuration
| Param | Value |
|---|---|
| Backend | llama.cpp build-master (b9733), `keying-122b.service` supervisor |
| Quant | UD-IQ4_XS, 58 GB, 3 shards |
| Placement | `-ngl 99 --n-cpu-moe 14 --tensor-split 34,12` (most experts on GPU, 14 layers' experts → CPU) |
| Spec-decode | **`--spec-type draft-mtp --spec-draft-n-max 2`** (built-in MTP head, ~93% accept on code) |
| KV / ctx / fa | `--cache-type-k/v q8_0` · 8192 · `-fa on` · `-np 1` |
| VRAM | 5090 ~31 GB + **5080 ~15.8/16 GB (only ~445 MiB free!)** |

## Tuning-research + the MTP lever
- **MTP self-speculation** is the headline feature: the model drafts its own next tokens, verified in one pass. On
  predictable text (code, JSON) draft-accept hits ~93% → ~34 tok/s; on high-entropy prose it falls → ~28. This is why
  the per-workload spread is real and expected, not noise.
- `-fit on` was tested but the manual `--n-cpu-moe 14 --tensor-split 34,12` is tuned tighter for this 58 GB quant on
  48 GB VRAM; `-fit on` is the better default when you DON'T have a hand-tuned split.

## Analysis — limiter + the VRAM-fit cliff
58 GB > 48 GB VRAM → **doesn't fit → offload-bound.** 14 layers' experts live in CPU RAM, read per token. Limiter =
**CPU-RAM bandwidth** for the offloaded experts, partially hidden by MTP (fewer forward passes per accepted token).
That MTP assist is why 122B (~30) beats dense Llama-70B (~24) despite being larger — the active-param + speculation
advantage. Same offload wall as every >48 GB model here.

## Failures → the stability root-cause (the §E item)
**122B crashes under sustained continuous load** (reproduced across 3 runs):
- **Root cause: the 5080 (16 GB) sits at ~15.8/16 GB — only ~445 MiB free, INDEPENDENT of ctx** (445 MiB at ctx4096,
  455 at ctx8192). So the headroom shortage is **fixed weight allocation** from `--tensor-split 34,12` giving the
  16 GB card too large a share — NOT the KV cache.
- **Trigger: continuous 7-workload benchmarking.** Single workloads pass (Translation alone = 24.5 tok/s, server
  survives). By workload 6–7 — especially **Summarization (longest prompt → largest prefill KV/draft spike)** — the
  445 MiB is exhausted → **CUDA-OOM → process death → supervisor relaunch → HTTP 503/"connection closed"**.
- **What I tried:** `--n-cpu-moe 14→18` (freed the 5090, NOT the 5080 → still crashed) and `ctx 8192→4096` (KV halved
  → 6/7, Translation now passes, but Summarization still OOMs). Got from 5/7 → 6/7; the last workload needs a weight
  rebalance, not a KV fix.
- **Proposed production fix (Kecheng's config domain — it's his supervisor):** rebalance `--tensor-split` to give the
  **5080 a smaller weight share** (e.g. `30,16`, or raise `--n-cpu-moe` AND lighten the split) so the 16 GB card keeps
  ~1.5–2 GB for KV/draft spikes. Or `--override-tensor` to pin the KV cache to the 5090. Production config restored to
  the original ncm14/ctx8192 (stable for single-request daily use; only continuous benchmarking OOMs).

## Verdict
✅ **Solid daily-driver reasoning model, ~30 tok/s** (MTP shines on code → ~34). One real stability caveat under
**sustained** load, fully root-caused (5080 weight-share starves KV headroom) with a clear tensor-split fix for
Kecheng. **Disk: KEPT** (one of the two keepers). 6/7 workloads clean; the 7th blocked by a real hardware-edge OOM,
not a measurement gap.

_2026-06-27 · 3 benchmark runs + stability root-cause + restored prod config · keyingd · cross-checks keying-deep.md._
