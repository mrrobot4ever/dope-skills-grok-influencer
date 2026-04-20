---
name: dope-skills-grok-influencer
description: Generate AI influencer video content using Grok Imagine for image and video generation. Creates a consistent fictional character across multiple images and videos with voice consistency, writes viral scripts, generates multi-segment talking-head videos, and produces social-media-ready content. Use when asked to create influencer content, UGC-style talking head videos with a consistent AI character, social media video ads, or "Jack Bass"-style content. Triggers on "influencer video", "AI character video", "grok video", "talking head content", "social media video", "consistent character video".
---

# Grok Influencer Video Pipeline

Generate social-media-ready talking-head videos with a consistent AI character using Grok Imagine APIs.

## Inputs Required

1. **Character description** -- detailed physical appearance (the "anchor block")
2. **Scene/setting** -- where the character is (office, poolside, jet, car, etc.)
3. **Script** -- what the character says to camera
4. **Seed number** -- integer for consistency (default: 42)

## Global Constants

| Constant | Value |
|----------|-------|
| Image model | `grok-imagine-image` |
| Video model | `grok-imagine-video` |
| Video segment duration | 4 seconds (optimal for speech pacing) |
| Aspect ratio | 9:16 (vertical/portrait for social) |
| Resolution | 720p (must specify explicitly) |
| Seed | User-specified (default 42) |
| API base | `https://api.x.ai/v1` |
| API key env | `~/.openclaw/workspace/.env.xai` (XAI_API_KEY) |

## Character Consistency

Character consistency across images comes from three things working together:

1. **Identical character anchor block** -- A detailed physical description that appears word-for-word at the START of every prompt. Include: age, build, body fat %, specific muscle details, face structure (jawline, cheekbones), facial hair, hair color/style/length, eye color, expression, skin tone/texture. The more specific, the less variance.

2. **Same seed** -- The seed controls the initial noise pattern. Same seed + same character block = same person interpreted by the model. Different seeds produce different people from identical prompts.

3. **Scene description at the end** -- Only the trailing portion of the prompt changes between images. The model weights earlier tokens more heavily, so the character block dominates.

### Character Anchor Block Template

```
[AGE]-year-old [gender] model named [NAME], [clothing], [build] build with [body fat]% body fat, [muscle details], [jawline], [cheekbones], [facial hair], [hair color/style/length], [eye color] eyes, [expression], [skin tone/texture]
```

Example:
```
22-year-old male model named Jack Bass, wearing a fitted white tank top, athletic build with low body fat 8 percent, visible muscle definition and vascularity on forearms and biceps, sharp jawline, high cheekbones, light stubble, medium-length textured dark brown hair swept back, hazel-green eyes, confident subtle smirk, tanned skin with natural skin texture and pores
```

### Image Generation

```bash
curl -s --max-time 120 \
  "https://api.x.ai/v1/images/generations" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $XAI_KEY" \
  -d '{
    "model": "grok-imagine-image",
    "prompt": "[CHARACTER ANCHOR BLOCK], [SCENE DESCRIPTION], shot on Sony A7IV [lens], 9:16 portrait orientation, no AI artifacts",
    "seed": 42
  }'
```

Response: `data[0].url` -- download with curl.

## Voice Consistency

Voice consistency across video segments comes from using the **same seed + same reference image + similar prompt structure**.

Tested and confirmed: Grok Imagine Video with identical seed, identical reference image, and the same character/scene description block produces nearly identical voices across separate generations. The seed anchors not just the visual output but the voice characteristics.

**Rules for voice consistency:**
- Always use the same seed across all segments of a video
- Always use the same reference image for all segments
- Keep the scene description portion of the prompt identical across segments
- Only change the dialogue text between segments

## Video Segment Duration

**4 seconds is optimal.** Tested 6s, 8s, and 15s segments:
- 15s: Voice can drift, more dead space, speech may not match prompt length
- 6s: Good but slightly more transitions/cuts visible
- 4s: Best speech-to-segment alignment, tightest pacing, least dead space

## Pipeline Overview

```
Define character → Generate reference image (Grok Image + seed)
  → Write script → Split into 4s segments
  → Generate all video segments (Grok Video + seed + ref image)
  → Concat in numeric order
  → Remove dead space (silence removal)
  → Final video
```

## Step-by-Step Process

### Step 1: Define Character

