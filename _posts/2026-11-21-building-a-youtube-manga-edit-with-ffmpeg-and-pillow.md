---
layout: post
title: "Make a YouTube Manga Edit with Only ffmpeg and Pillow"
date: 2026-11-21 00:00:00 +0530
categories: ffmpeg video python anime
---

A 21-second "Gojo vs Sukuna" manga edit for YouTube, built with two tools you may already have: [ffmpeg](https://ffmpeg.org/download.html) and Python's [Pillow](https://pillow.readthedocs.io/en/stable/reference/Image.html). No video editor, no ripped panels, no trending audio — original art and a synthesized beat, so nothing for Content ID to claim.

Every command below is copy-pasteable. Follow it top to bottom and you end up with a YouTube-ready `gojo-vs-sukuna-edit.mp4`.

## What You're Building

| Shot | Length | Visual |
|---|---:|---|
| 1. Title | 2.5 s | Split blue/red title card, radial speed lines |
| 2. Gojo | 3.5 s | Hollow Purple rings, blue impact frame |
| 3. Sukuna | 3.5 s | Dismantle cross-slashes, red impact frame |
| 4. Domain clash | 3.5 s | Unlimited Void × Malevolent Shrine split-screen |
| 5. Infinity vs Cleave | 4.0 s | Purple clash frame |
| 6. Outro | 4.0 s | "WHO WINS?" comment/subscribe card |

Every shot gets a punch-zoom, camera shake, film grain, letterbox bars, and white-flash cuts, over a synthesized 140 BPM beat.

## Step 0: Prerequisites

macOS or Linux, then:

```bash
brew install ffmpeg          # https://docs.brew.sh/Manpage
python3 -m pip install Pillow
```

One ffmpeg build check before you go further — some builds (including Homebrew's) ship **without** the text-rendering filter:

```bash
ffmpeg -hide_banner -filters | grep drawtext
```

If that prints nothing, `drawtext` is unavailable (`No such filter: 'drawtext'` when you use it). This tutorial doesn't care: all typography happens in Pillow, and ffmpeg only does motion. Either way, continue.

Make the project:

```bash
mkdir -p gojo-vs-sukuna-edit/{assets/panels,tmp,out}
cd gojo-vs-sukuna-edit
```

## Step 1: Render the Six Panels with Pillow

Each shot is one 2000×1160 PNG — text, speed lines, slashes, and rings baked in. Save this as `make_panels.py`:

```python
from pathlib import Path
from PIL import Image, ImageDraw, ImageFont
import math, random

OUT = Path(__file__).parent / "assets" / "panels"
OUT.mkdir(parents=True, exist_ok=True)
W, H = 2000, 1160

# Swap for any bold display font on Linux (e.g. /usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf)
f_big = ImageFont.truetype("/System/Library/Fonts/Supplemental/Impact.ttf", 210)
f_med = ImageFont.truetype("/System/Library/Fonts/Supplemental/Impact.ttf", 130)
f_sub = ImageFont.truetype("/System/Library/Fonts/Supplemental/Arial Bold.ttf", 48)
f_tag = ImageFont.truetype("/System/Library/Fonts/Supplemental/Arial Bold.ttf", 34)

def speed_lines(d, cx, cy, color, n=90, r0=260, w=3, seed=1):
    """Radial manga speed lines bursting from (cx, cy)."""
    rng = random.Random(seed)
    for i in range(n):
        a = (2 * math.pi * i / n) + rng.uniform(-0.02, 0.02)
        ri = r0 + rng.randint(-40, 80)
        d.line([(cx + ri*math.cos(a), cy + ri*math.sin(a)),
                (cx + 1500*math.cos(a), cy + 1500*math.sin(a))],
               fill=color, width=rng.randint(2, w+2))

def centered(d, y, text, font, fill=(255,255,255), stroke=(0,0,0), sw=8):
    bbox = d.textbbox((0,0), text, font=font, stroke_width=sw)
    d.text(((W - (bbox[2]-bbox[0])) // 2, y), text, font=font,
           fill=fill, stroke_width=sw, stroke_fill=stroke)

def manga_frame(d, accent):
    """Comic-panel border: white outer, accent inner."""
    d.rectangle([60, 110, W-60, H-110], outline=(255,255,255), width=5)
    d.rectangle([74, 124, W-74, H-124], outline=accent, width=3)

# 1. Title — split blue/red, one diagonal slash down the middle
im = Image.new("RGB", (W, H), (8, 8, 12)); d = ImageDraw.Draw(im)
speed_lines(d, W//2, H//2, (70, 70, 90), n=110, r0=320, seed=10)
d.polygon([(0,0),(W//2+80,0),(W//2-80,H),(0,H)], fill=(12, 22, 55))
d.polygon([(W//2+80,0),(W,0),(W,H),(W//2-80,H)], fill=(55, 10, 14))
speed_lines(d, W//2, H//2, (220, 220, 240), n=70, r0=380, w=2, seed=11)
d.line([(W//2+80,0),(W//2-80,H)], fill=(255,255,255), width=14)
manga_frame(d, (200,200,220))
centered(d, 250, "JJK // SHINJUKU SHOWDOWN", f_tag, fill=(255,210,80), sw=4)
centered(d, 380, "GOJO  VS  SUKUNA", f_big)
centered(d, 670, "THE HONORED ONE  //  KING OF CURSES", f_sub, fill=(225,225,235))
im.save(OUT/"p1.png")

# 2. Gojo — nested Hollow Purple rings
im = Image.new("RGB", (W, H), (6, 16, 46)); d = ImageDraw.Draw(im)
speed_lines(d, W//2, H//2, (70, 160, 255), n=120, r0=220, w=4, seed=22)
for r, col, w in [(420,(40,120,255),6),(320,(110,190,255),10),
                  (220,(200,240,255),14),(120,(255,255,255),6)]:
    d.ellipse([W//2-r, H//2-r, W//2+r, H//2+r], outline=col, width=w)
manga_frame(d, (90, 180, 255))
centered(d, 230, "SATORU GOJO — SIX EYES", f_tag, fill=(130,210,255), sw=4)
centered(d, 370, "LIMITLESS", f_big, fill=(235,248,255))
centered(d, 660, "CURSED TECHNIQUE LAPSE: BLUE  //  HOLLOW PURPLE", f_sub, fill=(170,220,255))
im.save(OUT/"p2.png")

# 3. Sukuna — Dismantle cross-slashes
im = Image.new("RGB", (W, H), (44, 6, 10)); d = ImageDraw.Draw(im)
speed_lines(d, W//2, H//2, (220, 40, 50), n=120, r0=240, w=4, seed=33)
for (p1, p2, w) in [((180,980),(1820,180),18), ((260,1060),(1900,260),10),
                    ((100,880),(1740,80),10), ((220,180),(1780,980),14),
                    ((340,140),(1880,920),8)]:
    d.line([p1, p2], fill=(255, 60, 60), width=w+6)
    d.line([p1, p2], fill=(255, 230, 230), width=max(2, w//2))
manga_frame(d, (255, 70, 70))
centered(d, 230, "RYOMEN SUKUNA — DISMANTLE & CLEAVE", f_tag, fill=(255,150,150), sw=4)
centered(d, 370, "KING OF CURSES", f_big, fill=(255,235,235))
centered(d, 660, "DOMAIN EXPANSION: MALEVOLENT SHRINE", f_sub, fill=(255,180,180))
im.save(OUT/"p3.png")

# 4. Domain clash — blue/red split screen
im = Image.new("RGB", (W, H), (10, 10, 16)); d = ImageDraw.Draw(im)
d.polygon([(0,0),(W//2+120,0),(W//2-120,H),(0,H)], fill=(10, 28, 78))
d.polygon([(W//2+120,0),(W,0),(W,H),(W//2-120,H)], fill=(78, 12, 18))
speed_lines(d, W//4, H//2, (90,180,255), n=70, r0=160, seed=44)
speed_lines(d, 3*W//4, H//2, (255,80,80), n=70, r0=160, seed=45)
d.line([(W//2+120,0),(W//2-120,H)], fill=(255,255,255), width=22)
manga_frame(d, (255,255,255))
centered(d, 230, "BARRIERLESS  VS  SURE-HIT VOID", f_tag, fill=(255,220,110), sw=4)
centered(d, 370, "DOMAIN CLASH", f_big)
centered(d, 660, "UNLIMITED VOID  ×  MALEVOLENT SHRINE", f_sub)
im.save(OUT/"p4.png")

# 5. Infinity vs Cleave — purple clash frame
im = Image.new("RGB", (W, H), (28, 8, 52)); d = ImageDraw.Draw(im)
speed_lines(d, W//2, H//2, (190, 90, 255), n=140, r0=180, w=5, seed=55)
for r in (460, 360, 260, 160):
    d.ellipse([W//2-r, H//2-r, W//2+r, H//2+r], outline=(220,140,255), width=8)
d.line([(160,1000),(1840,160)], fill=(255,255,255), width=16)
d.line([(160,160),(1840,1000)], fill=(255,90,120), width=12)
manga_frame(d, (210, 120, 255))
centered(d, 230, "200% HOLLOW PURPLE  //  WORLD-BISECTING SLASH", f_tag, fill=(230,170,255), sw=4)
centered(d, 385, "INFINITY VS CLEAVE", f_med)
centered(d, 640, "THE STRONGEST IN HISTORY  VS  THE STRONGEST TODAY", f_sub)
im.save(OUT/"p5.png")

# 6. Outro — CTA card
im = Image.new("RGB", (W, H), (6, 6, 8)); d = ImageDraw.Draw(im)
speed_lines(d, W//2, H//2, (120, 120, 140), n=90, r0=300, seed=66)
manga_frame(d, (255, 210, 80))
centered(d, 240, "STAND PROUD — YOU ARE STRONG", f_tag, fill=(255,210,80), sw=4)
centered(d, 370, "WHO WINS?", f_big)
centered(d, 660, "DROP YOUR WINNER IN THE COMMENTS  •  SUBSCRIBE", f_sub, fill=(230,230,230))
im.save(OUT/"p6.png")
print("Rendered p1..p6")
```

Run it:

```bash
python3 make_panels.py   # -> assets/panels/p1.png ... p6.png
```

Everything above is stock [Pillow `ImageDraw`](https://pillow.readthedocs.io/en/stable/reference/ImageDraw.html) primitives: `line`, `ellipse`, `polygon`, `rectangle`, `text` with `stroke_width`. Change any string, color, or ring radius and re-run — panels regenerate in under a second. Swap `p1.png`–`p6.png` for your own panels later and the rest of the pipeline doesn't change.

## Step 2: Animate Each Panel with ffmpeg

Save this as `build.sh`. It turns each PNG into a clip, then concatenates:

```bash
#!/bin/bash
set -euo pipefail
cd "$(dirname "$0")"
mkdir -p out tmp assets/panels

python3 make_panels.py

# Punch-zoom + handheld shake via an animated crop window
SHAKE="scale=2160:1252,crop=1920:1080:x='(in_w-out_w)/2+12*sin(2*PI*9*t)+28*exp(-7*t)*cos(2*PI*17*t)':y='(in_h-out_h)/2+10*cos(2*PI*11*t)+22*exp(-7*t)*sin(2*PI*19*t)'"
# Grain + vignette + 2.35:1 bars
GRADE="noise=alls=9:allf=t,vignette=PI/4.5,drawbox=y=0:w=iw:h=86:c=black:t=fill,drawbox=y=ih-86:w=iw:h=86:c=black:t=fill,format=yuv420p"

scene() { # $1=png $2=dur $3=in_color $4=out_color $5=out
  local fade_out; fade_out=$(python3 -c "print($2-0.08)")
  ffmpeg -hide_banner -loglevel error -y -loop 1 -t "$2" -r 30 -i "assets/panels/$1" \
    -vf "${SHAKE},${GRADE},fade=t=in:st=0:d=0.08:color=$3,fade=t=out:st=${fade_out}:d=0.08:color=$4,setsar=1" \
    -an -c:v libx264 -preset veryfast -crf 16 -pix_fmt yuv420p "tmp/$5"
}

scene p1.png 2.5 black white s1.mp4
scene p2.png 3.5 white white s2.mp4
scene p3.png 3.5 white white s3.mp4
scene p4.png 3.5 white white s4.mp4
scene p5.png 4.0 white white s5.mp4
scene p6.png 4.0 white black s6.mp4

# Synthesized 140 BPM beat: kick + 808 + fifth + snare + hats in one aevalsrc
ffmpeg -hide_banner -loglevel error -y \
  -i tmp/s1.mp4 -i tmp/s2.mp4 -i tmp/s3.mp4 \
  -i tmp/s4.mp4 -i tmp/s5.mp4 -i tmp/s6.mp4 \
  -f lavfi -i "aevalsrc='0.75*sin(2*PI*55*t)*exp(-13*mod(t,0.428571))+0.35*sin(2*PI*55*t)+0.18*sin(2*PI*82.5*t)+0.22*(random(0)-0.5)*exp(-24*mod(t+0.428571,0.857143))+0.10*(random(1)-0.5)*exp(-35*mod(t,0.214285))':s=48000:d=21" \
  -filter_complex "[0:v][1:v][2:v][3:v][4:v][5:v]concat=n=6:v=1:a=0[v];[6:a]afade=t=in:st=0:d=0.3,afade=t=out:st=19.8:d=1.2,alimiter=limit=0.92[a]" \
  -map "[v]" -map "[a]" -c:v libx264 -preset medium -crf 18 -pix_fmt yuv420p \
  -c:a aac -b:a 192k -ar 48000 -movflags +faststart -shortest \
  out/gojo-vs-sukuna-edit.mp4

rm -rf tmp
echo "DONE: out/gojo-vs-sukuna-edit.mp4"
```

### How the motion works

The edit's whole feel comes from one [`crop`](https://ffmpeg.org/ffmpeg-filters.html#crop) window whose position is an expression of `t` (see the [ffmpeg expression docs](https://ffmpeg.org/ffmpeg-utils.html#Expressions)):

| Term | What it does |
|---|---|
| `12*sin(2*PI*9*t)` | Constant ~9 Hz handheld jitter |
| `28*exp(-7*t)*cos(2*PI*17*t)` | Punch-zoom: a violent 17 Hz wobble that decays in ~0.4 s, exactly as each shot lands |
| `scale=2160:1252` first | Oversizes the still so the crop window has room to wander |

Layered on every shot:

- [`noise=alls=9:allf=t`](https://ffmpeg.org/ffmpeg-filters.html#noise) — manga print grain, re-rolled each frame
- [`vignette`](https://ffmpeg.org/ffmpeg-filters.html#vignette) + two black [`drawbox`](https://ffmpeg.org/ffmpeg-filters.html#drawbox) strips — 2.35:1 cinematic bars
- [`fade`](https://ffmpeg.org/ffmpeg-filters.html#fade) `color=white, d=0.08` — the white-flash whip cut between shots (black fades on first/last shot for a clean open/close)

Shot lengths in the six `scene` calls are the edit's timeline — change `2.5`/`3.5`/`4.0` to re-cut the video.

### How the beat works

No audio files, no samples — one [`aevalsrc`](https://ffmpeg.org/ffmpeg-filters.html#aevalsrc) expression is the entire track. The trick is `exp(-k*mod(t, period))`: an exponential decay that re-triggers on a metronome schedule.

| Expression | Instrument | Timing |
|---|---|---|
| `0.75*sin(2*PI*55*t)*exp(-13*mod(t,0.428571))` | Kick, 55 Hz | Every beat — `0.428571 s = 60/140` |
| `0.35*sin(2*PI*55*t) + 0.18*sin(2*PI*82.5*t)` | Sustained 808 + fifth drone | Continuous |
| `0.22*(random(0)-0.5)*exp(-24*mod(t+0.428571,0.857143))` | Snare (white noise burst) | Beats 2 & 4 |
| `0.10*(random(1)-0.5)*exp(-35*mod(t,0.214285))` | Hi-hat | Double time |

`0.857143` is two beats at 140 BPM; offsetting by `0.428571` lands the snare off-beat. [`alimiter`](https://ffmpeg.org/ffmpeg-filters.html#alimiter) stops the mix clipping. It's 100% synthesized, so no Content ID claim, ever.

## Step 3: Run It

```bash
chmod +x build.sh
./build.sh
```

About a minute later, verify the deliverable:

```bash
ffprobe -v error -show_entries format=duration -show_entries stream=codec_name,width,height,r_frame_rate out/gojo-vs-sukuna-edit.mp4
```

Expected: `1920x1080`, `30/1`, `h264` + `aac`, `duration=21.0`. `-movflags +faststart` moves the index to the front, so YouTube starts processing immediately.

## Step 4: Upload

Drag `out/gojo-vs-sukuna-edit.mp4` to [youtube.com/upload](https://studio.youtube.com/channel/upload), or just paste the file path into the upload dialog. Paste-ready metadata:

```text
Title:       GOJO VS SUKUNA — THE STRONGEST DECIDED [Manga Edit]
Description: The Strongest Sorcerer of Today vs The Strongest Sorcerer in History.
             Unlimited Void × Malevolent Shrine — who takes it? Drop your winner below.
Tags:        #jjk #jujutsukaisen #gojo #sukuna #gojovssukuna #mangaedit #animeedit
```

Any of `assets/panels/p1.png`–`p6.png` (2000×1160) works as the thumbnail.

## Troubleshooting

- **`No such filter: 'drawtext'`** — your ffmpeg lacks libfreetype. This pipeline never uses it; typography is in Pillow. Check with `ffmpeg -filters | grep drawtext`.
- **`No such file or directory: Impact.ttf`** — Linux path differs; use DejaVu Sans Bold (`/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf`) or install a display font.
- **Beat sounds hissy on headphones** — `random(0)` is raw per-sample white noise. Cleaner: replace those two terms with a bandpassed `anoisesrc` input and `amix` it in.
- **Temp disk blowup** — six intermediate clips at `crf 16` can hit ~13 GB. Lower the `scene()` encoder to `-crf 20` or build on a drive with room.

## Source

- [ffmpeg filters reference](https://ffmpeg.org/ffmpeg-filters.html) — [`crop`](https://ffmpeg.org/ffmpeg-filters.html#crop), [`noise`](https://ffmpeg.org/ffmpeg-filters.html#noise), [`vignette`](https://ffmpeg.org/ffmpeg-filters.html#vignette), [`drawbox`](https://ffmpeg.org/ffmpeg-filters.html#drawbox), [`fade`](https://ffmpeg.org/ffmpeg-filters.html#fade), [`concat`](https://ffmpeg.org/ffmpeg-filters.html#concat), [`aevalsrc`](https://ffmpeg.org/ffmpeg-filters.html#aevalsrc), [`alimiter`](https://ffmpeg.org/ffmpeg-filters.html#alimiter)
- [ffmpeg expression syntax](https://ffmpeg.org/ffmpeg-utils.html#Expressions) — the `sin`/`exp`/`mod` time language every motion and audio expression above uses
- [Pillow `ImageDraw` reference](https://pillow.readthedocs.io/en/stable/reference/ImageDraw.html)
- [Homebrew manpage](https://docs.brew.sh/Manpage) — `brew install ffmpeg`
