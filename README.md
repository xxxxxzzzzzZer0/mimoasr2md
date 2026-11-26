# Audio → Text (mimoasr2md.html)

**English** | [简体中文](README.zh-CN.md)

A single-file web tool: open it in a browser by double-clicking, pick a few recordings, turn them into text, and save the results merged into one Markdown document.

> The single source of truth for the spec and hard parameters is [`docs/方案.md`](docs/方案.md) (written in Chinese). This file only covers *how to use* the tool; every concrete value (audio format, size and duration thresholds, segment length, concurrency, retry and backoff) is defined in §4 of that document and is deliberately not repeated here.
>
> **Status: v2.6 — bilingual UI on top of v2.5; v2.5 was verified against the live API with a real key (2026-11-26). v2.6 adds only the interface language layer, whose own checks are listed in `docs/验收清单.md`.**
>
> The page UI is bilingual (Chinese / English) and opens in Chinese; a button at the top right switches it.

## 1. What it is

- **One HTML file**: `mimoasr2md.html` is the entire program — no install, no build step, and no resources loaded from the network, so the interface still works offline.
- **Many recordings at once**: multi-select files, drag and drop, or pick a whole folder.
- **Merged export**: all results are merged in selection order into a single `.md` file, one section per recording, each with the file name, basic info and the transcript.
- **Bilingual UI**: Chinese and English, Chinese on first open; the button at the top right switches instantly without reloading, and the choice is remembered by the browser.
- **Recognition is done by the provider**: transcription calls Xiaomi MiMo's speech API (model `mimo-v2.5-asr`) with your own API key. **Your audio is uploaded to the provider**; apart from that the page sends data nowhere else.
- **Explicitly out of scope**: speaker diarization, timestamps or per-sentence timing, a server or local proxy, offline/local recognition, UI languages other than Chinese and English, and automatic browser-language detection.

## 2. Quick start

### 2.1 Open the page

Download [`mimoasr2md.html`](mimoasr2md.html) (on GitHub, open the file and use **Download raw file**), put it in any local folder, and double-click it (Edge / Chrome recommended). A `file://` URL in the address bar is expected; the page loads fully without a network connection.

### 2.2 Enter your API key

1. Paste your key into the **API Key** field (the input is a password field and never echoes the value in clear text).
2. Pasting strips leading/trailing whitespace, an accidentally included `Bearer ` prefix, and invisible characters. Format problems only produce a warning — the provider has the final say.
3. Once confirmed, the field becomes a read-only mask such as "configured ····last 4 (total length)". The mask assumes no fixed prefix; it only shows the last 4 characters and the length.
4. To double-check, "show for 3 seconds" temporarily reveals it, then re-masks automatically. There is no one-click "copy in clear" action.
5. **The key lives only in this page's memory**: refreshing or closing the tab discards it and you have to enter it again.
6. The endpoint (Base URL) and model name live under "Settings" and are prefilled. These non-sensitive settings are remembered; if storage fails, they simply apply to the current session only.

### 2.3 Select audio

- Drag files in, click "choose files" (multi-select), or click "choose folder".
- When picking a folder, files are filtered by extension and non-audio files are ignored; nothing outside your selection is read or uploaded.
- Re-adding the same file is de-duplicated by file name, size and modification time, with a notice.
- Each list row shows size, duration, processing method and status, and offers per-file retry, a look at that file's raw response JSON (redacted), and removal.

### 2.4 Start transcription

- Until a key is configured, the "start" button is disabled.
- Progress updates in each row only; there is no modal dialog. For long jobs, keep the page in the foreground (a background tab may be throttled by the browser — see §5).
- You can cancel at any time; finished results are kept.
- If one segment of a file ultimately fails, that file is marked "partially failed" with the missing segment numbers; a manual retry re-runs only the failed segments.

### 2.5 Preview

Click "preview" to see the exact text that will be saved: at the top, generation time, model, language setting, file count and success/failure counts, total audio duration and the estimated cost; below, one section per recording with size, duration, processing method, status, elapsed time and the transcript. **Failed files also appear in the document, with the reason stated.**

