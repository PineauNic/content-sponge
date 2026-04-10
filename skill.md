---
name: content-sponge
description: Process social media content (Instagram, YouTube, TikTok, X/Twitter, LinkedIn, Reddit, Threads, Bluesky, Facebook, Pinterest, podcasts, articles) sent directly or via messaging. Extracts content, finds resources (GitHub repos, skills, MCPs, tools), and acts on instructions (install, save, analyze, search). Use when user sends a URL or content link with context like "check this out", "install this", "implement this", "save this", or any content link with notes.
allowed-tools: Bash,Read,Write,Glob,Grep,WebFetch,WebSearch
---

# Content Sponge Pipeline

Process social media content and turn it into action. When the user sends a link (Instagram reel, YouTube video, TikTok, tweet, article, podcast, etc.) with optional notes, extract the content, find mentioned resources, and execute or queue actions.

## When to Use This Skill

Activate when the user:
- Sends a social media URL (Instagram, YouTube, TikTok, X/Twitter, LinkedIn, Reddit)
- Sends a podcast URL (Spotify, Apple Podcasts)
- Sends an article or webpage URL with instructions beyond just "read"
- Says "check this out [URL]"
- Says "install this [URL]" or "install the skill/MCP from this"
- Says "implement this [URL]"
- Says "save this [URL]" or "bookmark this [URL]"
- Sends multiple URLs for batch processing
- Asks "what was that reel about X?" or "find my saved content about Y"
- Asks "summarize everything I've saved about [topic]"

**Do NOT activate** when the user just wants a plain transcript (use youtube-transcript) or just wants article text (use article-extractor). This skill is for when there's an ACTION to take beyond extraction.

## Core Workflow

### Step 1: Detect Content Type

```
instagram.com/reel/           → Instagram Reel (video)
instagram.com/p/              → Instagram Post/Carousel (image)
instagram.com/stories/        → Instagram Story (ephemeral - extract fast)
youtube.com/watch             → YouTube Video
youtu.be/                     → YouTube Video (short URL)
youtube.com/shorts/           → YouTube Short
tiktok.com/@*/video/          → TikTok Video
vm.tiktok.com/               → TikTok Video (short URL)
x.com/*/status/              → X/Twitter Post
twitter.com/*/status/         → X/Twitter Post
linkedin.com/posts/           → LinkedIn Post
linkedin.com/feed/            → LinkedIn Feed Post
reddit.com/r/*/comments/      → Reddit Thread
old.reddit.com/r/*/comments/  → Reddit Thread
threads.net/@*/post/          → Threads Post
threads.net/t/                → Threads Post (short URL)
bsky.app/profile/*/post/      → Bluesky Post
mastodon.social/@*/           → Mastodon Post (or any Mastodon instance)
facebook.com/*/posts/         → Facebook Post
facebook.com/watch/           → Facebook Video
facebook.com/reel/            → Facebook Reel
pinterest.com/pin/            → Pinterest Pin
snapchat.com/spotlight/       → Snapchat Spotlight
open.spotify.com/episode/     → Podcast Episode
podcasts.apple.com/           → Podcast Episode
*.pdf                         → PDF Document
Everything else               → Article/webpage
```

### Step 2: Extract Content

**GOLDEN RULE: For ANY video content (reels, TikToks, YouTube, Facebook videos, Snapchat, etc.), ALWAYS download the video and transcribe the actual audio with Whisper. Never rely on on-screen captions, auto-generated subtitles, or screenshot text alone — they are incomplete. The audio transcript is the source of truth. Screenshots and captions are supplementary only.**

#### YouTube Videos
Use the youtube-transcript skill approach:
1. `yt-dlp --write-auto-sub --skip-download --sub-langs en --output "temp_ci" "URL"`
2. Convert VTT to clean text (deduplicate)
3. Also grab video description: `yt-dlp --print "%(description)s" "URL"`
4. The description often contains the actual resource links

