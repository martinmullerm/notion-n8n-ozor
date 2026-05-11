# Notion SOP → LMS Training Video (n8n + Ozor)

An n8n workflow that watches a Notion "Document Hub" database for recently edited SOPs (Standard Operating Procedures) and automatically regenerates the matching LMS training video with [Ozor AI](https://www.ozor.ai/) — then appends the fresh MP4 back to the original Notion page.

Goal: keep internal training videos in sync with the source-of-truth docs in Notion, without anyone manually re-rendering anything.

## Workflow at a glance

```
┌──────────────────┐   ┌────────────────────────┐   ┌────────────────────┐   ┌─────────────────┐
│ Every 15 minutes │ → │ Notion: most-recently  │ → │ Notion: get child  │ → │ JS: flatten     │
│  (Schedule)      │   │  edited SOP            │   │  blocks of page    │   │  blocks → text  │
└──────────────────┘   └────────────────────────┘   └────────────────────┘   └────────┬────────┘
                                                                                      │
        ┌─────────────────────────────────────────────────────────────────────────────┘
        ▼
┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐
│ Ozor: analyze page │ → │ Ozor: generate     │ → │ Ozor: export MP4   │ → │ Notion: append     │
│  → video plan      │   │  video from plan   │   │  (public 1080p URL)│   │  video into page   │
└────────────────────┘   └────────────────────┘   └────────────────────┘   └────────────────────┘
```

## Node-by-node

| # | Node | What it does |
|---|------|--------------|
| 1 | **Every 15 Minutes** (Schedule Trigger) | Fires the workflow on a fixed cadence. |
| 2 | **Notion — Recently Edited SOPs** | Reads the single most recently edited page from the `Document Hub` database. |
| 3 | **Get many child blocks** | Pulls up to 50 child blocks (paragraphs, headings, images) from that page by URL. |
| 4 | **Code in JavaScript** | Loops the blocks and builds one big prompt string: each `content` line + each `image.file.url`, prefixed with a brand-context sentence. |
| 5 | **Ozor — Analyze Notion Page** | Sends the prompt to Ozor AI, which produces a *video plan* (layout, tone, structure). Source URL is set to the company site for brand context. |
| 6 | **Ozor — Generate From Plan** | Executes that plan: assembles scenes, applies style, renders motion + text. |
| 7 | **Export a video** | Exports the finished project as a public 1080p MP4. `waitForExportSingle: true` blocks until render completes. |
| 8 | **Append a block** | Appends an image block (the MP4 URL) at the end of the original Notion SOP page. |

## Requirements

- An n8n instance (self-hosted or cloud) with the **Ozor community node** (`n8n-nodes-ozor`) installed.
- A **Notion API credential** with read + write on the target `Document Hub` database.
- An **Ozor API credential**.
- Outbound network access to the Notion API, Ozor API, and Google Cloud Storage (Ozor stores exported MP4s there).
- Typical run time: **5–10 minutes** per execution (the video export is the bottleneck). Adjust the schedule interval if you expect long renders to overlap.

## Setup

1. **Import the workflow** — in n8n, *Workflows → Import from File* → select `notion-sop-to-lms-video.json`.
2. **Reconnect credentials** — the import will show "Credentials missing" on the Notion and Ozor nodes. Open each and bind your own Notion / Ozor credentials.
3. **Point at your own database** — open the *Notion — Recently Edited SOPs* node and replace the database reference with your own `Document Hub` (or whatever you call it).
4. **(Optional) tweak the JS prompt** — node 4 hard-codes the prefix sentence (`"I have put my company url for context..."`). Edit it to match your brand voice.
5. **(Optional) adjust scope** — by default the workflow grabs the **single** most recently edited SOP per run (`limit: 1`). Increase if you want batch processing, but be mindful of render costs.
6. **Test once** with *Execute workflow* before activating. Once happy, flip the workflow to **Active**.

## Notes & gotchas

- The schedule (15 min) can overlap with a long render. Either widen the interval or add a "is workflow already running?" guard if that becomes a problem in production.
- Node 8 appends an **image block** with the MP4 URL. Notion will render this as an embedded video for direct `.mp4` URLs hosted on supported domains (GCS works).
- There is currently no "has this SOP already been refreshed since last edit?" check — every run will re-render the most recently edited page. If you don't want continuous re-rendering, add a property check (e.g., compare `last_edited_time` vs a `last_video_generated_at` property on the Notion page) before the Ozor nodes.

## What's in the JSON

The exported file contains node configuration, connections, and credential **reference IDs** only — no secrets. You will need to rebind credentials to your own Notion and Ozor accounts after import.
