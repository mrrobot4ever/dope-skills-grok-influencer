# Command Reference

## Variables

```python
import os
XAI_KEY = open(os.path.expanduser("~/.openclaw/workspace/.env.xai")).read().split("=",1)[1].strip()
```

## Generate Character Image

```python
import json, subprocess

result = subprocess.run([
    'curl', '-s', '--max-time', '120',
    'https://api.x.ai/v1/images/generations',
    '-H', 'Content-Type: application/json',
    '-H', f'Authorization: Bearer {XAI_KEY}',
    '-d', json.dumps({
        "model": "grok-imagine-image",
        "prompt": f"{CHARACTER_ANCHOR}, {SCENE_DESCRIPTION}, shot on Sony A7IV 85mm, 9:16 portrait orientation, no AI artifacts",
        "seed": SEED
    })
], capture_output=True, text=True)

url = json.loads(result.stdout)['data'][0]['url']
subprocess.run(['curl', '-s', '-L', '-o', output_path, url])
```

## Compress Reference Image for Video

```python
from PIL import Image
import base64, io

img = Image.open(ref_image_path).convert("RGB").resize((360, 640))
buf = io.BytesIO()
img.save(buf, "JPEG", quality=50)
img_b64 = base64.b64encode(buf.getvalue()).decode()
```

## Submit Video Segment

```python
payload = {
    "model": "grok-imagine-video",
    "prompt": f"He talks to the camera saying: {text} {SCENE_DESC}",
    "image": {"url": f"data:image/jpeg;base64,{img_b64}"},
    "duration": 4,
    "aspect_ratio": "9:16",
    "resolution": "720p",
    "seed": SEED
}

result = subprocess.run([
    'curl', '-s', '--max-time', '120', '-X', 'POST',
    'https://api.x.ai/v1/videos/generations',
    '-H', f'Authorization: Bearer {XAI_KEY}',
    '-H', 'Content-Type: application/json',
    '-d', json.dumps(payload)
], capture_output=True, text=True)

req_id = json.loads(result.stdout).get('request_id')
```

## Poll Video Completion

```python
import requests, time

headers = {"Authorization": f"Bearer {XAI_KEY}"}
for i in range(40):
    time.sleep(10)
    r = requests.get(f"https://api.x.ai/v1/videos/{req_id}", headers=headers, timeout=30)
    data = r.json()
    if data.get("status") == "done":
        video_url = data["video"]["url"]
        subprocess.run(['curl', '-s', '-L', '--max-time', '120', video_url, '-o', outpath])
        break
    elif data.get("status") == "failed":
        break
```

## Concat Segments (Correct Order)

```python
concat_file = "/tmp/concat.txt"
with open(concat_file, "w") as f:
    for i in range(1, num_segments + 1):
        f.write(f"file '{base_dir}/seg{i}.mp4'\n")

subprocess.run([
    'ffmpeg', '-y', '-f', 'concat', '-safe', '0', '-i', concat_file,
    '-c:v', 'libx264', '-preset', 'fast', '-crf', '18',
    '-c:a', 'aac', '-b:a', '128k',
    '-pix_fmt', 'yuv420p', '-movflags', '+faststart',
    final_path
])
```

## Remove Dead Space

```python
import re

# Detect silence
result = subprocess.run([
    'ffmpeg', '-i', input_path, '-af', 'silencedetect=noise=-30dB:d=0.3', '-f', 'null', '-'
], capture_output=True, text=True)

starts = [float(x) for x in re.findall(r'silence_start: ([\d.]+)', result.stderr)]
ends = [float(x) for x in re.findall(r'silence_end: ([\d.]+)', result.stderr)]

# Build speaking segments
speaking = []
prev_end = 0.0
for s, e in zip(starts, ends):
    if s > prev_end + 0.05:
        speaking.append((prev_end, s))
    prev_end = e
if prev_end < total_dur:
    speaking.append((prev_end, total_dur))

# Extract and concat speaking segments only
for i, (s, e) in enumerate(speaking):
    dur = e - s
    if dur < 0.1:
        continue
    subprocess.run([
        'ffmpeg', '-y', '-ss', str(s), '-t', str(dur), '-i', input_path,
        '-c:v', 'libx264', '-preset', 'fast', '-crf', '18',
        '-c:a', 'aac', '-b:a', '128k',
        f'/tmp/speak-{i:02d}.mp4'
    ])
```

## Send to Telegram

```python
config_path = os.path.expanduser('~/.openclaw/openclaw.json')
with open(config_path) as f:
    tg_token = json.load(f)['channels']['telegram']['botToken']

subprocess.run([
    'curl', '-s', '-X', 'POST',
    f'https://api.telegram.org/bot{tg_token}/sendVideo',
    '-F', 'chat_id=5672156514',
    '-F', f'video=@{video_path}',
    '-F', f'caption={caption}',
    '-F', 'supports_streaming=true'
])
```