```bash
# Get transcript + description in parallel
VIDEO_TITLE=$(yt-dlp --print "%(title)s" "$URL" | tr '/' '_' | tr ':' '-' | tr '?' '' | tr '"' '')
VIDEO_DESC=$(yt-dlp --print "%(description)s" "$URL")
CHANNEL=$(yt-dlp --print "%(channel)s" "$URL")

# Download subtitles
yt-dlp --write-auto-sub --skip-download --sub-langs en --output "temp_ci" "$URL" 2>/dev/null

# Convert to text
VTT_FILE=$(ls temp_ci*.vtt 2>/dev/null | head -n 1)
if [ -f "$VTT_FILE" ]; then
    python3 -c "
import re
seen = set()
with open('$VTT_FILE', 'r') as f:
    for line in f:
        line = line.strip()
        if line and not line.startswith('WEBVTT') and not line.startswith('Kind:') and not line.startswith('Language:') and '-->' not in line:
            clean = re.sub('<[^>]*>', '', line)
            clean = clean.replace('&amp;', '&').replace('&gt;', '>').replace('&lt;', '<')
            if clean and clean not in seen:
                print(clean)
                seen.add(clean)
" > temp_transcript.txt
    rm "$VTT_FILE"
fi
```

#### Instagram Reels
**IMPORTANT: Always download and transcribe the actual audio — never rely on on-screen captions alone. Captions may be incomplete, auto-generated, or missing entirely.**

Extraction priority (try each in order until one works):

**Method 1: yt-dlp with browser cookies (best quality)**
```bash
# Download reel video with audio
yt-dlp --cookies-from-browser chrome -o "temp_reel.%(ext)s" "$URL"

# Extract audio
ffmpeg -i temp_reel.mp4 -vn -ar 16000 -ac 1 -c:a pcm_s16le temp_reel_audio.wav 2>/dev/null

# Transcribe with Whisper (use whisper-mcp tool if available, or local whisper)
# This gives you the ACTUAL spoken words, not just what's on screen
```

**Method 2: yt-dlp without cookies (sometimes works for public reels)**
```bash
yt-dlp -o "temp_reel.%(ext)s" "$URL"
# If this works, extract audio and transcribe same as above
```

**Method 3: Playwright video capture (fallback)**
```bash
# Navigate to the reel URL with Playwright
# Use browser_evaluate to find the video element source URL
# Download the video directly from the source URL
# Extract audio and transcribe with Whisper
```

**Method 4: Screenshot + caption extraction (LAST RESORT ONLY)**
Only use this if all audio extraction methods fail:
1. Navigate with Playwright, take screenshot
2. Extract caption text from the page
3. **Clearly note in the output that this is caption text only, NOT a full audio transcript**
4. Flag: "Audio transcription unavailable — install browser cookies for yt-dlp: `yt-dlp --cookies-from-browser chrome`"

**Also always extract the post caption/description** — this often contains links, hashtags, and context not mentioned in the video:
```bash
# Use Playwright to navigate to the post and extract caption text from the page DOM
# OR: yt-dlp --print "%(description)s" "$URL" (if yt-dlp can access it)
```

#### Instagram Posts/Carousels
Use Playwright to screenshot and extract:

```bash
# Navigate to post URL with Playwright
# Take screenshot
# Use Claude vision to read carousel slides
# Extract caption text from page
```

#### TikTok Videos
**IMPORTANT: Always download and transcribe the actual audio — never rely on on-screen captions.**

```bash
# Download TikTok video (yt-dlp supports TikTok natively)
yt-dlp -o "temp_tiktok.%(ext)s" "$URL"

# Extract audio and transcribe with Whisper
ffmpeg -i temp_tiktok.mp4 -vn -ar 16000 -ac 1 -c:a pcm_s16le temp_tiktok_audio.wav 2>/dev/null
# Transcribe with whisper-mcp or local whisper

# Get video description/caption (supplementary — not a replacement for audio transcript)
TIKTOK_DESC=$(yt-dlp --print "%(description)s" "$URL" 2>/dev/null)
TIKTOK_CREATOR=$(yt-dlp --print "%(uploader)s" "$URL" 2>/dev/null)
```

**If yt-dlp fails for TikTok:**
1. Try with `--cookies-from-browser chrome` flag
2. Try Playwright to find the video source URL, download directly, then transcribe audio
3. Screenshot + caption is LAST RESORT only — flag that audio transcript is missing

