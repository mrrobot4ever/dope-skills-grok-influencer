# Setup Guide

## Prerequisites

### 1. xAI API Key (Required)

The skill uses Grok Imagine for both image and video generation. You need an xAI API key.

1. Go to https://console.x.ai/
2. Sign up or log in
3. Navigate to API Keys
4. Create a new API key
5. Save it to a file your agent can read:

```bash
echo "XAI_API_KEY=your-key-here" > ~/.openclaw/workspace/.env.xai
```

Or wherever your agent's workspace is:
```bash
echo "XAI_API_KEY=xai-xxxxxxxxxxxxxxxx" > /path/to/workspace/.env.xai
```

**Pricing (as of April 2026):**
- grok-imagine-image: ~$0.01 per image
- grok-imagine-image-pro: ~$0.02 per image
- grok-imagine-video (4s clip): ~$0.05 per clip
- A typical 10-segment video costs ~$0.50-0.70 total

### 2. ffmpeg (Required)

Used for concatenating video segments and encoding the final video.

**macOS:**
```bash
brew install ffmpeg
```

**Ubuntu/Debian:**
```bash
sudo apt install ffmpeg
```

**Verify:**
```bash
ffmpeg -version
```

### 3. Python 3.9+ with PIL/Pillow (Required)

Used for image compression before video generation.

```bash
pip install Pillow
```

### 4. Python requests library (Required)

Used for polling video generation status.

```bash
pip install requests
```

### 5. Telegram Bot (Optional)

For delivering images/videos to Telegram. If not using Telegram, the skill saves files locally.

Bot token location: `~/.openclaw/openclaw.json` at `config.channels.telegram.botToken`

## API Key Location

The skill reads the xAI API key from:
```python
XAI_API_KEY = open("~/.openclaw/workspace/.env.xai").read().split("=",1)[1].strip()
```

Adjust the path if your workspace is elsewhere. The key should be in the format:
```
XAI_API_KEY=xai-your-actual-key-here
```

## Quick Test

After setup, verify everything works:

```bash
# Test image generation
curl -s --max-time 60 \
  "https://api.x.ai/v1/images/generations" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_XAI_KEY" \
  -d '{"model": "grok-imagine-image", "prompt": "a red apple on a white table"}'
```

If you get a JSON response with `data[0].url`, you're good to go.

## Models Used

| Model | Purpose | Endpoint |
|-------|---------|----------|
| `grok-imagine-image-pro` | Character images (preferred) | POST /v1/images/generations |
| `grok-imagine-image` | Character images (fallback) | POST /v1/images/generations |
| `grok-imagine-video` | Video segments with speech | POST /v1/videos/generations |

All models are accessed through the same base URL: `https://api.x.ai/v1/`

## Rate Limits

As of April 2026, Grok Imagine has generous rate limits:
- Image generation: No apparent per-minute limit for normal usage
- Video generation: All segments can be submitted in parallel
- Video polling: 10-second intervals are sufficient

If you hit rate limits, add a small delay between submissions (1-2 seconds).
