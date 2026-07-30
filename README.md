# legnext-ai/skills

Agent skill for integrating the [Legnext AI API](https://docs.legnext.ai) — Midjourney image and video generation via a REST API — into your project with a coding agent ([Claude Code](https://claude.com/claude-code), Codex, Cursor, OpenCode, and [70+ more](https://skills.sh)).

[![skills.sh](https://skills.sh/b/legnext-ai/skills)](https://skills.sh/legnext-ai/skills)

## Install

```bash
npx skills add legnext-ai/skills
```

This installs the `legnext` skill into your project's agent skill directory (e.g. `.claude/skills/`). Add `-g` for a global install, or `-a claude-code` to target a specific agent. See the [skills CLI docs](https://skills.sh/docs).

## What the skill does

It teaches your agent the **integration methodology** for Legnext — not an API-reference replay. When you ask your agent to "add Midjourney generation to my app" or "wire up the Legnext API", the skill activates and guides it through:

- Submitting `imagine`/`diffusion` tasks and reading the job object
- Choosing **polling vs webhooks** and implementing either
- Fetching and **persisting generated images** before their temporary delivery URLs expire
- Handling **credits** (freeze → consume → release) and `402` top-up flow
- Submitting **child tasks** (`variation`/`upscale`/`reroll`/`remix`/…) on a parent image
- **Content-moderation** and validation failures (user errors, not retries)
- **Retries & idempotency** (dedupe by `job_id`; the API does not dedupe by prompt)
- A per-error-class handling table and a production pre-flight checklist

## Layout

```
legnext/
├── SKILL.md                         # trigger + integration playbook (read first)
└── references/
    ├── parameters.md                # parameter + version matrix, cost multipliers
    ├── task-lifecycle.md            # submit → poll/webhook → fetch → store; credits; child tasks
    ├── errors.md                    # error taxonomy + per-class handling action
    └── production-checklist.md      # pre-flight checklist before shipping
```

## Use

After installing, just tell your agent in natural language, e.g.:

> I have `LEGNEXT_API_KEY` in my env. Add a Legnext image-generation endpoint to this app: submit a diffusion task, poll for the result, and save the image locally. Use v8.2 and a 16:9 aspect ratio.

The agent will follow the skill. Get an API key from the [Legnext dashboard](https://legnext.ai).

## Status of `parameters.md`

`references/parameters.md` is a **hand-placed initial version** mirrored from the public docs. It will be superseded by an auto-generated pipeline that derives the matrix from the backend spec. Until then, the live docs at <https://docs.legnext.ai> are authoritative.

## Full API reference

- Docs: <https://docs.legnext.ai>
- Whole spec in one file (drop into your editor): <https://docs.legnext.ai/llms-full.txt>

## License

MIT — see [LICENSE](./LICENSE).
