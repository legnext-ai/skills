---
name: legnext
description: Integrate the Legnext AI API (Midjourney image and video generation accessed via a REST API) into a codebase, script, or agent workflow. Covers submitting imagine/diffusion tasks, choosing polling vs webhooks, fetching and persisting generated images, handling credits (freeze/consume), child tasks (variation/upscale/reroll/remix) that depend on a parent task, content-moderation failures, retries/idempotency, and per-error-class handling. Use this skill when the user wants to add Legnext / Midjourney-via-API generation to their app, build a generation pipeline, or debug a Legnext API integration.
---

# Legnext API Integration

Legnext (`https://api.legnext.ai`) gives you Midjourney image and video generation through a synchronous REST API: you submit a task, it runs asynchronously, and you collect the result by polling or webhook. This skill is the **integration methodology** — the API reference lives at https://docs.legnext.ai and is linked from each section below. Read the reference files in `references/` on demand; do not load them all up front.

## The integration loop

Every generation is three steps:

1. **Submit** — `POST /api/v1/diffusion` with `{ "text": "<prompt>", "callback": "<optional webhook>" }` and the `x-api-key` header. Returns a job object with `status: "pending"` and a `job_id`. Credits are **frozen** (reserved) at submit time — see `references/task-lifecycle.md`.
2. **Wait** — either poll `GET /api/v1/job/{job_id}` until `status` is `completed` or `failed`, **or** receive a POST to your `callback` URL when the job finishes. Pick one — see *Webhook vs polling* below.
3. **Fetch & store** — on `completed`, read `output.image_url` (primary) and `output.image_urls` (grid). **Download to permanent storage immediately** — result URLs expire (see `references/task-lifecycle.md`).

Auth is a single header on every request: `x-api-key: $LEGNEXT_API_KEY`. Base URL: `https://api.legnext.ai`. Task endpoints live under `/api/v1/{endpoint}`; account under `/api/account/`.

## Decisions to make before writing code

### Webhook vs polling
- **Webhook** (pass `callback` on submit): lower latency, no polling cost. Use when you have a publicly reachable HTTPS endpoint. Respond `2xx` fast; the canonical result is still `GET /api/v1/job/{job_id}` — fetch it rather than trusting only the webhook body. **Dedupe deliveries by `job_id`** (a callback may arrive more than once).
- **Polling** (`GET /api/v1/job/{job_id}`): use when there is no public endpoint — local dev, CLIs, batch jobs behind NAT. Poll every 5–10s. Most fast-mode images finish in under a minute, but complex/slow jobs can run ~5 minutes; cap your poll loop at ~10–15 min and treat that as a failure/timeout to investigate.

### Pin the model version explicitly
A prompt with **no `--v` flag renders as v7** — the API default is v7, not Midjourney's newest web default. The API never injects `--v` for you. If the version matters, **always pin it** (`--v 8.2`, `--v 7`, …). See `references/parameters.md` for the full version × parameter matrix.

### Credits: freeze → consume → release
- On submit, the estimated cost is **frozen** (reserved) from your balance (`meta.usage.frozen`).
- On success, the actual cost is **consumed** (`meta.usage.consume`, usually equals frozen).
- On failure, frozen credits are **released back**.
- A `402 Payment Required` means insufficient credits at submit. Check balance with `GET /api/account/balance` and surface a top-up prompt to the user; do not silently retry a 402.
- Cost scales with flags: `--hd` is 1.5× on v8.1/v8.2 (4× on v8), `--turbo` is 2×, v7 `--draft` is half. Absolute per-task pricing is on the docs site — don't hardcode it; read `meta.usage` from the response.

### Child tasks depend on their parent
`variation`, `upscale`, `reroll`, `remix`, `pan`, `outpaint`, `inpaint`, `edit` operate on a **parent task's image**. They take the parent's `job_id` + an `imageNo` (0–3, the grid index). They are bound to the parent that produced the base image — so:
- Submit the parent first, wait for `completed`, read `output.available_actions` to see which actions are valid on which image indices, then submit the child.
- Some edit actions (`edit`, `enhance`, `retexture`, `remove_background`, `upload_paint`) only work on base images that support them. If you get *"re-generate the base image and retry"* or *"this action is not supported for the selected image"*, regenerate the base image and retry — do not loop.
- `available_actions` is **advisory**: for some edit-derived images it can list an action that the backend then rejects. Handle that 4xx gracefully instead of hard-crashing.

### Content moderation is a user error, not a service error
Prompts or images may be rejected by upstream content moderation with messages like `community guidelines`, `community standards`, `sensitive words`, `content moderation`, `图片不符合社区规范`, `Prompt图片未通过审核`. These are **user errors** — fix the prompt; do not retry the same prompt. Likewise `insufficient quota` and `Too many tasks` are user-side.

## Retries & idempotency
- The API **does not dedupe by prompt**. Each `POST /api/v1/diffusion` creates a new job and freezes credits. If you retry a submit, you get a new job and pay for both.
- Make retries idempotent **on your side**: persist the `job_id` before considering the submit successful, and dedupe webhook deliveries by `job_id`.
- Retry only transient failures (`429`, `5xx`, network errors, timeouts) with exponential backoff. Never retry `400/401/402/403/422` — they won't succeed unchanged.

## When something looks like an outage
If failures spike across **many unrelated jobs at once** — especially with empty/generic `error.message` or a generic *"execution failed, please retry"* — suspect a transient upstream Midjourney/Discord outage rather than a bug in your integration. Back off and retry; these windows are usually bounded. A single job failing is almost always a prompt/param/credit problem — see `references/errors.md`.

## Reference files
- `references/parameters.md` — full parameter + version matrix, cost multipliers, and the version-specific gotchas (`--cref`/`--oref`/`--hd`/`--draft` rules). ⚠️ Hand-placed initial version; will be superseded by an auto-generated pipeline.
- `references/task-lifecycle.md` — job object shape, status values, polling/webhook mechanics, storage TTL, credits freeze/consume, child tasks.
- `references/errors.md` — error object shape, HTTP status table, error taxonomy with a concrete handling action per class.
- `references/production-checklist.md` — pre-flight checklist before shipping an integration.

## Full API reference
Always-current endpoint/parameter docs: https://docs.legnext.ai. Drop `https://docs.legnext.ai/llms-full.txt` into your editor for the whole spec in one file.
