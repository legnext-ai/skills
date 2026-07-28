# Errors & Handling

## Error object shape

Every job carries an `error` object. On a failed job it is populated; on success the fields are zero/empty.

```json
"error": { "code": 403, "raw_message": "...", "message": "...", "detail": null }
```

- `code` - matches the HTTP status code on the response.
- `message` - human-readable description (safe to show users).
- `raw_message` - upstream/original message (useful for debugging).
- `detail` - optional structured detail.

HTTP error responses (on submit, before a job is created) use the same shape:

```json
{ "code": 401, "message": "Failed to verify api key", "raw_message": "", "detail": null }
```

## HTTP status codes

| Code | Meaning | Action |
|---|---|---|
| `400` | Bad request - invalid params or malformed body. | Fix the request. A `400` on v8.1/v8.2 usually names a rejected flag (`--cref`, `--oref`, `--q`, `--turbo`, `--sd`). Remove the flag or switch version. |
| `401` | Unauthorized - bad/missing API key. | Check `x-api-key` and that the key is for the right environment. **Do not retry.** |
| `402` | Payment required - insufficient credits. | Surface a top-up prompt; check `GET /api/account/balance`. **Do not retry unchanged.** |
| `403` | Forbidden - sensitive content or permission denied. | Treat as content-moderation rejection; fix the prompt. **Do not retry the same prompt.** |
| `413` | Payload too large. | Reduce input size (e.g. image upload). |
| `422` | Unprocessable entity - validation failed. | Check parameter formats/values. |
| `429` | Rate limit exceeded. | Back off and retry (exponential). |
| `500` | Internal server error - temporary. | Retry with backoff. |

## Error taxonomy & per-class action

### Auth (`401`)
Bad or missing API key. Verify the key is set in the environment, sent in the `x-api-key` header, and belongs to the environment you're targeting. Not retriable.

### Quota / credits (`402`, `insufficient quota`)
Not enough credits to freeze. Check balance via `GET /api/account/balance`; prompt the user to top up. Not retriable without a top-up. Note: `insufficient quota` can also appear as a job-level error (the account ran out mid-processing) - same remedy.

### Rate / concurrency (`429`, `Too many tasks`, `error limit`)
You're submitting too fast or have too many in-flight jobs. Back off (exponential), then retry. `Too many tasks` is a concurrency cap - wait for in-flight jobs to finish before submitting more.

### Content moderation (`403`, `community guidelines`, `community standards`, `sensitive words`, `content moderation`, `图片不符合社区规范`, `Prompt图片未通过审核`)
The prompt or a referenced image was rejected by upstream content moderation. This is a **user error**, not a service outage. Fix the prompt (remove or rephrase sensitive terms, change reference images); **do not retry the same prompt**.

### Validation (`400`, `422`, `Invalid prompt format`, `invalid prompt parameter`, `无法识别的参数`, `initParams: invalid argument`)
The prompt/flags are invalid for the chosen version. Common causes:
- `--cref` on v7+ (use `--oref` on v7, or switch to `--v 6.1 --cref`).
- `--oref` or `--q` on v8.1/v8.2 (removed; rejected).
- `--turbo` on any v8.
- `--draft` on v8.0 (fails upstream; not supported).
- Bad flag syntax or out-of-range values.

Fix the flag/version combo. Not retriable unchanged.

### Action not supported on this image (`this action is not supported for the selected image`, `re-generate the base image and retry`)
A child task (edit/enhance/retexture/remove_background/upload_paint/variation/…) was submitted against a base image that doesn't support it. The base image's generation context doesn't allow that action. **Regenerate the base image** (resubmit the parent) and retry the child. Don't loop on the same base image. Remember `available_actions` is advisory and can over-advertise for some edit-derived images - handle this 4xx gracefully.

### Transient service / upstream (`500`, network errors, timeouts, empty `error.message`, generic `execution failed, please retry`)
Retriable. Use exponential backoff. A **sudden spike** of these across many unrelated jobs at once - especially with empty/generic `error.message` - usually means a transient upstream Midjourney/Discord outage rather than a bug in your code. Back off; these windows are typically bounded and self-recover. A single job failing this way is rare; re-check the prompt/params/credits first.

### Personalization code issues (`个性化代码`)
A `--p` personalization code problem. Re-test the code on the target version (personalization pools differ across versions).

## Retry policy

- **Retriable**: `429`, `5xx`, network errors, timeouts. Exponential backoff (e.g. 2s, 4s, 8s, …) with a cap.
- **Not retriable** (fix-then-resubmit, often as a **new** job): `400`, `401`, `402`, `403`, `413`, `422`, content-moderation, action-not-supported.
- The API does **not** dedupe by prompt. Retrying a submit creates a new job and bills again. Make retries idempotent on your side: persist `job_id` after a successful submit, and dedupe webhook deliveries by `job_id`.

## Full reference
- Quickstart error table: https://docs.legnext.ai/getting-started/quickstart
- Get task (error field): https://docs.legnext.ai/api-reference/task-management/get-task
