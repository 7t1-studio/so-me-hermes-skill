# Example — A thread, a first comment, and a TikTok post

## Part 1 — A thread with a first comment

> **User:** "Turn my three lessons from this week into an X thread, and put the link in a first comment."

**Agent's reasoning:**
1. `--content` is the HEAD post, so the first `--thread` is the first reply.
2. X supports both a chain and a first comment, so no warning is expected.
3. Each part must be 280 characters or fewer.

**Agent execution:**

```bash
so-me posts:create --platform TWITTER \
  -c "Three things I learned shipping this week:" \
  --thread "1. Ship the smallest version first. The second version is cheaper once real users tell you what is wrong." \
  --thread "2. Read the error message. All of it. It is usually the answer." \
  --thread "3. Write the test before the fix, or you never learn whether the fix worked." \
  --first-comment "Full write-up: https://so-me.studio/blog/three-lessons"
```

**Sample response (parsed):**

```json
{
  "id": "p_01H...",
  "socialMedia": "TWITTER",
  "postStatus": "PENDING",
  "firstCommentStatus": "pending",
  "warnings": []
}
```

**Reply to user:**

> Posted the head plus 3 replies to X. The link goes up as a comment under the last part
> once the chain publishes.

### If the platform does not support the comment

A first comment on TikTok, Pinterest or Reddit is **not** an error. The post publishes
without it and the response carries:

```json
{ "warnings": [{ "code": "FIRST_COMMENT_UNSUPPORTED", "platforms": ["TIKTOK"],
  "message": "First comment is not supported on TIKTOK. The post publishes there without the comment." }] }
```

Repeat that warning to the user and do not retry. A chain on a platform outside TWITTER,
THREADS, BLUESKY and MASTODON is a 400 — put the whole message in `--content` there.

---

## Part 2 — A TikTok post

> **User:** "Post this clip to TikTok."

**Agent's reasoning:** TikTok rejects every post that carries no privacy level, and only
the creator may choose one. Never guess it.

**Agent execution:**

```bash
# 1. Validate the clip BEFORE posting, because the agent generated it
curl -s -H "X-API-Key: $SOMESTUDIO_API_KEY" -H 'Content-Type: application/json' \
  -X POST https://api.so-me.studio/v1/media/validate \
  -d '{"fileIds":["<file-uuid>"],"targets":[{"socialMedia":"TIKTOK","postType":"VIDEO"}]}'
```

If an issue comes back, act on its `fix` string, re-render, and validate again:

```json
{ "code": "ASPECT_RATIO_INVALID",
  "fix": "TIKTOK video requires an aspect ratio between 9:16 and 1:1; file is 1920x1080 (1.78:1). Crop to 1080x1920 (rotate to vertical) or 1080x1080 (centre square crop)." }
```

Then read the creator's own options — the agent uses the `get_tiktok_creator_info` MCP
tool for this — and **ask the user**:

> This account allows **Public** or **Only me**. I recommend **Public to everyone**.
> Which one do you want? And should viewers be able to comment?

Only after the user answers:

```bash
so-me posts:create --platform TIKTOK --post-type VIDEO -m <file-uuid> \
  -c "behind the scenes" \
  --tiktok-privacy PUBLIC_TO_EVERYONE \
  --tiktok-allow-comments
```

## Notes for the agent

- **Never guess a TikTok privacy level.** Read the allowed levels, offer only those,
  recommend `PUBLIC_TO_EVERYONE`, and ask about comments before posting.
- A creator who disabled comments, duet or stitch forces the matching option off. The
  response carries a `TIKTOK_SETTING_FORCED` warning — repeat it to the user.
- Per-part chain limits: TWITTER 280 characters, THREADS 500, BLUESKY 300 **graphemes**,
  MASTODON 500. At most 24 parts.
- Publishing is best effort past the head. A failed part never fails the post; the
  response says how many parts published.
