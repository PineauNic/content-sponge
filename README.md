# Content Harvest

A Claude Code skill that turns social media content into structured, searchable knowledge. Send it a URL from any major platform — YouTube, Instagram, TikTok, X/Twitter, LinkedIn, Reddit, podcasts, or articles — and it extracts the content, finds mentioned resources (repos, tools, skills, MCPs), auto-categorizes everything, saves it to a local knowledge base, and optionally acts on instructions like "install this" or "implement this."

## Install

Copy the skill file into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/content-harvest
cp skill.md ~/.claude/skills/content-harvest/skill.md
```

Claude Code will automatically detect and load the skill.

## Dependencies

### Required

```bash
# yt-dlp — downloads video/audio from YouTube, Instagram, TikTok, X, and more
brew install yt-dlp        # macOS
pip install yt-dlp          # or via pip

# ffmpeg — extracts audio from video for transcription
brew install ffmpeg         # macOS
sudo apt install ffmpeg     # Linux

# Python 3 — text processing (likely already installed)
python3 --version
```

### Recommended

```bash
# OpenAI Whisper — local audio transcription (for reels, TikToks, podcasts)
pip install openai-whisper

# GitHub CLI — for cloning repos and verifying GitHub resources
brew install gh             # macOS
sudo apt install gh         # Linux
```

### Optional

- **Playwright MCP** — browser automation for LinkedIn scraping, login-walled content, and fallback extraction
- **Gmail MCP** — search email for DM-gated / comment-gated resources
- **Whisper MCP** — if you prefer an MCP-based whisper server over local install
- **reader / trafilatura** — enhanced article extraction (curl fallback exists)

### Instagram Cookie Setup (one-time)

Instagram downloads require browser cookies. Run once:

```bash
# Option A: Use cookies from your browser (Chrome example)
yt-dlp --cookies-from-browser chrome "https://www.instagram.com/reel/test" --skip-download

# Option B: Export cookies manually to a file
yt-dlp --cookies /path/to/cookies.txt "https://www.instagram.com/reel/test" --skip-download
```

## Usage Examples

### Basic — Process a single URL

```
/content-harvest https://www.youtube.com/watch?v=abc123
```

### Save without action

```
Save this: https://www.instagram.com/reel/xyz789
```

### Install a resource from content

```
Install this: https://www.youtube.com/watch?v=abc123
```

### Batch — Multiple URLs at once

```
/content-harvest
https://www.youtube.com/watch?v=abc123
https://www.tiktok.com/@creator/video/456
https://x.com/user/status/789
```

### Search saved content

```
What was that reel about Claude skills?
Find my saved content about automation
```

### Summarize a topic

```
Summarize everything I've saved about AI tools
```

### No URL — describe it

```
I saw a reel about building Claude MCP servers, I think it was by @creator
```

## Supported Platforms

| Platform | URL Patterns | Method |
|----------|-------------|--------|
| YouTube | `youtube.com/watch`, `youtu.be/`, `youtube.com/shorts/` | yt-dlp (transcript + description) |
| Instagram Reels | `instagram.com/reel/` | yt-dlp + whisper transcription |
| Instagram Posts | `instagram.com/p/` | Playwright screenshot + vision |
| Instagram Stories | `instagram.com/stories/` | Playwright (ephemeral — extract fast) |
| TikTok | `tiktok.com/@*/video/`, `vm.tiktok.com/` | yt-dlp + whisper transcription |
| X / Twitter | `x.com/*/status/`, `twitter.com/*/status/` | WebFetch or Playwright |
| LinkedIn | `linkedin.com/posts/`, `linkedin.com/feed/` | Playwright (login may be required) |
| Reddit | `reddit.com/r/*/comments/`, `old.reddit.com/` | Append `.json` to URL or WebFetch |
| Podcasts | `open.spotify.com/episode/`, `podcasts.apple.com/` | Search transcript online, or download + whisper |
| Articles | Any other URL | reader / trafilatura / curl |
| PDFs | `*.pdf` | Direct download and parse |

## Knowledge Base Structure

All entries are saved to `~/content-harvest/`:

```
~/content-harvest/
  index.md                          # Searchable index of all entries
  entries/
    2026-03-28_instagram_claude-skills.md
    2026-03-29_youtube_mcp-tutorial.md
    2026-03-30_tiktok_ai-workflow.md
    ...
```

Each entry includes: date, source platform, creator, auto-generated tags, summary, extracted resources, key takeaways, and the full raw content.

## Auto-Categorization Tags

Content is automatically tagged with 1-3 of the following:

`ai` `business` `productivity` `marketing` `mindset` `tools` `coding` `personal-development` `design` `finance` `health` `leadership` `automation` `saas` `content-creation` `social-media` `sales`

## License

MIT