#### X/Twitter Posts
Extract tweet text and any embedded media:

```bash
# Use WebFetch to get tweet content
# WebFetch the URL — many tweet embed pages include the text

# For threads: follow the conversation chain
# Check for quoted tweets and referenced posts

# For media (images/video):
# Images: Playwright screenshot or WebFetch
# Video: yt-dlp "$URL" for embedded video
```

**Thread handling:**
1. Fetch the initial tweet via WebFetch
2. Look for "Show this thread" / thread indicators
3. Follow reply chain from the same author
4. Compile full thread text in order

**If WebFetch fails** (auth walls, etc.):
1. Try Playwright to navigate and screenshot
2. Use Claude vision to read the screenshot
3. For video content, try `yt-dlp "$URL"`

#### LinkedIn Posts
LinkedIn blocks most scraping, so use Playwright:

```bash
# Use Playwright to navigate to the LinkedIn post URL
# browser_navigate to the URL
# browser_snapshot to extract text content
# browser_take_screenshot for visual content

# LinkedIn may require login — if so, flag for manual handling
```

**Extraction steps:**
1. Navigate to the post URL with Playwright
2. Take a snapshot to extract text content
3. If the post contains images/carousels, take screenshots
4. Extract author name, post text, and any links shared
5. If login wall appears, flag it and save the URL for manual review

#### Reddit Threads
Reddit provides easy API-style access:

```bash
# Method 1: Append .json to the Reddit URL for structured data
# WebFetch "${URL}.json" and parse the JSON response

# Method 2: Use old.reddit.com for simpler HTML
# Replace reddit.com with old.reddit.com in the URL
# WebFetch the old.reddit.com version for cleaner extraction

# Method 3: Fallback to Playwright
```

**Extraction steps:**
1. Try appending `.json` to the URL and fetch with WebFetch
2. Parse the JSON for: title, selftext, top comments, linked URLs
3. If JSON fails, try the `old.reddit.com` version
4. Extract OP text + top-level comments (sort by best)
5. Pull out any URLs shared in comments

#### Podcast Episodes
Podcasts require finding transcripts or downloading audio:

```bash
# Step 1: Search for existing transcript
# WebSearch "[podcast name] [episode title] transcript"

# Step 2: If transcript found, fetch it with WebFetch

# Step 3: If no transcript, try downloading audio
# For Spotify: yt-dlp may work with some episodes
# For Apple Podcasts: extract the audio URL from the page, then download

# Step 4: Transcribe downloaded audio with whisper
# whisper podcast_audio.mp3 --model base --output_format txt
# For long episodes, consider --model tiny for speed
```

**Spotify episodes:**
1. Search for transcript via WebSearch first (many podcasts publish transcripts)
2. Try `yt-dlp` — it works for some Spotify content
3. If audio is long (>30 min), extract and transcribe the first 10 minutes, then ask if the user wants the full transcription

**Apple Podcasts:**
1. Fetch the page with WebFetch to find the audio stream URL
2. Download the audio file
3. Transcribe with whisper

#### Articles/Webpages
Use article-extractor skill approach:
1. Try `reader` (Mozilla Readability) first
2. Fallback to `trafilatura`
3. Last resort: `curl` + basic parsing

### Step 3: Resource Extraction (Critical Step)

After extracting content, analyze it to find actionable resources. This is the intelligence layer.

**What to look for in transcript/text:**
- GitHub repository URLs or mentions ("github.com/...", "available on GitHub", "link in bio to the repo")
- Skill/MCP names and install commands ("install the X skill", "add this MCP server")
- NPM packages, pip packages, brew packages
- Website URLs for tools, downloads, or resources
- Specific file names (.md, .json, .yaml configs)
- CLI commands shown or dictated

**What to look for in video descriptions (YouTube):**
- Direct URLs to resources
- Timestamps with resource mentions
- Affiliate/referral links (usually contain the actual tool URL)
- "Links mentioned in this video" sections

