# Video-to-LinkedIn Post Generator (n8n)

Turn a long video into **5 LinkedIn post drafts** that match a target writing style (e.g., your CEO’s last 10 posts), fully automated with **n8n**.

This repo contains:
- An **n8n workflow template** (sanitized) you can import
- Setup steps + env/credentials checklist
- Webhook input format + troubleshooting

---

## What it does (end-to-end)

1. **Webhook** receives a `video_url`
2. **Gladia STT**:
   - submits the video URL for transcription
   - polls until status is `done`
   - extracts and cleans transcript text
3. **Google Sheets** reads last **N** style examples (default 10) from a sheet column called `Post`
4. **Gemini** generates **exactly 5** LinkedIn drafts in the same tone + language
5. Drafts are parsed/split into rows and **saved to a Google Sheet**
6. Workflow returns a webhook response

---

## Repo structure

```
video-to-linkedin-post-generator/
  workflows/
    n8n-workflow.template.json
  examples/
    request.json
  .gitignore
  LICENSE
  README.md
```

---

## Prerequisites

- n8n (cloud or self-hosted)
- Gladia API key (for speech-to-text)
- Google AI Studio / Gemini API key
- Google Sheets OAuth connected in n8n

---

## Quick start (import the workflow)

1. Open **n8n → Workflows → Import from file**
2. Import: `workflows/n8n-workflow.template.json`
3. In n8n, open these nodes and configure:

### A) Gladia nodes
- **"STT - Gladia Submit"**: set header `x-gladia-key`
- **"STT - Gladia Poll"**: set header `x-gladia-key`

### B) Gemini node
- **"Gemini - Generate 5 Drafts"**: set header `x-goog-api-key`
- Keep `Content-Type: application/json`

### C) Google Sheets nodes
- **"Google Sheets - Read CEO_Posts"**:
  - set your **style sheet** (CEO posts)
  - ensure there is a column named **Post**
- **"Save drafts to Google Sheets"**:
  - set your **output sheet**
  - recommended columns: `created_at, Language, Angle, Draft, Status`

> Tip: You can keep the mapping as “Auto-map input data”, but your sheet should have matching column names.

---

## Webhook usage

**Webhook path** (as in the template):  
`POST /webhook/content-repurpose`

### Request body
Send JSON like:

```json
{
  "audio_url": "https://your-video-url.mp4",
  "max_posts": 10
}
```

- `audio_url`: Your public video URL (Google Drive “direct download or youtube video link ” link also works if public)
- `max_posts`: How many recent style posts to use (default 10)

### Example cURL
```bash
curl -X POST "https://YOUR_N8N_HOST/webhook/content-repurpose" \
  -H "Content-Type: application/json" \
  -d '{"audio_url":"https://example.com/video.mp4","max_posts":10}'
```

---

## Notes about Google Drive video URLs

If you use a Drive link, prefer a **direct download** format like:

- `https://drive.google.com/uc?export=download&id=FILE_ID`

And make sure the file is **publicly accessible**, otherwise Gladia can’t fetch it.

---

## Common issues & fixes

### 1) Gladia returns no transcript / empty `full_text`
- Increase the wait time (node “Wait 20s” → 40–60s for longer videos)
- Keep polling until `status == done`

### 2) Gemini returns text instead of JSON
This workflow already includes a parser that:
- removes ```json fences
- tries to extract JSON from text
- falls back to splitting drafts by headings

If you want **strict JSON**, update the Gemini prompt to:
- “Return ONLY JSON array with fields: id, post”

### 3) Google Sheets is empty even though workflow ran
- Confirm your output sheet has headers that match the fields:
  - `created_at`, `Language`, `Angle`, `Draft`, `Status`
- Or change the Sheets node from Auto-map to manual mapping.

### 4) Workflow loops forever on transcription
- Gladia may return statuses like `processing`, `queued`
- Ensure the IF node checks the correct status field (already handled in the “Extract transcript_clean” node)

---

## Security

The workflow template in this repo is **sanitized** (no real API keys).  
Never commit real keys into GitHub.

---

## License

MIT
