# Fourth & Glory shorts: automated Buffer queue

This repo is the hand-off point between the shorts pipeline and the scheduler.

- **Videos**: release assets on [`shorts-batch-1`](https://github.com/jbeukelman52/fourth-and-glory-shorts/releases/tag/shorts-batch-1). The URL pattern is
  `https://github.com/jbeukelman52/fourth-and-glory-shorts/releases/download/shorts-batch-1/<file>`.
- **Queue**: [`queue.json`](queue.json), the list of videos in posting order, with captions per platform.
  Raw: `https://raw.githubusercontent.com/jbeukelman52/fourth-and-glory-shorts/main/queue.json`

The local pipeline (`tools/make_vo_short.sh` → `tools/shorts_vo/publish.mjs`) uploads every finished short and
adds or updates its queue entry automatically. A scheduler (a Claude routine using the Buffer connector) follows the rules below.

## queue.json

```jsonc
{ "version": 1, "updatedAt": "...", "videos": [
  { "id": "v24",
    "file": "FourthAndGlory_v24.mp4",          // release asset name
    "url": "https://github.com/.../FourthAndGlory_v24.mp4",
    "videoTitle": "FourthAndGlory_v24.mp4",    // Buffer asset title = the DUPLICATE KEY
    "status": "ready",                          // ready | review | hold | posted
    "durationSec": 24.2,
    "youtube":   { "title": "...", "text": "..." },
    "tiktok":    { "text": "..." },
    "instagram": { "text": "..." },
    "script": "...", "notes": "..." } ] }
```

- `ready`: may be scheduled.
- `review`: uploaded but not yet approved or captioned. **Never schedule it.**
- `hold`: blocked (see `notes`). **Never schedule it.**
- `posted`: all posts sent and the asset deleted.
- Order in the array = posting order (first `ready` entry goes first).

## Buffer facts

- Organization **"My organization"** `6af398e98f177ae78b5f2cab`, timezone **America/Los_Angeles**.
- Channels:
  - YouTube `6af3991062b70b94b0ea3278`
  - TikTok `6af3995462b70b94b0ea3550`
  - Instagram `6af399d462b70b94b0ea3a58`
- Free plan: **at most 10 scheduled posts** in total.

## Scheduler run (do these in order)

1. **Read** `queue.json` and `get_account` (use `currentTime` as "now").
2. **Inventory**: `list_posts` with status `scheduled`, `sending`, `sent` and `error` (all pages).
   - For each channel, build the set of `assets[0].video.title` values. Those videos are taken on that channel.
   - Note every scheduled `dueAt`.
   - Posts without a title were posted by hand (v01–v05). Ignore them, and never schedule an id that isn't in `queue.json`.
3. **Clean up**:
   - Find each entry whose `videoTitle` is **sent on all three channels**, with nothing still scheduled, sending or errored for it.
   - Delete its release asset (`gh release delete-asset shorts-batch-1 <file> -R jbeukelman52/fourth-and-glory-shorts -y`).
   - Set its `status` to `"posted"` in `queue.json`.
   - If GitHub write access isn't available, list these in the report instead.
4. **Fill the queue** while scheduled posts < 10:
   - Slots are **6:00 AM and 6:00 PM Pacific** every day. Start from the first slot more than 30 minutes after now.
   - For each slot in time order, and each channel with **no post at that slot**:
     - Pick a video in this order:
       - (a) the video another channel already has in this slot, if it's `ready` and not taken on this channel;
       - (b) otherwise the **first `ready` entry** in `queue.json` that is not taken on this channel.
     - Create the post (templates below) with `mode: customScheduled`, `dueAt` = the slot with the correct Pacific offset (−08:00 in winter, −07:00 in summer), and `schedulingType: automatic`.
     - Mark it taken right away, so one run can never double-book.
   - Keep the three channels in step: the same video goes to all three in the same slot whenever possible.
   - Stop at 10 scheduled posts, or when nothing `ready` is left.
5. **Report**:
   - What was scheduled (video, channel, time) and what was cleaned up.
   - Any `error` posts (don't retry them blindly).
   - How many `ready` videos remain. Warn when fewer than 4 remain.

## Post templates

Use the entry's text verbatim. The video asset is always
`{"video": {"url": <entry.url>, "metadata": {"title": <entry.videoTitle>}}}`.

| Channel | text | metadata |
|---|---|---|
| YouTube | `youtube.text` | `{"youtube": {"title": youtube.title, "categoryId": "20", "privacy": "public", "license": "youtube", "notifySubscribers": true, "madeForKids": false, "embeddable": true, "isAiGenerated": false}}` |
| TikTok | `tiktok.text` | `{"tiktok": {"isAiGenerated": false}}` |
| Instagram | `instagram.text` | `{"instagram": {"type": "reel", "shouldShareToFeed": true}}` |

## Hard rules

- **Never post the same video twice to the same channel.** The check is the asset title, across scheduled, sending, sent and error posts.
- One video per slot per channel. Slots are 6 AM and 6 PM Pacific only.
- Only `ready` entries. Never `review`, `hold` or `posted`.
- Never edit or delete posts that were already sent. Posted shorts are final.
- Fourth & Glory is a **football deckbuilder**. Never call it a "roguelike" in new text. Hashtag `#cardgame`, not `#roguelike`.
- Don't write captions. If a `ready` entry lacks text for a platform, skip that platform and report it.
