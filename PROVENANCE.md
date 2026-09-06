# PROVENANCE

Review-grade snapshot of mlx-lm decode-lane dependency modules for
Rapid-MLX PRs #3141 (compiled-decode) and #3142 (megakernel).

**Base:** the source mlx-lm build is version 0.32.0, forked from stock
mlx-lm **v0.31.3** (ml-explore/mlx-lm). Only the *new, additive* modules
listed below are included; no other source-tree code is present, and the
three divergent core files (generate.py, cache.py, server.py) are
intentionally excluded (see README).

## Files (LOC)
```
     836 mlx_lm/compiled_decode.py
     573 mlx_lm/compiled_qualification.py
     208 mlx_lm/megakernel_lane.py
    1303 mlx_lm/models/qwen4_megakernel.py
    2207 mlx_lm/models/qwen4_megakernel_body.py
     473 mlx_lm/models/qwen4_megakernel_config.py
     220 mlx_lm/models/qwen4_megakernel_contract.py
     380 mlx_lm/models/qwen4_megakernel_device.py
     857 mlx_lm/models/qwen4_megakernel_pack.py
    1190 mlx_lm/models/qwen4_megakernel_runtime.py
     999 mlx_lm/models/qwen4_megakernel_schedule.py
     763 mlx_lm/models/qwen4_megakernel_tune.py
   10009 total
```

## Anonymization edits applied to this snapshot

Five lines were altered from the source originals to remove
internal identifiers. No logic was changed.

| File | Change |
|---|---|
| compiled_qualification.py | env-key filter prefix `MLXUAG_` -> `MLXPRIV_` |
| models/qwen4_megakernel.py | dropped a contributor name from a code comment |
| models/qwen4_megakernel_tune.py | default lock path -> `/tmp/mlx-megakernel/gpu.lock` |
| models/qwen4_megakernel_tune.py | default cache path -> `~/.cache/mlx-megakernel/megakernel-tune.json` |
| models/qwen4_megakernel_config.py | docstring cache path -> `~/.cache/mlx-megakernel/...` |

Verified: zero remaining matches for internal names, host addresses,
home paths, or email across all included files.
