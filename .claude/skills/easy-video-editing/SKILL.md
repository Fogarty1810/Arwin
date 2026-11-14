---
name: easy-video-editing
description: Simple video editing for short-form reels with FFmpeg + Python. Use when the person uploads a video and asks to transcribe it ("write down what they say"), add captions or pop-up captions, make it vertical / 9:16 / follow my face, cut silent parts or pauses, trim ("cut from 0:05 to 0:20"), join clips, add background music softly, speed up, make it smaller for WhatsApp, or grab a cover photo. Handles English and Malay speech.
---

# Easy Video Editing

The person is a beginner. Talk in plain, simple words. Don't show commands unless they ask.

## 0. Make sure the tools are there (every new session)

Scripts live in this skill's `scripts/` folder (call it `$S` below:
`S=<this skill's base directory>/scripts`). Check the tools quickly; install only if something is missing:

```bash
python3 -c "import faster_whisper, cv2, os; assert cv2.__version__ < '5' and os.path.exists(cv2.data.haarcascades + 'haarcascade_frontalface_default.xml')" 2>/dev/null \
  && fc-list | grep -qi montserrat && echo "tools OK" || {
  apt-get update -q >/dev/null; apt-get install -y -q ffmpeg fonts-montserrat >/dev/null
  pip install --break-system-packages -q faster-whisper "av<19" "opencv-contrib-python-headless<5" numpy; }
```

Why the pins: PyAV 19 breaks faster-whisper; OpenCV 5 drops the face detector file. `fonts-montserrat`
needs `apt-get update` first or it is "not found". The Whisper model downloads from Hugging Face on first
use (the environment must allow the huggingface.co / *.hf.co hosts). If it can't download, say:
"Speech-to-text needs the Hugging Face websites allowed in your cloud settings." Everything else still works.

## Where files are

- Uploads: `~/.claude/uploads/<session-id>/` (older setups: `/mnt/user-data/uploads/`). Find with
  `ls -t ~/.claude/uploads/*/ /mnt/user-data/uploads/ 2>/dev/null`.
- Never touch the original. Copy it to a work folder in the scratchpad.
- Put finished videos in `outputs/` in the project folder so the person can open them in the app
  (`outputs/` is git-ignored: videos are never committed). Name them clearly, e.g. `reel-captions.mp4`.

## Check your own work after every edit

```bash
ffmpeg -loglevel error -y -i out.mp4 -vf "fps=1/3,scale=270:-1,tile=6x1" -frames:v 1 sheet.jpg   # look at it
ffprobe -v error -show_entries format=duration:stream=codec_name,width,height -of csv=p=0 out.mp4
```

Reels: 1080x1920 (9:16), H.264 + AAC, `-pix_fmt yuv420p -movflags +faststart`.
Always remind them: "Watch it once before you post."

## Recipes

**Transcribe / captions file**
```bash
python3 $S/transcribe.py in.mp4 --lang en
python3 $S/transcribe.py in.mp4 --lang ms --model medium --prompt "names, brand words"   # Malay
```
Makes `in.srt` + `in_words.json`. Show the person the text and fix wrong names in the .srt by hand.
If they mix Malay and English ("rojak"), use `--lang ms --model medium`.

**Pop-up captions burned in** (video must already be 1080x1920; make it vertical first)
```bash
python3 $S/popcaps.py in.srt caps.ass --accent "FREE,PLAN"
ffmpeg -i in.mp4 -vf "ass=caps.ass" -c:v libx264 -crf 18 -pix_fmt yuv420p -movflags +faststart -c:a copy out.mp4
```

**Plain captions burned in** (any size)
```bash
ffmpeg -i in.mp4 -vf "subtitles=in.srt:force_style='FontName=Montserrat,FontSize=14,Bold=1,Outline=2,MarginV=60'" -c:v libx264 -crf 18 -pix_fmt yuv420p -c:a copy out.mp4
```

**Vertical + follow the face** (about 3 s of work per 1 s of video)
```bash
python3 $S/facetrack.py in.mp4 out.mp4      # calmer camera: --smooth 0.04 ; faster: --every 5
```
Already-vertical video stays as is. With two people it follows the biggest face. If it says
"no face found", it keeps the centre: offer the blurred-background version instead.

**Vertical with blurred background** (no cropping)
```bash
ffmpeg -i in.mp4 -filter_complex "[0]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,boxblur=20[bg];[0]scale=1080:-2[fg];[bg][fg]overlay=(W-w)/2:(H-h)/2" -c:v libx264 -crf 18 -pix_fmt yuv420p -c:a copy out.mp4
```

**Cut the pauses**
```bash
python3 $S/cutpauses.py in.mp4 out.mp4 --min 0.35
```

**Trim** (keep 0:05 to 0:20)
```bash
ffmpeg -ss 00:00:05 -to 00:00:20 -i in.mp4 -c:v libx264 -crf 18 -pix_fmt yuv420p -c:a aac -movflags +faststart out.mp4
```

**Join clips** (re-encodes, so clips from different phones still join)
```bash
ffmpeg -i a.mp4 -i b.mp4 -filter_complex "[0:v]scale=1080:1920:force_original_aspect_ratio=decrease,pad=1080:1920:(ow-iw)/2:(oh-ih)/2,setsar=1,fps=30[v0];[1:v]scale=1080:1920:force_original_aspect_ratio=decrease,pad=1080:1920:(ow-iw)/2:(oh-ih)/2,setsar=1,fps=30[v1];[0:a]aresample=48000[a0];[1:a]aresample=48000[a1];[v0][a0][v1][a1]concat=n=2:v=1:a=1[v][a]" -map "[v]" -map "[a]" -c:v libx264 -crf 18 -pix_fmt yuv420p -c:a aac -movflags +faststart out.mp4
```
More clips: add inputs, their `[vN]`/`[aN]` chains, and set `n=`. A clip with no sound needs a silent
track first: `ffmpeg -i clip.mp4 -f lavfi -i anullsrc=r=48000:cl=stereo -shortest -c:v copy -c:a aac clip-sound.mp4`.

**Music softly under the voice** (music ~15%, fades out over the last 2 s)
```bash
DUR=$(ffprobe -v error -show_entries format=duration -of csv=p=0 in.mp4)
ffmpeg -i in.mp4 -stream_loop -1 -i music.mp3 -filter_complex "[1:a]volume=0.15,afade=t=out:st=$(echo "$DUR-2" | bc):d=2[m];[0:a][m]amix=inputs=2:duration=first:normalize=0[a]" -map 0:v -map "[a]" -c:v copy -c:a aac out.mp4
```
`normalize=0` matters: without it amix halves the voice. Only use music the person owns or that is
royalty-free. Never put song lyrics on screen.

**Speed up 1.2x**
```bash
ffmpeg -i in.mp4 -filter_complex "[0:v]setpts=PTS/1.2[v];[0:a]atempo=1.2[a]" -map "[v]" -map "[a]" -c:v libx264 -crf 18 -pix_fmt yuv420p out.mp4
```

**Smaller for WhatsApp**
```bash
ffmpeg -i in.mp4 -vf "scale=-2:'min(1280,ih)'" -c:v libx264 -crf 28 -preset slow -pix_fmt yuv420p -c:a aac -b:a 96k -movflags +faststart small.mp4
```

**Cover photo**
```bash
ffmpeg -ss 00:00:03 -i in.mp4 -frames:v 1 -q:v 2 cover.jpg
```

## Rules
- Simple words. One-line explanation if something fails, then try the next simplest way.
- Keep the original untouched; clear output names.
- Captions sit in the lower middle so TikTok/Instagram buttons don't cover them.
