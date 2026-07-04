---
name: dip-pen-sketch-outline
description: Turn a photo into a rough old-school dip-pen / nib-and-India-ink sketch in pure BLACK INK with NO color (monochrome line art with ink hatching), on a blank paper background (original background removed). The "Philly sketch" look, outline-only. Use when the user says "dip pen sketch outline", "dip-pen-sketch-outline", "ink sketch no color", "black and white pen sketch", "philly sketch (no color / outline)", or asks to turn a photo into a rough black-ink line sketch. By Reblis.com.
---

# dip-pen-sketch-outline

Converts any photo into a **rough dip-pen ink** illustration: thin scratchy black India-ink nib lines (loose and messy, NOT polished), pure black-and-white with ink hatching for shading, no color at all, subject(s) on blank white sketchbook paper with the original background removed. Faces stay recognizable.

This is the no-color variant. For the same look WITH loose marker coloring, use **dip-pen-sketch-color**. For BOTH at once with matching lines, use **dip-pen-sketch-combo**.

## Requirements

- **[Claude Code](https://claude.com/claude-code)**
- **The fal.ai MCP server** connected, with credits. This skill runs **`fal-ai/nano-banana-2/edit`** — Google's **Nano Banana 2** image-editing model, served through fal.ai (strong at preserving facial likeness). **~$0.03–0.04 per image.**
- `curl` + ImageMagick (`convert`).
- Optional free alternative: the **nanobanana / Gemini MCP** (`gemini_edit_image`), same model family — but its free-tier quota is often exhausted, so fal is the reliable default.

## Inputs

Resolve the input in **Step 0**:
- **Local path** (`/tmp/foo.jpg`).
- **Direct image URL** (`Content-Type: image/*`) — pass straight to fal.
- **Social / page link** (Instagram, Facebook, Pinterest) — HTML pages, not images; Meta CDN links are signed/expiring and block hotlinking. Download locally first.

## Procedure

0. **Resolve the input to a usable image.**
   - Direct image URL fal can fetch? `curl -sIL "$URL" | grep -iE '^(HTTP/|content-type:)'`; if `200` + `image/*`, pass it straight to `image_urls` in Step 3 and skip Steps 1–2.
   - Otherwise download it: `curl -sL -A 'Mozilla/5.0' -o /tmp/_dippen_dl.jpg "$URL" && file /tmp/_dippen_dl.jpg`. If blocked (Instagram/Facebook), open the post in a browser, grab the real image src or screenshot it, save to disk. That file is `$SRC`.
   - Local path → `$SRC` directly.

1. **Downscale the source** (CMYK→sRGB; ~1024px). *(Skip if passing a direct URL through.)*
   ```bash
   convert "$SRC" -colorspace sRGB -resize 1024x -quality 82 /tmp/_dippen_src.jpg && identify /tmp/_dippen_src.jpg
   ```
   Optionally `Read` it to tailor the "keep recognizable" clause.

2. **Get a URL fal can fetch (`$IMG_URL`).** *(Skip for direct-URL passthrough.)*
   - **Upload to fal's CDN via REST (reliable default):** don't pass the image as base64 `data` through `mcp__fal-ai__upload_file` — a ~1024px JPEG encodes to ~190K base64 chars, which overflows a tool call and gets truncated. Use fal's REST upload with the same key the fal MCP server is configured with:
     ```bash
     resp=$(curl -s -X POST "https://rest.alpha.fal.ai/storage/upload/initiate" \
       -H "Authorization: Key $FAL_KEY" -H "Content-Type: application/json" \
       -d '{"file_name":"_dippen_src.jpg","content_type":"image/jpeg"}')
     upload_url=$(echo "$resp" | python3 -c "import sys,json;print(json.load(sys.stdin)['upload_url'])")
     IMG_URL=$(echo "$resp" | python3 -c "import sys,json;print(json.load(sys.stdin)['file_url'])")
     curl -s -o /dev/null -w '%{http_code}' -X PUT "$upload_url" -H "Content-Type: image/jpeg" --data-binary @/tmp/_dippen_src.jpg   # expect 200
     ```
   - **Your own web server:** copy the file into its docroot and use that public URL, e.g. `cp /tmp/_dippen_src.jpg /var/www/<your-site>/_dippen_src.jpg` → `https://<your-site>/_dippen_src.jpg` (delete it after the run).

3. **Run the edit** with `mcp__fal-ai__run_model`:
   - `endpoint_id`: `fal-ai/nano-banana-2/edit`
   - `input`:
     - `image_urls`: `[$IMG_URL]` (or the direct passthrough URL)
     - `resolution`: `"2K"`, `output_format`: `"jpeg"`, `aspect_ratio`: `"auto"` (override only on request)
     - `prompt`: **the locked prompt below**, with the "keep recognizable" clause adapted to the subject(s).

4. **Download the result** (clean up the uploaded/staged source if you used your own server):
   ```bash
   curl -s -o /tmp/<name>_dippen_outline.jpg "<RESULT_URL>" && identify /tmp/<name>_dippen_outline.jpg
   ```
   Then normalize the paper to pure white — generations come back on slightly different paper tones, and this makes every run deliver the same blank-white background (the corner-sampled flood only touches near-paper pixels, so the ink is safe):
   ```bash
   cd /tmp && f=<name>_dippen_outline.jpg
   bg=$(convert "$f" -format '%[pixel:p{3,3}]' info:)
   convert "$f" -fuzz 12% -fill white -opaque "$bg" "$f"
   ```

5. **`Read` the result**, report the saved path, offer tweaks (tighter crop, rougher/denser hatching, 4K).

## Locked prompt (outline / no color)

> Transform this photo into a ROUGH, loose dip-pen and India-ink line sketch. CRITICAL: keep the subject(s) clearly recognizable with their exact same face(s), expression(s), hair, facial hair, pose, sunglasses/accessories, and clothing. Linework: thin black India-INK lines from an old-school DIP PEN / steel nib, drawn fast, ROUGH and SCRATCHY and messy — loose energetic scribbled strokes, overlapping built-up scratchy hatching, uneven wobbly lines, some lines doubled or left incomplete and overshooting, occasional ink splatter dots, a quick raw gesture-sketch feel. Thin nib lines but NOT clean, NOT refined, NOT a polished illustration — deliberately rough and sketchy like a fast unfinished pen doodle. PURE BLACK INK ONLY — absolutely NO color, monochrome black-and-white line art, use ink cross-hatching and stippling for ALL shading and tone. Plain blank white sketchbook paper background — remove the original background entirely, just the figure(s) on empty paper.

## Notes

- The differentiator from the color skill: **no marker, monochrome only.** If any color sneaks in, re-run stressing "pure black ink, no color, black and white only."
- If output is too clean, re-run emphasizing "rougher, scratchier, wobbly, incomplete lines."
- Always adapt the "keep recognizable" clause to the actual photo for best likeness.
- **Want both an outline AND a matching color version with identical lines?** Use **dip-pen-sketch-combo**, which generates the colored sketch and *derives* the outline from it — the only method that keeps the linework identical stroke-for-stroke — and normalizes both backgrounds to the same paper color (two separate output images).
