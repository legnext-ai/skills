# Task Lifecycle

The Legnext API is asynchronous: submission returns immediately with a pending job, generation runs out-of-band, and you collect results by polling or webhook.

## Base URL & auth
- Base: `https://api.legnext.ai`
- Task endpoints: `POST/GET /api/v1/{endpoint}`
- Account: `GET /api/account/balance`
- Auth header on every request: `x-api-key: <YOUR_API_KEY>`

## Submit a task

`POST /api/v1/diffusion` (text-to-image). Other generation endpoints: `variation`, `upscale`, `reroll`, `blend`, `describe`, `shorten`, `pan`, `outpaint`, `inpaint`, `remix`, `edit`, `upload-paint`, `retexture`, `remove-background`, `enhance`, `video-diffusion`, `extend-video`, `video-upscale`, `enhance-upscale`.

### Request body (diffusion)
| Field | Type | Required | Description |
|---|---|---|---|
| `text` | string | yes | Prompt (1–8192 chars), including `--flags`. |
| `callback` | string | no | HTTPS webhook URL to POST to on completion. |

### Response (pending job)
```json
{
  "job_id": "789320d3-4b20-46d3-a804-6165a079176a",
  "model": "midjourney",
  "task_type": "diffusion",
  "status": "pending",
  "config": { "service_mode": "public", "webhook_config": { "endpoint": "", "secret": "" } },
  "input": null,
  "output": { "image_url": "", "image_urls": null, "seed": "" },
  "meta": {
    "created_at": "2025-12-17T11:08:36Z",
    "started_at": "0001-01-01T00:00:00Z",
    "ended_at": "0001-01-01T00:00:00Z",
    "usage": { "type": "point", "frozen": 80, "consume": 0 }
  },
  "detail": null,
  "logs": [],
  "error": { "code": 0, "raw_message": "", "message": "", "detail": null }
}
```

The `job_id` is your handle for everything that follows. `meta.usage.frozen` is the credits reserved at submit.

## Status values

| Status | Meaning |
|---|---|
| `pending` | Queued, waiting to be processed. |
| `staged` | Prepared and staged for processing. |
| `processing` | Currently generating. |
| `completed` | Done; results are in `output`. |
| `failed` | Failed; see `error`. |

Terminal states are `completed` and `failed`. Keep polling (or wait for the webhook) until you hit one.

## Wait for completion

### Option A — Poll
`GET /api/v1/job/{job_id}` with the `x-api-key` header. Poll every **5–10 seconds**. Most fast-mode images finish in under a minute, but complex/slow jobs can run **~5 minutes**. Cap your poll loop at ~10–15 min and treat a timeout as a failure to investigate (don't poll forever).

### Option B — Webhook
Pass `callback` on submit. When the job finishes, Legnext POSTs the job object to your URL. Recommendations:
- Respond `2xx` quickly; do slow work after acknowledging.
- **Dedupe by `job_id`** - a callback may be delivered more than once.
- **Fetch the canonical result** with `GET /api/v1/job/{job_id}` rather than acting on the webhook body alone; the GET is authoritative.
- Use HTTPS for the callback URL.

Pick one of A or B per job; both work. Polling is simplest for local/CLI/batch; webhooks are better for low-latency production.

## Completed job (result)

```json
{
  "job_id": "a616de55-...",
  "model": "midjourney",
  "task_type": "diffusion",
  "status": "completed",
  "output": {
    "image_url": "https://cdn.legnext.ai/temp/<...>.png",
    "image_urls": [
      "https://cdn.legnext.ai/mj/<job_id>_0.png",
      "https://cdn.legnext.ai/mj/<job_id>_1.png",
      "https://cdn.legnext.ai/mj/<job_id>_2.png",
      "https://cdn.legnext.ai/mj/<job_id>_3.png"
    ],
    "seed": "1904983176",
    "available_actions": {
      "edit": [0,1,2,3],
      "inpaint": [0,1,2,3],
      "outpaint": [0,1,2,3],
      "pan": [0,1,2,3],
      "remix": [0,1,2,3],
      "reroll": true,
      "upscale": [0,1,2,3],
      "variation": [0,1,2,3]
    }
  },
  "meta": {
    "created_at": "2025-10-14T02:22:04Z",
    "started_at": "2025-10-14T02:22:06Z",
    "ended_at": "2025-10-14T02:22:50Z",
    "usage": { "type": "point", "frozen": 80, "consume": 80 }
  },
  "error": { "code": 0, "raw_message": "", "message": "", "detail": null }
}
```

- `output.image_url` — primary result URL.
- `output.image_urls` — the grid (typically 4 images; a v8.1/v8.2 `--draft` returns 24, but only indices 0–3 are actionable).
- `output.seed` — the seed used; reuse it with the same prompt+version for reproducibility.
- `output.available_actions` — which child actions are valid, and on which image indices. **Use this to drive your UI/next-step logic** instead of hardcoding the action set.
- `meta.usage.consume` — credits actually consumed (usually equals `frozen`).

## Fetch & store

**Download result images to permanent storage immediately.** Result URLs on the Legnext CDN are temporary:

- Generated images are stored for **30 days**, then deleted.
- Operations on a generated image (upscale, variation, …) are valid for **3 days** after creation. After that, the parent job can no longer be acted on.

If your pipeline needs to upscale/variation a result later, do it within 3 days, and always persist the final image to your own bucket/disk.

## Credits: freeze → consume → release

- **Submit**: estimated cost is **frozen** (reserved) from your balance → `meta.usage.frozen`.
- **Success**: actual cost is **consumed** → `meta.usage.consume` (usually equals frozen).
- **Failure**: frozen credits are **released** back to your balance.

Check balance anytime with `GET /api/account/balance` (returns `available_credits`, `available_points`, `balance_usd`, `low_balance_alert`). On `402 Payment Required` at submit, surface a top-up prompt to the user; do not silently retry.

## Child tasks (operate on a parent image)

`variation`, `upscale`, `reroll`, `remix`, `pan`, `outpaint`, `inpaint`, `edit`, `enhance`, `retexture`, `remove_background`, `upload_paint` all act on a **parent task's image**. They take the parent's `job_id` + an `imageNo` (0–3) indicating which grid image to operate on.

Pattern:
1. Submit the parent (e.g. `diffusion`), wait for `completed`.
2. Read `output.available_actions` to see which actions are valid on which `imageNo`.
3. Submit the child with the parent's `job_id` and the chosen `imageNo`.

Child tasks are **bound to the parent** that produced the base image. Consequence: some edit actions only work on base images that support them. If a child returns *"re-generate the base image and retry"* or *"this action is not supported for the selected image"*, regenerate the base image (resubmit the parent) and retry the child — don't loop on the same base image. Also treat `available_actions` as advisory: for some edit-derived images it can list an action the backend then rejects; handle that 4xx gracefully.

## Full reference
- Submit (text-to-image): https://docs.legnext.ai/api-reference/image-generation/diffusion
- Get task: https://docs.legnext.ai/api-reference/task-management/get-task
- Balance: https://docs.legnext.ai/api-reference/account-management/get-balance
- Output storage: https://docs.legnext.ai/getting-started/output-storage
