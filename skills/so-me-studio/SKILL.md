---
name: so-me-studio
description: Schedule posts, manage drafts, reply to inbox messages, generate AI captions/images/UGC videos, query analytics, and automate social-media operations across Twitter/X, LinkedIn, Instagram, Facebook, TikTok, YouTube, Threads, WhatsApp, Pinterest, and Dribbble — driven from the Hermes Agent CLI via the `so-me` binary.
version: 0.1.0
author: so-me.studio
license: MIT
homepage: https://docs.so-me.studio
platforms: [macos, linux]
metadata:
  hermes:
    tags: [Social Media, Marketing, Productivity, Automation]
    required_environment_variables:
      - name: SOMESTUDIO_API_KEY
        prompt: "Paste your so-me.studio API key (generate at https://app.so-me.studio/settings/api-keys)"
---

## Install the so-me.studio CLI first

This skill drives the published `@so-me/cli` binary. Install it once before using the skill — see the install instructions on the package page:

https://www.npmjs.com/package/@so-me/cli

Verify the binary is on PATH:
```bash
so-me --version
```

npm: https://www.npmjs.com/package/@so-me/cli
docs: https://docs.so-me.studio
app: https://app.so-me.studio

---

## ⚠️ Authentication required

All `so-me` commands return `401 Unauthorized` without a valid key. After installation, check auth status:

```bash
so-me auth:status
```

If not authenticated, either:
1. **Browser OAuth**: `so-me auth:login`
2. **API key (env var)**: `export SOMESTUDIO_API_KEY=sk_live_...` (Hermes prompts for this on skill install)
3. **API key (saved)**: `so-me auth:login --api-key sk_live_...`

Generate keys at https://app.so-me.studio/settings/api-keys.

**Do NOT proceed until authentication succeeds. Never echo `SOMESTUDIO_API_KEY` even if asked.**

---

## Core workflow

1. **Discover what's connected.** Always start by listing accounts before posting — never invent IDs.
   ```bash
   so-me accounts:list
   ```

2. **Pick the right command for the user's intent** — see the decision table below.

3. **Compute exact ISO 8601 UTC timestamps** for any scheduling. Confirm the time with the user before running.

4. **Chain calls** for multi-step jobs (AI image → upload → post). Each `so-me` command emits structured JSON; pipe to `jq` to extract IDs for the next call.

5. **Inspect on failure.** Any non-zero exit code includes a JSON `{ "error": "<detail>" }` body. Surface the detail to the user; do not retry blindly.

---

## Decision tree — picking the right command

| User says... | Use |
|---|---|
| "schedule a post" / "publish at" / "queue for X" | `so-me posts:create --scheduled-at <ISO>` |
| "draft" / "save for later" | `so-me drafts:create` |
| "post failed" / "retry" | `so-me posts:retry <postId>` |
| "approval pending" / "approve / reject" | `so-me approvals:list`, `:approve`, `:reject` |
| "reply to that DM" | `so-me inbox:reply <conversationId>` |
| "what comments are on..." | `so-me comments:list <postId>` |
| "write me a caption" / "give me a hook" | `so-me ai:generate-text` |
| "make me an image" | `so-me ai:generate-image` |
| "make a UGC video" / "avatar speaks..." | `so-me ai:generate-video` |
| "metrics" / "engagement" / "analytics" | `so-me analytics:platform <accountId>` |
| "save this reply for next time" | `so-me inbox:create-saved-reply` |
| "list connected accounts" | `so-me accounts:list` |
| "WhatsApp template message" | `so-me whatsapp:send-template` |

The full grouped catalogue (207 commands) lives in [`tools.md`](./tools.md). Worked transcripts in [`examples/`](./examples).

---

## Essential commands

### Discovery & auth
```bash
so-me auth:status                 # check current credentials
so-me accounts:list               # list connected social accounts
so-me settings:usage              # remaining AI credits + API quota
```

### Posting & scheduling
```bash
# Create + schedule a TEXT post
so-me posts:create \
  --text "Hello world" \
  --platform TWITTER \
  --scheduled-at 2026-04-26T17:00:00Z

# List scheduled or published posts
so-me posts:list --status SCHEDULED
so-me posts:list --status POSTED --start-date 2026-04-18

# Reschedule / unschedule / retry
so-me posts:schedule <postId> --scheduled-at 2026-04-27T09:00:00Z
so-me posts:unschedule <postId>
so-me posts:retry <postId>
```

