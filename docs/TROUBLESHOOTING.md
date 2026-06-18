# X Article Publisher Troubleshooting

This document records failure modes seen in real publishing runs. Check this before changing code.

Chinese version: [TROUBLESHOOTING_CN.md](TROUBLESHOOTING_CN.md)

---

## Quick Triage

| Symptom | Likely cause | Fix |
|---|---|---|
| Feishu download fails with `no such host`, `WRONG_VERSION_NUMBER`, or `server gave HTTP response to HTTPS client` | Local proxy/DNS resolves `open.feishu.cn` to a fake IP | Fix proxy/DNS so `open.feishu.cn` uses real HTTPS |
| All videos appear at the article tail | Videos were not restored near Feishu block anchors | Rerun the latest `prepare_article_source.py` and check `video_download_errors` |
| Uploads stop around the 26th body media item | Observed X Articles body-media limit | Split the article or merge consecutive images |
| PNG upload does not create a media block | X editor ignored that PNG | Convert the image to JPG and retry |
| X draft opens as blank or Playwright cannot control it | Persistent profile is already held by Chrome | Close only the dedicated profile process, then reopen |
| Clicks fail after video upload | `Uploading media...` overlay is still active | Wait until upload state disappears |
| Browser closes or drifts during video upload | Video is too large or high-bitrate for stable X processing | Transcode to 1280px wide H.264/AAC and resume from the draft URL |
| Media count is correct but order is wrong | One item was inserted under a later anchor | Run preview anchor audit, delete only the misplaced item, and reinsert it |
| Feishu highlight block shows `Tip` in X | Feishu callout marker leaked into Markdown | Rerun source preparation or remove the marker while keeping quote content |

---

## Feishu OpenAPI TLS Or DNS Errors

Typical errors:

```text
lookup open.feishu.cn: no such host
ssl.SSLError: WRONG_VERSION_NUMBER
http: server gave HTTP response to HTTPS client
```

Check DNS first:

```bash
dig +short open.feishu.cn
```

If it returns `198.18.x.x`, the local proxy is likely using fake-ip DNS. Adjust proxy/DNS settings or make the command use the correct proxy path.

Check HTTPS:

```bash
curl -I https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal
```

Once HTTPS returns a normal response, rerun:

```bash
python ~/.codex/skills/x-article-publisher/scripts/prepare_article_source.py "https://your-domain.feishu.cn/docx/..."
```

---

## X Body Media Limit

In field tests, X Articles may silently stop accepting body media around `25` items:

- The file input accepts the file.
- No toast appears.
- No media block is added.
- `Uploading media...` does not appear.

This is usually not a network issue. Recommended handling:

1. Split into two X Articles.
2. Merge consecutive screenshots into a long image.
3. Keep the most important videos and remove low-value media.

---

## PNG Upload Does Nothing

If a PNG upload does not create a body media block, convert it to JPG:

```bash
python3 - <<'PY'
from PIL import Image
src = "/path/to/image.png"
dst = "/path/to/image.jpg"
Image.open(src).convert("RGB").save(dst, quality=95)
print(dst)
PY
```

Then retry with the JPG.

---

## Persistent X Profile Is Busy

Typical message:

```text
正在现有的浏览器会话中打开。
```

Close only the dedicated profile, not the user's daily Chrome profile:

```bash
pkill -f "$HOME/.codex/browser-profiles/x-articles"
```

Then reopen:

```bash
bash ~/.codex/skills/x-article-publisher/scripts/open_x_articles_browser.sh
```

---

## Video Upload Appears Stuck

X shows `Uploading media...` while processing video. Do not insert the next media item while this overlay exists.

Rules:

1. Upload one video at a time.
2. Large videos may take minutes.
3. If a media block is visible and no error is shown, keep waiting.
4. If there is no media block and no upload state, re-place the cursor at the anchor and upload again.

If the video is large or high-bitrate, transcode before retrying:

```bash
ffmpeg -y -i input.mov \
  -vf "scale='min(1280,iw)':-2" \
  -c:v libx264 -preset medium -crf 25 -pix_fmt yuv420p \
  -c:a aac -b:a 64k -movflags +faststart \
  output.x1280.mp4
```

Resume from the existing draft URL instead of creating a new draft.

---

## Count Looks Correct But Order Is Wrong

Do not rely only on editor-side media counts. Verify the preview page:

- `img[alt="Image"]` count should match body images.
- `video[aria-label="Embedded video"]` count should match body videos.
- For every source media item, the next visible media after its `after_text` anchor should have the expected type.

If one item is misplaced:

1. Delete only that media block in the editor.
2. Reinsert the missing media at the correct anchor.
3. Reinsert the misplaced media at its own anchor.
4. Re-run the preview audit.

---

## Feishu Callout Shows `Tip`

Feishu highlighted blocks can export as:

```markdown
> Tip
> Important highlighted text
```

or:

```markdown
> [!TIP]
> Important highlighted text
```

The skill removes the marker and preserves the highlighted text. If `Tip` still appears, rerun `prepare_article_source.py` with the latest version or remove only the marker line manually.

---

## Feishu Video Ordering Is Wrong

The current flow reads video file blocks from Feishu dump JSON and uses nearby text as an anchor.

Check the download output:

```json
{
  "video_tokens_found": 10,
  "videos_downloaded": 10,
  "videos_appended": 10,
  "video_download_errors": []
}
```

If `video_download_errors` is not empty, do not publish yet. Fix the anchor or manually verify video positions first.
