# serve-qwen3.5-122b-a10b — how to serve **Qwen3.5-122B-A10B (MTP)** efficiently (any GPU)

*A GPU-agnostic operations manual for one model. The reference tok/s is from one rig; the **recipe applies to any GPU**
— figure out your VRAM, then follow it.*

- **Architecture:** MoE + native MTP self-speculation (122B total / 10B active)
- **Fits in VRAM?** NO — IQ4_XS ≈58 GB; must offload some experts

## Recipe
**Offload + MTP:** keep most on GPU via `--n-cpu-moe <small>` (just enough to fit) + asymmetric `--tensor-split` (small card gets less) + **`--spec-type draft-mtp --spec-draft-n-max 2`** (the built-in draft head) + q8 KV. **MTP speed is workload-dependent** — high on predictable code, low on prose.

**Serving flags (llama.cpp):**
```
-ngl 99 --n-cpu-moe 14 --tensor-split <fit> -fa on --cache-type-k q8_0 --cache-type-v q8_0 --spec-type draft-mtp --spec-draft-n-max 2
```

## Reference throughput
**~34 tok/s on code at 93% draft-accept; ~24 on prose.** The single 'avg' number is misleading — report per-workload. The much-quoted '37' is the code peak.

## Failures → fixes
- Crashes/503 under sustained load when VRAM is on the edge (KV/draft spike → process death + relaunch). **Bump `--n-cpu-moe` a notch for headroom** or cap ctx.
- Empty `content` → it's a thinking model; `/no_think` or `enable_thinking:false`.

## Verdict
Solid daily reasoning driver; MTP shines on code. Watch the VRAM edge.

---
## The one decision: does it FIT in your VRAM?
Estimate size ≈ params × bytes/weight (Q4≈0.5, Q8≈1, FP16≈2 B/param) + KV + ~2–3 GB overhead.
- **Fits** → full GPU residency, **no offload**, single card if it fits on one → *bandwidth-bound, fast.*
- **Doesn't fit** → offload experts to RAM (use `-fit on`), keep the active path on GPU → *RAM-bandwidth-bound, slower.*

## Measure honestly
Use the server's **`/completion` decode timings** (`predicted_per_second`), greedy, cache off, multiple workloads —
NOT short OpenAI wall-time (it understates decode). See `bench_decode.py`.

## Files
- `REPORT.md` — the detailed benchmark (throughput · config · tuning-research+sources · analysis · failures), if present.
- `bench_decode.py` — honest decode-tok/s measurement (`/completion` timings).
- `mctl-entry.json` — the `models.json` snippet for the [mctl](https://github.com/) switcher.

*Part of a per-model serving-playbook set. Cross-model comparison: see the `_ALL-MODELS-SUMMARY.md` overview.*