### 2.6 Save the Markdown

- Click "save md": Edge / Chrome opens the system **Save As** dialog so you can choose the folder and file name. If the browser does not support it or the call is denied, it falls back to a download in the browser's download folder (Firefox always takes this path).
- Output file name: type your own in the "output file name" field — Chinese characters and spaces are allowed (characters Windows forbids are replaced). Leave it empty for automatic naming, which follows the current UI language:
  - one recording → `<original audio name>_转写.md` (English UI: `<original audio name>_transcript.md`)
  - several recordings → `转写记录_YYYYMMDD_HHMMSS.md` (local time; English UI: `Transcript_YYYYMMDD_HHMMSS.md`)
- Merge order matches **the order you selected the files**; completion order does not matter.
- There is also a "copy" button that puts the whole document on the clipboard.
- Document options (under Settings): clearing "include metadata block" keeps only the title, section headings and transcripts (failed items still keep a one-line reason); "BOM" helps very old Notepad; "protect original formatting" (on by default) stops leading `---`, code fences and similar Markdown from breaking the section structure. Very long lines without line breaks are wrapped at sentence boundaries for readability only — the text itself is unchanged.
- Cancelling the save is not an error; the content stays in the preview area and can be saved or copied again.
- The document follows the UI language: the English UI produces the English template (English title and field labels), while audio file names and transcript text are never translated.

### 2.7 Other options

The language button at the top right, and the switches at the bottom of the page and under "Settings" (non-sensitive ones are remembered; if storage fails they apply to the current session only):

| Option | Meaning |
|---|---|
| Interface language | The button at the top right shows the target language (`EN` on the Chinese UI, `中文` on the English UI); switching is instant with no reload, the choice is remembered, and the first open is Chinese |
| Language | Auto / Chinese / English, auto by default; an explicit language reduces misdetection |
| Concurrency | Number of segment requests in flight; higher is faster but more likely to hit provider rate limits, lower is safer (see §4 of [`docs/方案.md`](docs/方案.md) for the range) |
| Silence-aware splitting | On by default: cut at silences to avoid chopping words in half; when off, cut at fixed-length boundaries |
| Streaming (experimental) | Off by default; the provider publishes no official streaming sample for ASR, so enabling it concatenates increments and is experimental |
| Protect original formatting | On by default; leading Markdown syntax in the transcript cannot break the document's sections |
| Include metadata block / BOM | See §2.6 |

## 3. Supported inputs

| Input format | How it is handled |
|---|---|
| `wav` / `mp3` | Uploaded as-is when conditions allow (keeps audio quality, avoids a re-encode); when over the per-request size limit or long in duration, converted to the format the API requires and split as needed |
| `m4a` / `aac` / `ogg` / `flac` / `webm` | Decoded in the browser → converted to the format the API requires → split as needed |
| `amr` / `wma` / `3gp` and others | **Not supported in this version**: the browser cannot decode them, so the page says "not supported by the browser, convert to mp3/wav first" and **sends no request and bills nothing** |

Notes:

- Splitting exists to get around the per-request size limit; it **does not reduce the total upload**. The audio format, segment length, size limit and duration threshold are in §4 of [`docs/方案.md`](docs/方案.md).
- Both a duration and a file-size threshold apply; files beyond them are rejected with a request to split them first. For very long recordings the UI shows a memory estimate — splitting beforehand is recommended.
- No codecs need to be installed; convert AMR/WMA to mp3 or wav with another tool first.

## 4. Security and privacy

- **The key never touches disk**: it exists only in page memory and is used to build the request header. It is not written to localStorage / sessionStorage / IndexedDB / cookies, and never appears in logs, the preview area, the exported Markdown, the raw JSON or any file. Refreshing discards it.
- **Logs and errors are redacted**: the key is replaced with `[KEY]` in UI logs, error messages and preview text — seeing `[KEY]` is expected.
- **No third-party requests**: there is no CDN, external script/style/font/icon or analytics, and no request goes anywhere except the endpoint configured under Settings.
- **The HTML is safe to forward**: the file contains no key or other credential; each colleague simply enters their own key.
- **Risks to be aware of**:
  - Audio is uploaded to the provider for recognition; a persistent notice at the top of the page shows the endpoint actually in use.
  - Use it within whatever terms you have agreed with the provider, and on a trusted device.
  - When you pick a folder, the file names in that folder are visible to the page (it only reads and uploads the audio you selected).
  - Cancelling does not undo requests that already completed and were billed; the cost line reflects actual usage.