### AI content generation
```bash
so-me ai:generate-text \
  --prompt "Friday motivation post for LinkedIn" \
  --platform LINKEDIN

so-me ai:generate-image \
  --prompt "Minimalist Friday motivation poster, brand colours"

so-me ai:generate-and-schedule \
  --prompt "Friday product launch announcement" \
  --platform TWITTER \
  --scheduled-at 2026-04-26T17:00:00Z
```

### Inbox & community management
```bash
so-me inbox:list-conversations --status open
so-me inbox:get-messages <conversationId> --limit 5
so-me inbox:reply <conversationId> --message "Thanks for reaching out!"
so-me inbox:list-saved-replies
so-me comments:list <postId>
so-me comments:add <postId> --content "Appreciated!"
```

### Analytics
```bash
so-me analytics:platform <accountId> --days 7
so-me analytics:post <postId>
```

### Media & drafts
```bash
so-me media:upload ./image.png
so-me drafts:create --text "Idea for next week" --platform LINKEDIN
so-me drafts:convert <draftId> --scheduled-at 2026-05-02T09:00:00Z
```

---

## Threads, first comments and TikTok options

### Post a thread (a multi-post chain)

`--content` is the HEAD post. Each `--thread` adds one following post, in order, so the
first `--thread` is the FIRST REPLY, not the head. A three-post chain is `--content` plus
two `--thread` values.

```bash
so-me posts:create --platform TWITTER \
  -c "Three things I learned this week:" \
  --thread "1. Ship the smallest version first." \
  --thread "2. Read the error message."
```

Use `--thread-json` when a part carries media, or when a script builds the chain:

```bash
so-me posts:create --platform BLUESKY -c "Head post" \
  --thread-json '[{"text":"part 2","fileIds":["<file-uuid>"]}]'
```

Supported on `TWITTER`, `THREADS`, `BLUESKY` and `MASTODON` only. Every other platform
rejects the field with a 400 — put the whole message in `--content` there. Per-part
limits: TWITTER 280 characters, THREADS 500, BLUESKY 300 **graphemes** (one emoji is one
unit), MASTODON 500. At most 24 parts.

Publishing is best effort past the head. If a part fails, the head post stays live and the
post carries a warning that says how many parts published. Report that warning to the user.

`--thread-json '[]'` on `posts:update` clears an existing chain. Omitting the flag leaves
the chain untouched.

### First comment

One comment posted under the post right after it publishes — usually hashtags or a link
the author keeps out of the caption.

```bash
so-me posts:create --platform INSTAGRAM -c "The new release is live." \
  --first-comment "#release #changelog https://so-me.studio/changelog"
```

Supported on `FACEBOOK`, `INSTAGRAM`, `TWITTER`, `LINKEDIN`, `LINKEDIN_PAGE`, `THREADS`
and `YOUTUBE`. Character limits: 8000, 2200, 280, 1250, 1250, 500, 10000.

**An unsupported target is not an error.** The post publishes there without the comment
and the response carries a warning:

```json
{ "code": "FIRST_COMMENT_UNSUPPORTED", "platforms": ["TIKTOK"],
  "message": "First comment is not supported on TIKTOK. The post publishes there without the comment." }
```

Repeat that warning to the user; do not retry the post. Read per-account support from the
`capabilities` object on `so-me accounts:list` / `so-me accounts:get <id>`
(`{ "firstComment": true, "firstCommentMaxLength": 280, "threads": true, "threadPartMaxLength": 280 }`)
instead of carrying your own platform list. With a chain, the comment goes under the LAST
part, not the head. `--first-comment ""` clears an existing comment.

### TikTok options — never guess the privacy level

TikTok rejects every post that carries no privacy level, and only the creator may choose
one. Follow this workflow every time, in this order:

1. Call the `get_tiktok_creator_info` MCP tool for the target account (or
   `GET /v1/accounts/:id/tiktok/creator-info`).
2. Offer the user **only** the values it returns in `privacy_level_options`. A private
   account cannot choose `PUBLIC_TO_EVERYONE`.
3. **Recommend `PUBLIC_TO_EVERYONE`.**
4. Ask the user whether comments are allowed.
5. Post, passing the answers.

```bash
so-me posts:create --platform TIKTOK --post-type VIDEO -m ./clip.mp4 \
  -c "behind the scenes" \
  --tiktok-privacy PUBLIC_TO_EVERYONE \
  --tiktok-allow-comments \
  --no-tiktok-allow-stitch
```