**What to look for in Instagram/TikTok captions:**
- "Link in bio" → navigate to profile, find linktree/link
- "DM me [keyword]" → flag for manual action, then check email later
- "Comment [keyword]" → flag for manual action, then check email later
- Direct URLs in caption

**What to look for in tweets/LinkedIn posts:**
- URLs in the post text
- Mentions of tools, repos, or resources by name
- Thread continuations with additional links
- Comments/replies containing resource links

**What to look for in Reddit threads:**
- URLs in the original post and comments
- GitHub repo links (very common in tech subreddits)
- Tool recommendations with links
- Pinned/top comments often have curated resource lists

**Resource extraction prompt pattern:**
```
Analyze this content and extract ALL mentioned resources:

Content: [transcript/text]
Description: [video description or caption]
User notes: [what the user said when sending this]

Extract:
1. Direct URLs found in text
2. GitHub repos (by name or URL)
3. Tools/software mentioned
4. Skills, MCPs, or Claude-related resources
5. Installation commands mentioned
6. Any gated resources ("DM for link", "comment X", "link in bio")

For each resource:
- Name/identifier
- URL (if found) or search terms to find it
- How to access (direct, link-in-bio, DM-gated, email-gated)
- What it does (based on context)
```

### Step 4: Cross-Platform Resource Retrieval

#### Layer 1: Direct Links
- URLs found in descriptions/captions → validate with WebFetch or Playwright
- GitHub repos → verify they exist: `gh repo view owner/repo`
- Direct download links → download with curl/wget

#### Layer 2: Link-in-Bio / Website
- Navigate to creator's profile using Playwright
- Find linktree, beacons, or direct link
- Match resource to what was discussed in content
- Download if freely available

```
# Playwright workflow:
1. browser_navigate to profile URL
2. browser_snapshot to find link-in-bio
3. browser_click on the link
4. browser_snapshot the landing page
5. Look for the specific resource mentioned
6. Download if available
```

#### Layer 3: DM/Email-Gated Resources
When content says "DM me X" or "comment X to get the link":

1. **Flag it**: Save to pending-resources with instructions
2. **After user engages**: Search email for the resource
   - Use Gmail MCP (if available): search with creator name or resource keyword
   - Look for recent emails (within last hour/day)
   - Extract download link from email body
3. **Common patterns**:
   - ManyChat automations send from the creator's email or a noreply
   - Subject often contains the keyword or resource name
   - Body contains a direct download link (Google Drive, Notion, Gumroad, etc.)

```
Email search strategies:
- Search by creator name: "from:creator@email.com"
- Search by keyword: "subject:[resource name]"
- Search by recency: "newer_than:1d [keyword]"
- Search by common senders: "from:manychat OR from:noreply subject:[keyword]"
```

#### Layer 4: Cannot Retrieve
- Save full context to pending-resources/
- Include: what resource, where it might be, how to get it
- The user can manually engage and then ask to find it in email

### Step 5: Action Classification & Execution

Based on the user's notes and content analysis:

#### Auto-Execute (when user explicitly says to):
```
"Install this skill" →
  1. Clone/download the skill files
  2. Copy to ~/.claude/skills/[skill-name]/
  3. Verify skill.md exists and is valid
  4. Report: installed, how to invoke

"Install this MCP" →
  1. Find the MCP server package/repo
  2. Add to Claude Code MCP configuration
  3. Verify it loads
  4. Report: installed, available tools

"Clone this repo" →
  1. gh repo clone [repo] ~/Projects/[repo-name]
  2. Read README for setup instructions
  3. Report: cloned, key files, setup steps needed
```

#### Confirm First (when user doesn't explicitly say):
```
"Check this out" / no explicit instruction →
  1. Extract and analyze content
  2. Summarize what it's about
  3. List found resources
  4. Ask: "Want me to [install/implement/save] any of this?"

"Implement this" →
  1. Extract and analyze
  2. Create implementation plan
  3. Present plan for approval
  4. Execute after confirmation
```

#### Save for Later Mode
When the user says "save this" or "bookmark this" without further instruction:
1. Extract content (transcript, text, metadata)
2. Auto-categorize with tags (see Step 5b)
3. Save to knowledge base (Step 6)
4. Quick acknowledgment: "Saved: [title] — tagged [tags]"