## 5. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| "API key invalid" (401) | The key is incomplete or expired. Copy and paste it again and make sure there is no extra whitespace; if it still fails, check the provider console. |
| "Insufficient balance" (402) | The provider account is out of credit; top up and retry. |
| "Content rejected by the server" (421) | The provider blocked the content; try another recording or contact the provider. |
| "Format/size not valid" (400) | The page already re-split automatically and was still rejected; check what the provider returned, or convert the audio to wav/mp3 first. |
| Rate limited (429) | The page backs off and retries automatically; if it still fails, requests are too dense — wait and retry, or lower concurrency. |
| Region/risk control (403), not found (404), server error (500/503) | Region or account restriction, a wrong endpoint or model name, or a temporary provider outage; check Settings or retry later. |
| "Save md" downloads instead of showing a dialog | The browser does not support "Save As" (e.g. Firefox), so it fell back to a download; nothing is lost, and the content also stays in the preview area. |
| The document contains "possibly incomplete" | A segment hit the model's output length ceiling and stayed truncated even after one automatic binary re-split. Split very long recordings into pieces and transcribe them separately. |
| "No valid speech detected (likely silence or music)" | The segment returned empty text although the provider recorded a duration — usually silence or pure music. |
| File rejected with a request to split | It exceeds the duration or size threshold (thresholds in §4 of [`docs/方案.md`](docs/方案.md)). Split it or transcribe it in parts. |
| Progress stalls or suddenly slows | The browser throttles tabs that are backgrounded, minimized, locked or asleep. Keep the page in the foreground and avoid sleeping. |
| Cannot decode `.amr` / `.wma` | Not supported in this version; convert to mp3/wav first. Such files send no request and cost nothing. |
| Costs incurred after cancelling | Requests that completed before the cancel are billed as usual; the document's cost line reflects actual usage. |

## 6. Browser support

| Browser | Notes |
|---|---|
| Edge / Chrome (Windows) | Primary path, including "Save As" with a chosen location |
| Firefox | Works; saving falls back to a browser download |
| Safari / others | Untested; saving uses the download fallback |

Single file, works offline: the page opens and performs local decoding and splitting without a network connection; only the actual recognition calls need one.

## 7. Development and self-testing

- Offline self-test: open `mimoasr2md.html?mock=1` — the whole flow runs against the test stub embedded in the page and sends no network requests.
- Integration testing: a local CORS mock on `127.0.0.1:8787` and a static server on `127.0.0.1:8788`; both ports are exclusive and shut down after use.
- Real-device and mock acceptance are ticked off item by item in [`docs/验收清单.md`](docs/验收清单.md) (Chinese). The offline and mock test credential is always `FAKE_KEY_TEST_0000`; a real key belongs only in the page's input field and must never be written to any file, screenshot or report.
- Single source of truth for the spec: [`docs/方案.md`](docs/方案.md). Changing any hard parameter means changing §4 of that document first, then the code, in the same commit.

## 8. Repository contents and license

| Path | Contents |
|---|---|
| `mimoasr2md.html` | The only runtime deliverable — one file, double-click to use |
| `README.md` | This English README (the default view) |
| `README.zh-CN.md` | Chinese README |
| `docs/方案.md` | Single source of truth for the spec and hard parameters (Chinese) |
| `docs/验收清单.md` | Real-device / mock acceptance checklist (Chinese) |
| `LICENSE` | MIT |

- The repository contains no credentials; each user enters their own API key in their own browser, and it never leaves that page's memory.
- The development log (known defects and test records) is kept locally only and is not published with the repository.
