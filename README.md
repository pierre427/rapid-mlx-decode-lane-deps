# mlx-lm decode-lane dependencies (companion to Rapid-MLX #3141 / #3142)

This repository is a **review-grade snapshot** of the mlx-lm–side modules that
two Rapid-MLX pull requests depend on:

- **[Rapid-MLX #3141](https://github.com/raullenchai/Rapid-MLX/pull/3141)** —
  compiled-decode-replay plain-decode lane (dense MoE / 35B).
- **[Rapid-MLX #3142](https://github.com/raullenchai/Rapid-MLX/pull/3142)** —
  megakernel plain-decode lane (Flash-Next `qwen4_exp` + dense MoE).

Neither PR vendors model or kernel code into Rapid. Both are thin, opt-in,
fail-closed lanes that are a **silent no-op on a stock mlx-lm wheel** and only
engage when the installed mlx-lm build exposes the surfaces below. This repo
exists so the depended-on code can be **read and reviewed** alongside those PRs.

> **Read this first — what this repo is and is not.**
> This is a snapshot of the *new, additive* modules only. It is **not a
> standalone-importable package** and is **not** the full build. The modules
> here import from other mlx-lm modules that are **not** included (see
> *Why the wiring isn't here*), so `pip install`-ing this directory will not
> produce a working lane. It is provided for code review. End-to-end device
> numbers are reproduced by the source lab and reported in the PR bodies; a
> joint validation run can be arranged if desired.

---

## What's included

### Compiled-decode surface (backs #3141)
- `mlx_lm/compiled_decode.py` — the compiled-replay step, context policy,
  numerics acceptance, shape-stable-cache conversion.
- `mlx_lm/compiled_qualification.py` — the reviewed serving-manifest identity
  and acceptance (`serving_qualification_reason`).

### Megakernel surface (backs #3142)
- `mlx_lm/megakernel_lane.py` — the lane object and its entry points
  (`megakernel_lane_enabled`, `attach_megakernel_lane`,
  `preload_megakernel_lane`), invoked from inside `generate_step`.
- `mlx_lm/models/qwen4_megakernel*.py` (9 modules) — the whole-token persistent
  Metal kernel, its device calibration, packing, runtime, schedule, contract,
  and config.

Full file list and a record of the small anonymization edits made to this
snapshot are in [`PROVENANCE.md`](PROVENANCE.md).

---

## Why the wiring isn't here

Both lanes hook into three core mlx-lm files — `mlx_lm/generate.py`,
`mlx_lm/models/cache.py`, and `mlx_lm/server.py`. On the source tree those three
files have diverged from stock mlx-lm (v0.31.3, this build's base) by roughly
**+12,700 lines**, and that divergence carries **many unrelated features**, not
just these two lanes. There is no clean seam that extracts "only the
compiled-decode and megakernel hooks" from those files in runnable form, so they
are **deliberately not reproduced here**. Instead, the exact surface those files
must expose is documented below, so a reviewer can see precisely how the
included modules attach.

Concretely, the included modules import from non-included modules — e.g.
`compiled_decode.py` imports `RingKVCache` / `_ring_buckets` from
`mlx_lm/models/cache.py`, and `megakernel_lane.py` imports the `qwen4_exp`
(Flash-Next) model. That is why this snapshot does not run on its own.

---

## Integration surface (what the host mlx-lm build must expose)

### For #3141 (compiled-decode)
- `mlx_lm.compiled_decode`: `CompiledDecodeStep`, `compiled_decode_step`,
  `compiled_decode_enabled`, `compiled_decode_context_policy`,
  `compiled_decode_numerics_accepted`, `compiled_decode_serving_reason`,
  `model_is_compilable`, `to_shape_stable_cache`; the class-1 bucket-ladder
  acceptance and the `short` / `memory` / `latency` / `long` context profiles
  (`long` → 262144 on the extended ladder).
- `mlx_lm.compiled_qualification`: `serving_qualification_reason`; identity =
  config / weights / parameter_layout / runtime / environment, plus numerical +
  serving evidence and spot-check revalidation, bound via the
  `MLX_LM_COMPILED_DECODE_QUALIFICATION` environment variable.
- `mlx_lm.models.cache`: `RingKVCache` (with `from_kv_cache` / `to_kv_cache`,
  `reserve`, `size`, `trim`, `is_trimmable`) and the `_RING_KV_BUCKETS` ladder
  (`_ring_buckets`).
- `mlx_lm.generate.generate_step` request-private compiled plumbing: the
  `compiled_decode`, `_prompt_cache_is_request_private`, and
  `_compiled_decode_status` keyword arguments. An older `generate_step` without
  these makes the lane a no-op.
- Server-side APC publish-back lives in `mlx_lm.server`
  (`_compiled_cache_publishable`, `_compiled_cache_for_publication`,
  `_compiled_request_selected`, `insert_cache`). The Rapid runner mirrors this
  at its own surface and publishes the **stock** cache form
  (`RingKVCache.to_kv_cache()`), never a `RingKVCache`.

### For #3142 (megakernel)
- `mlx_lm.megakernel_lane`: `megakernel_lane_enabled`,
  `attach_megakernel_lane(model, prompt_cache, *, max_tokens, status)`,
  `preload_megakernel_lane`, and the in-`generate_step` hook that attaches the
  lane for width-1 plain requests.
- The master switch is the `MLX_QWEN4_MEGAKERNEL` environment variable; a build
  that captures it at import must see it set before the model loads.
- Model eligibility is resolved inside the lane (it targets the `qwen4_exp`
  Flash-Next geometry and the dense `qwen3_5_moe` geometry); there is **no**
  separate `geometry_for_model_type` registry.

---

## Two operational gotchas (why "have the code" ≠ "runs the lane")

1. **Per-device calibration.** The megakernel requires a validated *primitives*
   calibration keyed by **both** the GPU signature and the **mlx core version**
   (a per-device tune cache). On a device/core with no cached, validated entry,
   the lane **declines cleanly (fail-closed to eager)** rather than engaging.
   Running the lane on a new machine requires the one-time primitive tune for
   that device and core.
2. **Pin the mlx core.** In testing, an mlx core one patch version off the
   calibration declined the lane (the fail-closed path, as designed). When
   enabling the megakernel lane, pin the mlx core to the calibrated version.

The compiled-decode lane (#3141) has no per-device kernel calibration, but is
gated by the reviewed serving manifest above and serves the mixed 4/6-bit
checkpoint natively via `mx.quantized_matmul` (no custom 6-bit decode).

---

## How to use this repo

- **To review the PRs:** read the modules here against the *Integration surface*
  section; that is the full contract the Rapid-side lanes rely on. The Rapid
  PRs are default-OFF and fail-closed, so their safety on a stock mlx-lm (the
  no-op path) is verifiable without this code.
- **To run end-to-end:** this snapshot is not sufficient on its own (see
  *Why the wiring isn't here*). Reach out to arrange a build or a joint
  validation session.

## License

MIT — see [LICENSE](LICENSE). This matches the license of the upstream mlx-lm
project these modules extend.
