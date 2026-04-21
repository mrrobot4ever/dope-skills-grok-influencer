---
name: dope-skills-grok-influencer
description: Generate AI influencer video content using Grok Imagine for image and video generation. Creates a consistent fictional character across multiple images and videos with voice consistency, writes viral scripts, generates multi-segment talking-head videos, and produces social-media-ready content. Use when asked to create influencer content, UGC-style talking head videos with a consistent AI character, social media video ads, or "Jack Bass"-style content. Triggers on "influencer video", "AI character video", "grok video", "talking head content", "social media video", "consistent character video".
---

# Grok Influencer Video Pipeline

Generate social-media-ready talking-head videos with a consistent AI character using Grok Imagine APIs.

## Prerequisites

See `references/setup.md` for full installation and API key setup instructions. Minimum requirements:
- xAI API key (sign up at https://console.x.ai/)
- ffmpeg installed
- Python 3.9+ with Pillow and requests

## Quick Start

If you just want to generate a video fast, here's the minimum flow:

1. Pick a character name and write a detailed physical description (the "anchor block")
2. Generate a 9:16 image with `grok-imagine-image-pro` using the anchor + scene + seed
3. Write a script, split into segments of ~15 words / 22-26 syllables each
4. Generate 4-second video clips for each segment (same image, same seed, fixed camera)
5. Concatenate clips in numeric order with ffmpeg
6. Done. Do NOT remove dead space.

Details for each step are below.

## Inputs Required

1. **Character description** -- detailed physical appearance (the "anchor block")
2. **Scene/setting** -- where the character is (office, poolside, jet, car, etc.)
3. **Script** -- what the character says to camera
4. **Seed number** -- integer for consistency (default: 42)

## Global Constants

| Constant | Value | Notes |
|----------|-------|-------|
| Image model | `grok-imagine-image-pro` | Higher quality than standard. Use `grok-imagine-image` as fallback only. |
| Video model | `grok-imagine-video` | Supports speech generation with lip sync |
| Segment duration | 4 seconds | Best speech-to-video alignment |
| Aspect ratio | `9:16` | Vertical/portrait for social media (Shorts, Reels, TikTok) |
| Resolution | `720p` | MUST specify explicitly or output defaults to 480p |
| Seed | User-specified (default 42) | Same seed = same character + same voice |
| Camera | Fixed/static | Add "static camera, no camera movement, locked tripod shot, fixed angle, no zoom, no pan" to every video prompt |
| API base | `https://api.x.ai/v1` | |
| API key location | `~/.openclaw/workspace/.env.xai` | Format: `XAI_API_KEY=xai-xxxxx` |

## Character Consistency

Character consistency across images comes from three things working together:

1. **Identical character anchor block** -- A detailed physical description that appears word-for-word at the START of every prompt. The more specific, the less variance between generations.

2. **Same seed** -- Controls the initial noise pattern. Same seed + same character block = same person. Different seeds = different people from identical prompts.

3. **Scene description at the end** -- Only the trailing part of the prompt changes. The model weights earlier tokens more heavily, so the character block dominates.

### Character Anchor Block Template

Include ALL of these attributes for maximum consistency:

```
[AGE]-year-old [gender] model named [NAME], [clothing], [build] build with [body fat]% body fat, [muscle details], [jawline], [cheekbones], [facial hair], [hair color/style/length], [eye color] eyes, [expression], [skin tone/texture]
```

**Example:**
```
22-year-old male model named Jack Bass, wearing a fitted white tank top, athletic build with low body fat 8 percent, visible muscle definition and vascularity on forearms and biceps, sharp jawline, high cheekbones, light stubble, medium-length textured dark brown hair swept back, hazel-green eyes, confident subtle smirk, tanned skin with natural skin texture and pores
```

**Important:** This exact block must appear word-for-word in every prompt for this character. Even small changes (adding or removing a word) can shift the face.

### Generating the Image

```python
import json, subprocess

payload = {
    "model": "grok-imagine-image-pro",
    "prompt": f"{CHARACTER_ANCHOR}, {SCENE_DESCRIPTION}, shot on Sony A7IV [lens]mm, 9:16 portrait orientation, no AI artifacts",
    "seed": 42
}

result = subprocess.run([
    'curl', '-s', '--max-time', '180',
    'https://api.x.ai/v1/images/generations',
    '-H', 'Content-Type: application/json',
    '-H', f'Authorization: Bearer {XAI_KEY}',
    '-d', json.dumps(payload)
], capture_output=True, text=True)

url = json.loads(result.stdout)['data'][0]['url']
# Download: curl -s -L -o output.jpg "$url"
```

**Always show the image to the user for approval before generating video.**

## Voice Consistency

Voice consistency across video segments comes from:
- **Same seed** across all segments
- **Same reference image** for all segments
- **Same scene description block** in every prompt
- **Only the dialogue text changes** between segments

Tested and confirmed: these four rules together produce nearly identical voices across separate generations.

## Fixed Camera (Required)

**Always include fixed camera instructions in every video prompt.** Without this, the camera angle drifts between segments, creating jarring transitions when concatenated.

Add this to the end of every scene description block:
```
static camera, no camera movement, locked tripod shot, fixed angle, no zoom, no pan
```

This produces consistent framing across all segments so cuts between them are seamless.

## Word Density & Syllable Count

**High density produces the best lip sync results.**

Target per 4-second segment:
- **~15 words**
- **22-26 syllables**
- Natural speaking pace is ~5-6 syllables/second

Word count alone is unreliable because word length varies enormously:
- "I lost my best friend" = 6 words, 6 syllables
- "I accidentally discovered institutions" = 4 words, 14 syllables

**After writing each segment, count the syllables.** This is the most reliable predictor of whether the speech will fit naturally in 4 seconds.

If a segment exceeds 26 syllables, split it or replace long words with shorter ones:
- "financial compensation" → "cash"
- "unfortunately" → "sadly"
- "accidentally discovered" → "stumbled on"

### Density Comparison (Tested)

| Density | Words/seg | Syllables/seg | Segments for 45s script | Result |
|---------|-----------|---------------|------------------------|--------|
| High (best) | ~15 | 22-26 | 9 | Best lip sync, fewest transitions |
| Normal | ~13 | 20-24 | 11 | Good but slightly looser sync |
| Low | ~10 | 16-20 | 14 | More dead space, more transitions |

## Pipeline Overview

```
1. Define character (anchor block + name + seed)
2. Generate 9:16 reference image (grok-imagine-image-pro)
3. Get user approval on image
4. Write script with viral hooks and open loops
5. Split into 4-second segments (~15 words, 22-26 syllables each)
6. Generate all video segments in parallel (grok-imagine-video + fixed camera)
7. Concatenate in numeric order (NOT lexicographic)
8. Send final video to user (do NOT remove dead space)
```

## Step-by-Step Process

### Step 1: Define Character

Create the character anchor block with every physical detail. Store it as a variable for reuse. Give the character a name.

### Step 2: Generate Reference Image

Generate a 9:16 portrait image using `grok-imagine-image-pro` with:
- Character anchor block at the START of the prompt
- Scene description after the anchor
- Camera/lens details
- `9:16 portrait orientation, no AI artifacts` at the end
- The chosen seed number

**Show the image to the user. Do not proceed until approved.**

### Step 3: Write Script

Write the full script. Guidelines for engaging content:

**Story-driven scripts (best for engagement):**
- Tell a real-feeling personal story
- Include specific details (dollar amounts, time frames, locations)
- Show vulnerability -- moments of failure or doubt
- Let the viewer feel like they're hearing something private

**Viral techniques:**
- **Hook in segment 1** -- curiosity gap, bold claim, or pattern interrupt ("Stop scrolling", "Nobody warned me about...")
- **Open loops** -- tease information that comes later ("I'll tell you what changed", "But that's not even the crazy part")
- **Controversy/enemy framing** -- "The gurus don't want you knowing this"
- **Specificity** -- "$412 profit", "fourteen months ago", "two in the morning"
- **Relatability** -- "I couldn't afford a sandwich", "sleeping on my boy's couch"
- **Scarcity in final segment** -- "before I take it down", "link in bio right now"

**Pure story scripts (no selling) perform best for building audience trust.** Save the CTA/pitch scripts for after you've built a following.

### Step 4: Split Script into Segments

Split the script into segments targeting:
- **~15 words per segment**
- **22-26 syllables per segment**

Count syllables for each segment. Adjust vocabulary if needed.

**Example of good segmentation:**
```
Seg 1: "I remember the exact morning everything changed for me. I was sitting right here with my last forty bucks." (18w, 28syl → split this)

Better:
Seg 1: "I remember the exact morning everything changed. I was sitting here with my last forty bucks." (15w, 24syl ✓)
```

### Step 5: Generate Video Segments

**Submit ALL segments in parallel** for speed:

```python
payload = {
    "model": "grok-imagine-video",
    "prompt": f"He talks to the camera saying: {segment_text} {SCENE_BLOCK}",
    "image": {"url": f"data:image/jpeg;base64,{img_b64}"},
    "duration": 4,
    "aspect_ratio": "9:16",
    "resolution": "720p",
    "seed": 42
}
```

**The scene block MUST include fixed camera instructions:**
```
Young [description] man [location], speaking directly to camera, natural lip movements, [energy/emotion], [lighting], static camera, no camera movement, locked tripod shot, fixed angle, no zoom, no pan, cinematic
```

**Reference image preparation:** Compress to 360x640 JPEG quality 50 before base64 encoding. This keeps the payload under API limits.

**Polling:** Check status every 10 seconds. 4-second clips typically complete in 15-30 seconds.

### Step 6: Concatenate in Correct Order

**CRITICAL: Never use sorted() on filenames.** `sorted(["seg1.mp4", "seg10.mp4", "seg2.mp4"])` produces wrong order.

Always use numeric iteration:

```python
with open("concat.txt", "w") as f:
    for i in range(1, num_segments + 1):
        f.write(f"file 'seg{i}.mp4'\n")
```

Re-encode for consistent output:

```bash
ffmpeg -y -f concat -safe 0 -i concat.txt \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 128k \
  -pix_fmt yuv420p -movflags +faststart \
  final.mp4
```

### Step 7: Do NOT Remove Dead Space

**Do not run silence removal on AI-generated talking head videos.**

The "silent" moments contain critical visual information:
- Mouth closing between sentences
- Breathing movements
- Facial expression transitions

Removing these frames breaks lip sync. The audio sounds smooth but the mouth jumps between positions with no transition.

The concatenated video from Step 6 IS the final deliverable.

### Step 8: Deliver

Send the final video to the user immediately. Never make them ask for it.

## Creating Multiple Characters

Each character needs:
1. A unique anchor block with distinct physical features
2. A unique seed number (character A = seed 42, character B = seed 100, etc.)

Characters with different seeds will look completely different even with similar descriptions. This is by design.

## Error Handling

| Issue | Cause | Fix |
|-------|-------|-----|
| Segments out of order | Lexicographic filename sort | Use numeric `for i in range(1, N+1)` |
| Voice drift between segments | Different seeds or ref images | Same seed + same image + same scene block for all segments |
| Low resolution (480p) | Missing resolution param | Always include `"resolution": "720p"` |
| Character looks different | Anchor block changed | Keep anchor block identical, word-for-word |
| Camera angle jumps between segments | No fixed camera instruction | Add static camera instructions to every prompt |
| Lip sync broken after editing | Dead space was removed | Do NOT remove dead space from AI video |
| Content filtered by other APIs | Shirtless/swimwear in reference image | Grok handles this fine; Google Veo does not |
| Image generation timeout | API slow on pro model | Retry once; pro model occasionally takes longer |

## File Organization

```
projects/<character-name>/
  <character>-<scene>.jpg           -- approved 9:16 reference image
  <scene>-seg1.mp4 ... segN.mp4    -- individual video segments
  <character>-<scene>-final.mp4     -- concatenated final video
```

## See Also

- `references/setup.md` -- API key setup, dependencies, quick test
- `references/command-reference.md` -- Copy-paste code for every step
- `references/lessons-learned.md` -- All pitfalls and findings from testing
- `examples/` -- Demo videos generated with this skill