#### Smart Recommendations
When processing content, analyze it against what you know about the user (from CLAUDE.md, memory, past conversations, their workflow/stack) and proactively recommend:

```
After extracting and analyzing content, add a "Recommendation" section:

RECOMMENDATION:
Based on your [role/workflow/stack/goals]:
- Relevance: [HIGH/MEDIUM/LOW] — [why this matters to them specifically]
- Should implement: [YES/NO/MAYBE] — [specific reasoning]
- Priority: [NOW/SOON/BACKLOG] — [based on their current priorities]
- How it fits: [where this plugs into their existing workflow]
- Conflicts: [anything in their current setup this would conflict with]
- Effort: [QUICK (< 30 min) / MEDIUM (1-3 hrs) / LARGE (day+)]

If YES or MAYBE:
- Here's how I'd implement it: [2-3 sentence implementation plan]
- Want me to proceed?
```

The recommendation should reference specific details about the user's setup — not generic advice. If you know they use Notion, reference how this integrates with Notion. If you know they run automations on Railway, reference that. The more personalized, the more useful.

When the user asks "what should I implement?" or "what's worth doing?" about previously saved content:
1. Search the knowledge base for unactioned entries
2. Re-evaluate each against current context and priorities
3. Rank by: relevance to current goals > effort required > recency
4. Present top 3-5 recommendations with reasoning
5. No further action unless requested

### Step 5b: Auto-Categorization

After extracting content, automatically assign tags from this list based on content analysis:

```
ai, business, productivity, marketing, mindset, tools, coding,
personal-development, design, finance, health, leadership,
automation, saas, content-creation, social-media, sales
```

**Rules:**
- Assign 1-3 tags per entry (most relevant only)
- Base tags on the actual content, not the platform
- If content clearly doesn't fit any tag, use the closest match
- Tags are lowercase, hyphenated for multi-word

### Step 6: Save to Knowledge Base

Every processed piece of content gets saved to `~/content-sponge/entries/`.

Create the directory if it doesn't exist: `mkdir -p ~/content-sponge/entries`

**Entry format:**
```markdown
---
date: YYYY-MM-DD
source: instagram_reel | youtube_video | youtube_short | tiktok | x_post | linkedin_post | reddit_thread | podcast | article
url: [original URL]
creator: [creator name/handle]
topic: [main topic]
tags: [auto-categorized tags from Step 5b]
status: processed | pending_resources | actioned | saved
---

## Summary
[2-3 sentence summary of what the content covers]

## User Notes
[What the user said when sending this]

## Resources Found
- [Resource 1]: [URL] — [status: retrieved/pending/installed]
- [Resource 2]: [URL] — [status]

## Actions Taken
- [What was done, e.g., "Installed nano-banana skill to ~/.claude/skills/"]

## Key Takeaways
[Bullet points of the most important/actionable information]

## Raw Content
<details>
<summary>Full transcript/text</summary>

[Full extracted content]
</details>
```

**Naming convention:** `YYYY-MM-DD_[platform]_[short-description].md`
Example: `2026-03-28_instagram_claude-skills-nano-banana.md`

### Step 7: Update Index

After saving an entry, update `~/content-sponge/index.md`:

```markdown
# Content Sponge Index

## Recent Entries
| Date | Platform | Topic | Creator | Tags | Status | File |
|------|----------|-------|---------|------|--------|------|
| 2026-03-28 | Instagram | Claude skills | @creator | ai, tools | actioned | [link] |
```

Create the index file if it doesn't exist.

## Batch Processing

When the user sends multiple URLs (separated by newlines, spaces, or in a list):

1. Detect all URLs in the input
2. Process each URL sequentially through Steps 1-6
3. Update the index once at the end (not after each entry)
4. Return a combined summary:

```
BATCH PROCESSED: [N] items

1. [Platform] — [Title/Topic] by [Creator]
   Tags: [tags]
   Resources: [count found]
   Status: [processed/saved/actioned]

2. [Platform] — [Title/Topic] by [Creator]
   ...

All saved to knowledge base.
```