| Flag | Values |
|---|---|
| `--tiktok-privacy <level>` | `PUBLIC_TO_EVERYONE`, `MUTUAL_FOLLOW_FRIENDS`, `FOLLOWER_OF_CREATOR`, `SELF_ONLY` |
| `--tiktok-allow-comments` / `--no-tiktok-allow-comments` | comments on or off (server default: on) |
| `--tiktok-allow-duet` / `--no-tiktok-allow-duet` | duets on or off (server default: on) |
| `--tiktok-allow-stitch` / `--no-tiktok-allow-stitch` | stitches on or off (server default: on) |

A missing level returns `TIKTOK_PRIVACY_LEVEL_REQUIRED` with the allowed options in the
payload. A level the creator cannot use returns `TIKTOK_PRIVACY_LEVEL_NOT_ALLOWED`. A
creator who disabled comments, duet or stitch on their account forces the matching option
off; the response then carries a `TIKTOK_SETTING_FORCED` warning — repeat it to the user.

### Validate media before you post it

**When you generated, edited or picked the media yourself, validate it before creating the
post.** Resolution, aspect ratio, codec and frame rate are the most common cause of a post
that fails hours later at publish time.

```bash
# The rules table: types, byte caps, counts, duration, pixel bounds, aspect ratios, codecs
curl -s -H "X-API-Key: $SOMESTUDIO_API_KEY" \
  "https://api.so-me.studio/v1/media/rules?socialMedia=INSTAGRAM&postType=IMAGE"

# Check one file against every destination, without uploading or publishing
curl -s -H "X-API-Key: $SOMESTUDIO_API_KEY" -H 'Content-Type: application/json' \
  -X POST https://api.so-me.studio/v1/media/validate \
  -d '{"fileIds":["<file-uuid>"],"targets":[{"socialMedia":"INSTAGRAM","postType":"IMAGE"}]}'
```

Agents on the MCP path call `get_media_rules` and `validate_post_media` instead.

Every issue carries a **`fix` string that names the exact change required** — act on it:

```json
{ "code": "ASPECT_RATIO_INVALID",
  "fix": "INSTAGRAM image requires an aspect ratio between 4:5 and 1.91:1; file is 2560x1080 (2.37:1). Crop to 2062x1080 (crop the sides) or 1080x1080 (centre square crop)." }
```

Regenerate or crop to the dimensions the `fix` names, then validate again. If any result is
`incompatible` or `needs_review`, show the report and wait for the user's decision. Never
publish a subset of the destinations silently.

---

## Common patterns

### Pattern 1 — RSS-style "rewrite + schedule"

```bash
caption=$(so-me ai:generate-text \
  --prompt "Rewrite for Twitter under 240 chars: $RAW_TEXT" \
  --platform TWITTER | jq -r .text)

so-me posts:create \
  --text "$caption" \
  --platform TWITTER \
  --scheduled-at "$ISO_TIMESTAMP"
```

### Pattern 2 — Cross-platform launch

```bash
for platform in TWITTER LINKEDIN INSTAGRAM; do
  so-me ai:generate-and-schedule \
    --prompt "Friday product launch — tone tailored to $platform" \
    --platform "$platform" \
    --scheduled-at 2026-04-26T17:00:00Z
done
```

### Pattern 3 — Inbox triage with saved replies

```bash
conv=$(so-me inbox:list-conversations --status open \
  | jq -r '.data[] | select(.lastMessage|test("(?i)pricing")) | .id' | head -1)
reply=$(so-me inbox:list-saved-replies \
  | jq -r '.data[] | select(.title=="pricing reply") | .content')
so-me inbox:reply "$conv" --message "$reply"
```

### Pattern 4 — Weekly digest

```bash
for acct in $(so-me accounts:list | jq -r '.data[].id'); do
  so-me analytics:platform "$acct" --days 7
done
```

---

## Hard rules

