# YouTube Summary Skill

A Claude skill that turns a YouTube link into a short narrative summary, written in the video's original language.

## What it does

Paste a `youtube.com` or `youtu.be` link and ask for a summary (or just paste the link) — Claude will fetch the video's transcript and return a brief, fluent summary in the language the video is actually spoken in, regardless of what language you're chatting in.

## How it gets the transcript

There's no reliable public API for pulling a YouTube transcript directly, so this skill tries a chain of methods, falling back automatically if one doesn't work:

1. **A third-party transcript tool, via Claude in Chrome** (primary). Claude navigates to a free web tool (e.g. [Tactiq](https://tactiq.io)), pastes in the video URL, and reads back the transcript. This is the most reliable method — it doesn't run into YouTube's own automation detection.
2. **Directly on the YouTube page, via Claude in Chrome** (backup). Claude expands the video description and opens the built-in transcript panel itself. This sometimes gets blocked by YouTube, which can silently reject the request for browsers it detects as automated — the skill checks for this and falls back quickly instead of hanging.
3. **A plain `web_fetch` of the video page** (light fallback). Rarely returns a transcript, but occasionally picks up a usable description or chapter list.
4. **Asking you to paste it manually** (last resort, and the only method guaranteed to work every time). Instructions: under the video, click "...more" to expand the description, then "Show transcript", and copy the text.

## Requirements

- Works in [claude.ai](https://claude.ai) (or any Claude client that supports skills).
- The automated methods (1 and 2) require the **Claude in Chrome** browser extension to be installed and connected. Without it, the skill falls straight to the `web_fetch` attempt and then to asking you to paste the transcript — it still works, just without the automation.

## Installation

1. Download this repository (or just `SKILL.md`) as a `.zip`.
2. In claude.ai: **Settings → Capabilities** (Free/Pro/Max) or **Organization settings → Skills** (Team/Enterprise) — make sure code execution is enabled.
3. Go to **Customize → Skills** and upload the `.zip`.
4. Toggle the skill on.

## Known limitations

- YouTube can reject transcript requests made by an automated browser with an unhandled error, which makes its own transcript panel hang indefinitely instead of showing an error. This is a limitation on YouTube's side, not something this skill can reliably work around — which is why the third-party tool is the primary method rather than YouTube itself.
- Auto-dubbed videos are more prone to failures in the direct YouTube panel.
- If no transcript is available anywhere (not even inside YouTube), the skill will say so rather than invent content.

## License

MIT — use, modify, and share freely.
