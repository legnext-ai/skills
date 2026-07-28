# Legnext API Parameters

> ⚠️ **This file is a hand-placed initial version.** It will be superseded by an auto-generated pipeline that derives the parameter matrix from the backend spec. Until then, treat the live docs at https://docs.legnext.ai/getting-started/image-parameters and https://docs.legnext.ai/getting-started/models as authoritative; this file is a convenience mirror for the agent.

Parameters are appended to the prompt text as Midjourney-style flags, e.g. `A sunset over mountains --ar 16:9 --v 8.2 --stylize 200`. The `text` field of a request carries both the natural-language prompt and the flags.

## Version selection

| Version | Flag | Notes |
|---|---|---|
| v6, v6.1 | `--v 6` / `--v 6.1` | Legacy. `--cref` (character reference) works here. |
| v7 | `--v 7` | **API default** when no `--v` is given. Supports `--oref` (Omni Reference), `--draft`. |
| v8 | `--v 8` | Native 2K via `--hd`. Sunset is planned; new integrations should target v8.1/v8.2. `--hd`/`--q 4`/`--sv 6` cost 4×. |
| v8.1 | `--v 8.1` | Native 2K via `--hd` at 1.5×. `--cref`, `--oref`, `--q`, `--turbo`, `--sd` are **rejected** (HTTP 400 naming the flag). |
| v8.2 | `--v 8.2` | Aesthetics update on v8.1. Parameter surface identical to v8.1; migration is a one-flag change. |
| niji 6 | `--niji` | Anime / East-Asian aesthetic model. |

**Default-version gotcha (hard-won):** a prompt with no `--v` renders as **v7**, not the newest model. The API intentionally keeps v7 as the default so existing integrations don't shift behavior, and it never injects `--v` on your behalf. **Pin `--v` explicitly whenever the version matters.**

**Cross-version reference rules:** `--cref` works only on v6 / v6.1 / niji 6. v7+ replaced it with `--oref`. v8.1 / v8.2 support **neither** `--cref` nor `--oref`. So a no-`--v` (i.e. v7) prompt using `--cref` fails with `Invalid prompt format`; use `--v 7 --oref` or `--v 6.1 --cref`.

## Image parameter matrix

| Parameter | Flag | Values | Notes |
|---|---|---|---|
| Aspect Ratio | `--ar` / `--aspect` | up to 14:1 (4:1 for v8.1/v8.2 HD) | Integer ratios only (`85:110`, not `1.39:1`). |
| Chaos | `--c` / `--chaos` | 0–100 (default 0) | Higher = more unusual results. |
| Negative Prompt | `--no` | text | v6/v6.1/v7/v8/v8.1/v8.2 (not niji 6). |
| Quality | `--q` / `--quality` | 0.25, 0.5, 1, 2, 4 | v8 supports `--q 4` (4× cost). **Removed on v8.1/v8.2** (rejected). |
| Seed | `--seed` | 0–4294967295 | Same seed + prompt + version = consistent results. |
| Stop | `--stop` | 10–100 (default 100) | Not on v8 / v8.1. |
| Stylize | `--s` / `--stylize` | 0–1000 (default 100) | Lower = prompt-focused; higher = more artistic. |
| Weird | `--w` / `--weird` | 0–3000 (default 0) | |
| Raw Mode | `--raw` | (flag) | Reduces default aesthetic styling. |
| Tile | `--tile` | (flag) | Seamless tiled images. |
| Image Weight | `--iw` | 0–3 (default 1) | Influence of image prompts. |
| Style Reference | `--sref` | image URL | Match the look of another image. Pair with `--sw` (style weight). |
| Omni Reference | `--oref` | image URL | v7/v8 only. Replaces `--cref`. Incompatible with inpaint/outpaint/pan and speed modes. ~2× GPU. **Removed on v8.1/v8.2.** |
| Character Reference | `--cref` | image URL | v6 / v6.1 / niji 6 only. Pair with `--cw` (character weight). |
| Personalization | `--p` | code | v6/v6.1/v7/v8/v8.1 (not niji 6). Re-test when changing versions (different pools). |
| Experimental | `--exp` | (flag / 0–100) | v7 / v8 / v8.1. |
| HD Mode | `--hd` | (flag) | v8: native 2K at **4×**. v8.1/v8.2: native 2K (2048²) at **1.5×**; default (no `--hd`) is 1024² SD at 1×. |
| Draft | `--draft` | (flag) | v7: 4 images at **half** price. v8.1/v8.2: a **24-image batch at standard rate** (no discount), but only the **first 4** images are actionable (indices 0–3). **Not available on v8.0** (fails upstream). |
| Fast | `--fast` | (flag) | Default speed; usually < 1 min. |
| Turbo | `--turbo` | (flag) | ~4× faster, **2×** cost, v5+ only. **Not supported on any v8.** |
| Relax | `--relax` | (flag) | Currently **not supported** by the API. |
| Motion | `--motion` | 1–4 (default 2) | Video tasks only. |

### Not supported for image generation
`--repeat`, `--bs`/`--batchsize` (video-only, values 1/2/4), `--personalize`, `{}` variation syntax, `--stealth`/`--public`, `--video` (use video endpoints).

## Cost multipliers (relative)

Cost scales with flags; the per-task absolute price is published on the docs site and also appears in the response as `meta.usage.frozen` / `meta.usage.consume`. Use the response fields for billing, never a hardcoded table.

| Lever | Multiplier |
|---|---|
| Standard SD (default) | 1× |
| `--hd` on v8.1 / v8.2 (native 2K) | 1.5× |
| `--hd` / `--q 4` / `--sv 6` on v8 | 4× |
| `--turbo` | 2× |
| `--draft` on v7 | 0.5× |
| `--draft` on v8.1 / v8.2 | 1× (no discount; 24-image batch) |

**Child-task pricing edge (be aware):** child tasks (`remix`/`variation`/`reroll`) estimate cost from the **parent** prompt. A flag introduced only in the child prompt (e.g. `--hd`) may be mispriced (undercharged). Don't rely on child-task cost estimates for strict budget enforcement.

## Prompt construction tips
- Order: subject → style/medium → details → environment → flags.
- Multi-prompts: `::` separates concepts; `::2` weights a concept 2×.
- Wrap literal text to render in `"quotes"` (v8+ renders text well).
- Keep flag combinations logical; start moderate and adjust.

## Full reference
- Image parameters: https://docs.legnext.ai/getting-started/image-parameters
- Models & version support: https://docs.legnext.ai/getting-started/models
- Video parameters: https://docs.legnext.ai/getting-started/video-parameters