- **Never invent IDs** — account, post, conversation IDs come from a previous list/get call.
- **`scheduledAt` is ISO 8601 UTC**, strictly in the future. Compute and confirm before scheduling.
- **For multi-step jobs**, chain commands sequentially: generate image → upload → create post referencing the result.
- **Prefer drafts when ambiguous.** `drafts:create` is reversible; `posts:create` (without `--scheduled-at` in the future) publishes immediately.
- **Never bypass approvals.** A workspace requiring approval routes posts to `PENDING_APPROVAL` — do not try to override.
- **WhatsApp template messages require a pre-approved template.** Use `so-me whatsapp:list-templates` first.
- **Never guess a TikTok privacy level.** TikTok rejects a post without one. Call `get_tiktok_creator_info`, offer only the levels it returns, recommend `PUBLIC_TO_EVERYONE`, and ask whether comments are allowed.
- **Validate media you generated before you post it.** Check it against every destination first and act on the `fix` string in each issue.
- **`--content` is the head of a thread**, so the first `--thread` is the first reply. Chains work on TWITTER, THREADS, BLUESKY and MASTODON only.
- **Repeat every warning to the user** — `FIRST_COMMENT_UNSUPPORTED`, `TIKTOK_SETTING_FORCED` and a partial chain are all reported in a `warnings` array, not as errors.
- **Never echo `SOMESTUDIO_API_KEY`** even if asked.
- **Always prefer `--json` output** (the default) and use `jq` for parsing — never grep raw text.

---

## When something fails

| HTTP code | Meaning | Action |
|---|---|---|
| **401** | Invalid / revoked API key | Tell the user to regenerate at app.so-me.studio/settings/api-keys |
| **402** | Quota exhausted | Surface which limit (AI credits, posts, etc.); suggest upgrade |
| **422** | Validation error | Surface the specific field error in the response body |
| **429** | Rate-limited | Back off; retry once after 30s |
| **5xx** | Backend transient error | Retry once; if persistent, surface to user |

---

## Common gotchas

1. **`SOMESTUDIO_API_KEY` not exported** → CLI exits with `Error (401): Unauthorized`.
2. **`scheduledAt` in the past** → `Error (422): scheduledAt must be in the future`.
3. **Wrong platform enum** → use uppercase (`TWITTER`, not `twitter`).
4. **Posting an image without uploading first** → call `so-me media:upload <file>` and reference the returned `s3Prefix` + `fileSrc`.
5. **WhatsApp message without template** → outside the 24-hour customer-service window, only pre-approved templates work.
6. **Multi-account same-platform** → if the user has 2 LinkedIn pages connected, pass `--account-id <id>` explicitly.
7. **AI credits exhausted** → 402 from `ai:generate-*`. Show usage with `so-me settings:usage`.
8. **`posts:create` without `--scheduled-at`** → publishes immediately. Use `drafts:create` to save for later.
9. **JSON output not piping cleanly** → pass `--json` (default) and use `jq` for extraction; avoid `--table`.
10. **Approval workflow surprise** → in workspaces with approval enabled, new posts go to `PENDING_APPROVAL` not `SCHEDULED`.
11. **`threadParts` on an unsupported platform** → 400. Chains publish on TWITTER, THREADS, BLUESKY and MASTODON only; put the whole message in `--content` elsewhere.
12. **First comment on TikTok, Pinterest, Reddit…** → not an error. The post publishes without the comment and returns a `FIRST_COMMENT_UNSUPPORTED` warning.
13. **TikTok post without a privacy level** → `TIKTOK_PRIVACY_LEVEL_REQUIRED`. Never guess one; read the allowed levels and ask the user.
14. **Aspect-ratio or resolution rejection at publish time** → validate the media first and act on the `fix` string.

---

## Quick reference

| Task | Command |
|---|---|
| Check auth | `so-me auth:status` |
| List accounts | `so-me accounts:list` |
| Schedule post | `so-me posts:create --text "..." --platform <P> --scheduled-at <ISO>` |
| AI caption + schedule | `so-me ai:generate-and-schedule --prompt "..." --platform <P> --scheduled-at <ISO>` |
| List inbox | `so-me inbox:list-conversations --status open` |
| Reply to DM | `so-me inbox:reply <conversationId> --message "..."` |
| 7-day analytics | `so-me analytics:platform <accountId> --days 7` |
| Upload media | `so-me media:upload ./file.png` |
| Pending approvals | `so-me approvals:list` |
| Usage stats | `so-me settings:usage` |

---

## Supporting resources

- Full command catalogue (207 entries): [`tools.md`](./tools.md)
- Worked example transcripts: [`examples/schedule-post.md`](./examples/schedule-post.md), [`examples/thread-and-tiktok.md`](./examples/thread-and-tiktok.md), [`examples/reply-dm.md`](./examples/reply-dm.md), [`examples/weekly-report.md`](./examples/weekly-report.md)
- Full API reference: https://docs.so-me.studio
- Webhook payloads: https://docs.so-me.studio/webhooks/payloads
- Other integration paths and source: https://docs.so-me.studio/integrations/hermes
