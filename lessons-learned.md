# Lessons Learned

## Character Consistency

- Seed alone is NOT enough. The detailed character anchor block is what actually drives consistency. The seed locks the randomness so the model interprets the same description the same way.
- Put the character description FIRST in the prompt, before the scene. Models weight earlier tokens more heavily.
- The character anchor block must be word-for-word identical across all prompts for a character. Even small changes (adding/removing a word) can shift the face.
- Different seeds produce different people from the same character description. Each character needs their own dedicated seed.

## Voice Consistency

- Tested: same seed + same reference image + same scene description block = nearly identical voice across separate video generations.
- The seed controls voice characteristics alongside visual output.
- Changing the scene description or reference image may shift the voice.
- For multi-segment videos, keep everything constant except the dialogue text.

## Segment Duration

- 4 seconds is the sweet spot for talking-head content.
- 15s segments: too much room for voice drift, dead space, and speech misalignment.
- 6s segments: decent but more noticeable transitions.
- 4s segments: tightest speech-to-prompt alignment, least dead space, cleanest cuts.

## Segment Ordering Bug

- NEVER use Python's sorted() on filenames like seg1.mp4, seg10.mp4, seg11.mp4.
- Lexicographic sort: seg1, seg10, seg11, seg2, seg3... (wrong)
- Numeric iteration: seg1, seg2, seg3... seg10, seg11 (correct)
- Always use: `for i in range(1, N+1)` to build the concat list.
- Alternative: use zero-padded filenames (seg01, seg02... seg11).

## Resolution

- Grok Imagine Video defaults to 480x848 if you only specify aspect_ratio "9:16".
- Must explicitly include `"resolution": "720p"` in the payload to get 720x1280 output.
- Always verify output dimensions after first generation.

## Content Filters

- Grok Imagine has minimal content filtering -- handles shirtless/swimwear content without issues.
- Google Veo has aggressive content filters -- rejects shirtless reference images.
- If using Veo as an alternative, characters must wear clothing in reference images.

## Dead Space Removal

- AI-generated video segments typically have 0.3-1.5s of silence at boundaries.
- Over 11 segments, this accumulates to 5-8 seconds of dead space.
- Silence detection + extraction is essential as a final step.
- Use silencedetect with noise=-30dB and d=0.3 (minimum 300ms silence gap).

## Image Model Quality

- `grok-imagine-image-pro` produces noticeably better quality than `grok-imagine-image` standard.
- Always use pro for final/production images.
- Standard is fine for quick drafts and iteration.
- Both support the same parameters (seed, prompt).

## API Behavior

- Grok video jobs complete in ~15-30 seconds for 4s clips.
- All segments can be submitted in parallel -- no need to serialize.
- Polling interval of 10 seconds is sufficient.
- Submit via curl (not Python requests) for large base64 payloads to avoid SSL issues.
- Poll via Python requests (small responses work fine).

## Script Writing

- **8-10 words per 4-second segment** is optimal (tested: -15% density beat standard and +15%).
- +15% words (~12-14): lip sync degrades, speech feels rushed, model can't fit it all.
- Standard (~10-12): decent baseline but not optimal.
- -15% words (~8-10): best lip sync, most natural pacing, cleaner mouth movements. Produces more segments but superior quality.
- Accept more total segments over denser segments -- quality > compression.
- Hook must be in segment 1 -- viewers decide to stay or scroll in the first 2 seconds.
- End with a CTA (call to action) in the final segment.
- Short punchy phrases > long complex sentences.