## Search Saved Content

When the user asks "what was that reel about X?" or "find my saved content about Y":

1. Search `~/content-sponge/entries/` for matching files
2. Use grep for keyword matching across file contents
3. Also check filenames for platform/topic matches
4. Also search by tags in the YAML frontmatter
5. Return summaries of matching entries:

```
FOUND [N] matches for "[query]":

1. [Date] — [Platform] — [Title]
   Tags: [tags]
   Summary: [2-3 sentences]
   File: ~/content-sponge/entries/[filename]

2. ...
```

If no matches found, suggest broadening the search terms.

## Topic Aggregation

When the user asks "summarize everything I've saved about [topic]":

1. Search entries by tag and keyword matching
2. Compile all matching entries
3. Generate a cross-entry summary:

```
TOPIC SUMMARY: [topic]
Entries: [N] items from [date range]

## Key Themes
- [Theme 1 with supporting entries]
- [Theme 2 with supporting entries]

## Top Resources
- [Most mentioned/valuable resources across entries]

## Patterns
- [Recurring ideas, tools, or advice across multiple entries]

## Recommended Actions
- [Actionable next steps based on aggregated content]
```

## Response Format

When processing content, respond with:

```
PROCESSED: [Content title/topic]
Source: [platform] by [creator]
Tags: [auto-categorized tags]

FOUND:
- [Resource 1]: [status]
- [Resource 2]: [status]

ACTION:
- [What was done or what's recommended]

Saved to knowledge base.
```

Keep it concise — the user may be on mobile when sending these.

## Handling "I saw a reel about X" (No URL)

If the user describes content but doesn't have the URL:
1. Search for it using WebSearch with key terms + platform
2. Try to find the specific creator or content
3. If found, process normally
4. If not found, save the intent/topic to knowledge base anyway for future reference

## Error Handling

**Instagram download fails:**
- Check if cookies are configured: `ls ~/.config/yt-dlp/` or check browser cookie availability
- If no cookies: use Playwright as fallback for screenshots
- If Playwright fails: ask the user to screenshot and send

**TikTok download fails:**
- Try with `--cookies-from-browser chrome` flag
- Try with `--user-agent` flag (TikTok can block default yt-dlp user agent)
- Fallback: Playwright screenshot + vision analysis

**X/Twitter extraction fails:**
- Twitter/X frequently changes their anti-scraping measures
- Try Playwright as primary fallback
- Nitter instances (if available) can be an alternative: replace x.com with a nitter instance

**LinkedIn blocks access:**
- LinkedIn aggressively blocks scraping — Playwright with logged-in session is often required
- If login wall appears, save the URL and flag for manual review
- Try WebFetch first in case the post is public

**Reddit .json fails:**
- Try old.reddit.com version
- Fallback to WebFetch on the standard URL
- Last resort: Playwright

**YouTube transcript unavailable:**
- Try downloading audio and transcribing with whisper
- If audio too long: extract first 5 minutes for quick analysis

**Podcast audio unavailable:**
- Search for transcript online first (many podcasts publish them)
- Try alternative podcast directories for the same episode
- If audio is very long, offer to transcribe a segment

**Resource URL is dead/moved:**
- Try Wayback Machine: `web.archive.org/web/[URL]`
- Search for alternative source
- Save as pending with note

**Email search returns nothing (if Gmail MCP available):**
- Might not have arrived yet — save to pending-resources with retry instructions
- Suggest checking spam folder
- Try broader search terms

## Dependencies

**Required:**
- yt-dlp (YouTube, Instagram, TikTok, X video download)
- Python 3 (text processing)
- ffmpeg (audio extraction from video)
- WebFetch/WebSearch (resource lookup, article extraction)

**Recommended:**
- whisper (OpenAI Whisper for local audio transcription)
- Playwright MCP (browser automation for LinkedIn, fallback scraping)
- gh CLI (GitHub repo operations)

**Optional:**
- Gmail MCP (email search for gated resources)
- reader / trafilatura (article extraction — fallback to curl exists)
- Whisper MCP (if available as MCP server instead of local install)