Create the character anchor block. Store it for reuse across all content for this character. Give the character a name for easy reference.

### Step 2: Generate Reference Image

Generate a 9:16 portrait image using Grok Imagine with the character anchor block + scene description + seed.

**Always send the image to the user for approval before proceeding.**

### Step 3: Write Script

Write the full script the character will say to camera. Target length based on desired video duration (roughly 10-12 words per 4-second segment).

**Script writing guidelines for viral content:**
- **Hook in first segment** -- curiosity gap, flex, or bold claim
- **Open loops** -- promise information that comes later ("I'll tell you what changed")
- **Pattern interrupts** -- contrast common behavior with the character's approach
- **Specificity** -- concrete numbers, named strategies, real details
- **Relatability** -- rags-to-riches arc, "I was where you are"
- **Scarcity/urgency in final segment** -- "before I take it down", "link in bio"

### Step 4: Split Script into 4-Second Segments

Each segment should be one natural phrase or sentence. Aim for 10-12 words per segment. The dialogue must fit comfortably in 4 seconds of speech.

### Step 5: Generate Video Segments

Submit ALL segments to Grok Imagine Video in parallel:

```python
payload = {
    "model": "grok-imagine-video",
    "prompt": f"He talks to the camera saying: {segment_text} {scene_description}",
    "image": {"url": f"data:image/jpeg;base64,{img_b64}"},
    "duration": 4,
    "aspect_ratio": "9:16",
    "resolution": "720p",
    "seed": 42
}
```

**Reference image preparation:** Compress to 360x640 JPEG quality 50 before base64 encoding.

Poll all jobs until done. Save with numeric naming: `seg1.mp4`, `seg2.mp4`, etc.

### Step 6: Concatenate in Correct Order

**CRITICAL: Never use sorted() on filenames.** Lexicographic sort puts seg10 before seg2. Always iterate numerically:

```python
with open(concat_file, "w") as f:
    for i in range(1, num_segments + 1):
        f.write(f"file 'seg{i}.mp4'\n")
```

Concat with re-encode for consistent encoding:

```bash
ffmpeg -y -f concat -safe 0 -i concat.txt \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 128k \
  -pix_fmt yuv420p -movflags +faststart \
  final.mp4
```

### Step 7: Remove Dead Space

Detect silence gaps and extract only speaking portions:

```python
# Detect silence
ffmpeg -i input.mp4 -af "silencedetect=noise=-30dB:d=0.3" -f null -

# For each gap between silences, extract the speaking segment
# Concat speaking segments only
```

This typically removes 5-8 seconds from a 44-second video.

**Send the final video to the user immediately upon completion.**

## Prompt Structure

Every video segment prompt follows this structure:

```
He talks to the camera saying: [DIALOGUE]. [SCENE DESCRIPTION WITH CHARACTER]
```

The scene description block should be identical across all segments:

```
Young [clothing] [build description] man [location details], speaking directly to camera, natural lip movements, confident energy, [lighting], cinematic
```

## Error Handling

| Issue | Cause | Fix |
|-------|-------|-----|
| Segments out of order | Lexicographic filename sort | Use numeric iteration |
| Voice drift between segments | Different seeds or prompts | Same seed + same ref image + same scene block |
| Low resolution (480p) | Missing resolution param | Always include `"resolution": "720p"` |
| Character looks different | Changed anchor block or seed | Keep anchor block word-for-word identical |
| Dead space in video | Natural gaps between segments | Run silence removal as final step |
| Content filtered | Prompt or image triggers safety | Rephrase prompt, add clothing to reference image |

## File Organization

```
projects/<character-name>/
  <character>-portrait.jpg        -- approved 9:16 reference image
  seg1.mp4 ... segN.mp4          -- individual video segments
  <character>-final.mp4           -- concatenated video
  <character>-tight.mp4           -- dead space removed (final deliverable)
```

## Creating a New Character

1. Write a new character anchor block with unique physical details
2. Choose a new seed number (different from other characters)
3. Generate a hero image and get user approval
4. All subsequent images and videos for this character use the same anchor + seed

Multiple characters can coexist -- each has their own anchor block + seed pair.

## Delivery

Send all generated images and videos to the user via Telegram immediately upon generation. Never make the user ask for a file they're waiting on.

For Telegram delivery, read the bot token from `~/.openclaw/openclaw.json` at `config['channels']['telegram']['botToken']`.
