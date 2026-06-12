# 🎬 Video Download Skills Collection

Hermes Agent skills for downloading videos from Chinese and international platforms — handling authentication, stream extraction, cookie management, and gateway stability.

---

## Skills Included

### 1. `chinese-video-download`

Download videos from **Chinese education and content platforms** (小鹅通/Xiaoeknow, Bilibili, Tencent Video, etc.) with support for SMS authentication, API extraction, and stream URL discovery.

**Features:**
- 6-tier approach hierarchy: yt-dlp → direct API probing → browser SMS login → cookie-based extraction → browser recording extraction → gateway stability
- Xiaoeknow (小鹅通) platform support with cookie injection & stream extraction
- Bilibili playurl bypass for 412 Precondition Failed errors and member-exclusive videos
- Tencent Video handling

**Files:**
| File | Description |
|------|-------------|
| `SKILL.md` | Main skill definition (359 lines) |
| `references/bilibili-playurl-bypass.md` | Bilibili playurl API bypass technique |
| `references/xiaoknow-cookie-injection.md` | Cookie injection & stream extraction notes |
| `references/xiaoknow-api-patterns.md` | Xiaoeknow API endpoints and auth flow details |

---

### 2. `video-download-pipeline`

Complete **end-to-end video download pipeline** for Chinese platforms with gateway stability safeguards. Handles platform detection, authentication via cookies or SMS login, stream URL extraction, and reliable downloads using yt-dlp/ffmpeg.

**Features:**
- 6-phase workflow: platform detection → yt-dlp attempt → cookie-based API extraction → browser extraction → download → verification
- Gateway stability safeguards (timeout config, background downloads, health monitoring)
- Cookie handling automation including Python scripts for converting Chrome DevTools JSON to Netscape format
- Platform identification matrix covering Xiaoeknow, Bilibili, Tencent Video

**Files:**
| File | Description |
|------|-------------|
| `SKILL.md` | Main skill definition (320 lines) |

---

## Installation

### Option A: Hermes Skill Manager (Recommended)

```bash
# Install both skills via Hermes CLI
hermes skills install https://github.com/memory125/video-download --skill chinese-video-download
hermes skills install https://github.com/memory125/video-download --skill video-download-pipeline
```

### Option B: Manual Installation

```bash
# Clone this repo and symlink to your Hermes skills directory
git clone https://github.com/memory125/video-download.git
cd video-download

# Symlink each skill
ln -s $(pwd)/skills/chinese-video-download ~/.hermes/skills/chinese-video-download
ln -s $(pwd)/skills/video-download-pipeline ~/.hermes/skills/media/video-download-pipeline
```

### Option C: Third-Party Skill Installation

Use the `third-party-skill-installation` Hermes skill to handle installation, including bypassing Skills Guard false positives via clone + symlink.

---

## How They Work Together

These two skills are **complementary**:

1. **`chinese-video-download`** provides platform-specific knowledge — API patterns, auth flows, cookie handling, and bypass techniques for each Chinese platform.

2. **`video-download-pipeline`** provides the workflow structure — a 6-phase pipeline that orchestrates the download process with stability safeguards, gateway health monitoring, and verification steps.

Use `chinese-video-download` when you need deep platform-specific knowledge. Use `video-download-pipeline` when you need a reliable end-to-end workflow with error handling and recovery.

---

## Supported Platforms

| Platform | chinese-video-download | video-download-pipeline |
|----------|----------------------|------------------------|
| 小鹅通 (Xiaoeknow) | ✅ Full support | ✅ Pipeline integration |
| Bilibili (哔哩哔哩) | ✅ Playurl bypass | ✅ Pipeline integration |
| 腾讯视频 (Tencent Video) | ✅ API extraction | ✅ Pipeline integration |

---

## Requirements

- **yt-dlp** — Latest version recommended (`pip install yt-dlp`)
- **ffmpeg** — For stream merging and format conversion
- **Hermes Agent** — Browser tools for SMS login & cookie extraction workflows

---

## License

MIT
