---
name: video-download-pipeline
description: "Complete end-to-end video download pipeline for Chinese platforms (小鹅通/Xiaoeknow, Bilibili, etc.) with gateway stability safeguards. Handles platform detection, authentication via cookies or SMS login, stream URL extraction, and reliable downloads using yt-dlp/ffmpeg."
category: media
tags: [video-download, chinese-platforms, xiaoe, cookie-auth, gateway-stability, pipeline]
created_by: agent
---

# Video Download Pipeline — Complete Workflow

End-to-end skill for downloading videos from Chinese education and content platforms. Encapsulates the full process: platform detection → authentication → stream extraction → download → verification. Includes gateway stability safeguards to prevent Hermes Gateway crashes during long-running operations.

## TRIGGERS

- User shares a URL to a Chinese video/education platform
- URL contains `xetslk.com`, `xiaoe`, `xiaoeknow`, `bilibili.com`, or similar
- User says "下载视频" (download video), "帮我下载", or shares a course link
- User provides cookies for an authenticated session

## PREREQUISITES CHECK

Before starting, verify tools are available:

```bash
# Check yt-dlp
yt-dlp --version 2>&1 | head -1

# Check ffmpeg (required for HLS/m3u8 streams)
ffmpeg -version 2>&1 | head -1

# If missing, install:
pip install yt-dlp    # or: pipx install yt-dlp
```

## GATEWAY STABILITY SAFEGUARDS

**Critical**: Long-running video downloads can cause Hermes Gateway to become unresponsive. Apply these safeguards:

### 1. Increase timeouts for download sessions

When running via gateway, the default `gateway_timeout: 1800` (30 min) may be insufficient for large videos. Ensure config has:

```yaml
agent:
  max_turns: 60           # Allow enough turns for complex workflows
  gateway_timeout: 1800   # 30 minutes — sufficient for most downloads
  api_max_retries: 3      # Retry failed API calls
terminal:
  timeout: 300            # 5 min per command — ffmpeg needs this
```

### 2. Run downloads in background with notification

For downloads expected to take >1 minute, use `terminal(background=true, notify_on_complete=true)` instead of foreground commands:

```python
# GOOD — runs in background, notifies when done
terminal(command='yt-dlp "URL" -o "~/Downloads/output.mp4"', 
         background=True, notify_on_complete=True)

# AVOID for long downloads — blocks the gateway
terminal(command='yt-dlp "URL" -o "~/Downloads/output.mp4"', timeout=300)
```

### 3. Monitor gateway health during operations

Before starting a download session:
```bash
hermes gateway status 2>&1
```

If gateway is not running, start it:
```bash
hermes gateway start 2>&1
```

During long downloads, check process status with `process(action='poll')` instead of waiting.

### 4. Cookie-based auth avoids browser overhead

When the user provides cookies (most common case), use them directly with curl/yt-dlp instead of launching a browser session — this is faster and more stable:

```bash
# Convert cookie JSON to Netscape format for yt-dlp
python3 -c "
import json, sys
cookies = json.loads(open(sys.argv[1]).read())
with open('cookies.txt', 'w') as f:
    f.write('# Netscape HTTP Cookie File\n')
    for c in cookies:
        domain = c['domain']
        secure = 'TRUE' if c.get('secure') else 'FALSE'
        path = c.get('path', '/')
        if c.get('session'):
            expires = 0
        else:
            expires = int(c.get('expirationDate', 0))
        httponly = 'TRUE' if c.get('httpOnly') else 'FALSE'
        f.write(f'{domain}\t{secure}\t{path}\t{expires}\t{httponly}\t{c[\"name\"]}\t{c[\"value\"]}\n')
print('Written cookies.txt')
" cookie_json_file.json

# Use with yt-dlp
yt-dlp --cookies cookies.txt "URL" -o "output.mp4"
```

## WORKFLOW — STEP BY STEP

### Phase 1: Platform Detection & URL Resolution

```bash
# Check if URL redirects (many Chinese platforms use short links)
curl -sL -o /dev/null -w "%{url_effective}" "USER_PROVIDED_URL"
```

**Platform identification matrix:**

| Domain Pattern | Platform | Auth Required? | Strategy |
|---|---|---|---|
| `*.h5.xiaoeknow.com` | 小鹅通 (Xiaoeknow) | Yes — cookies or SMS | Cookie → API → Browser |
| `xetslk.com/sl/*` | Xiaoeknow short link | Yes | Resolve redirect first |
| `bilibili.com/video/*` | Bilibili public | No | yt-dlp directly |
| `bilibili.com/v/*` (paid) | Bilibili paid | Yes — cookies | Cookie-based |
| `v.qq.com` | Tencent Video VIP | Yes | Browser extraction |

### Phase 2: Try Fastest Method First (yt-dlp)

```bash
# Attempt direct download — works for public content or with cookies
yt-dlp --no-check-certificates \
  --cookies cookies.txt \
  -o "~/Downloads/%(title)s.%(ext)s" \
  "URL" 2>&1 | tail -20
```

**Expected outcomes:**
- ✅ Success → Skip to Phase 5 (Verification)
- ❌ `UnsupportedError` / redirect to login → Proceed to Phase 3
- ⚠️ Falls back to generic extractor → Proceed to Phase 3

### Phase 3: Cookie-Based API Extraction (Xiaoeknow Specific)

When user provides cookies, extract video stream via API calls:

```bash
# Step 1: Build cookie string from JSON
COOKIE_STRING=""
for cookie in "${cookies[@]}"; do
  COOKIE_STRING+="${cookie.name}=${cookie.value}; "
done

# Step 2: Try course detail API
curl -sL "https://APPID.h5.xiaoeknow.com/api/h5/course/getCourseDetail?course_id=COURSE_ID&app_id=APPID" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -H "Accept: application/json" \
  -H "Referer: https://APPID.h5.xiaoeknow.com/" \
  -b "$COOKIE_STRING"

# Step 3: Try live detail API (for recordings)
curl -sL "https://APPID.h5.xiaoeknow.com/api/h5/live/getLiveDetail?live_id=LIVE_ID&app_id=APPID" \
  -H "User-Agent: Mozilla/5.0 ..." \
  -b "$COOKIE_STRING"

# Step 4: Try v4 alive endpoint
curl -sL "https://APPID.h5.xiaoeknow.com/v4/course/alive/LIVE_ID?app_id=APPID" \
  -H "User-Agent: Mozilla/5.0 ..." \
  -b "$COOKIE_STRING" | python3 -m json.tool
```

**What to look for in API response:**
- `play_url` or `video_url` — direct MP4 link
- `m3u8_url` or HLS playlist URL
- `chapters[]` with individual video URLs (multi-part courses)

### Phase 4: Browser-Based Extraction (Last Resort)

When cookies + API fail, use browser automation:

```bash
# Use Hermes built-in browser tools
browser_navigate(url="TARGET_URL")
browser_snapshot(full=true)

# Extract video source from DOM
browser_console(expression="
  // Find all video elements and their sources
  const videos = document.querySelectorAll('video');
  const result = [];
  videos.forEach(v => {
    result.push({src: v.src, currentSrc: v.currentSrc});
  });
  
  // Check for HLS streams in JS variables
  if (window.GlobalState) {
    result.push({globalState: JSON.stringify(window.GlobalState).substring(0, 500)});
  }
  if (window.__INITIAL_STATE__) {
    result.push({initialState: JSON.stringify(window.__INITIAL_STATE__).substring(0, 500)});
  }
  
  JSON.stringify(result);
")
```

### Phase 5: Download the Stream

Once you have a direct video URL or HLS stream:

```bash
# For direct MP4 URLs:
yt-dlp "VIDEO_URL" -o "~/Downloads/output.mp4"

# For HLS/m3u8 streams (requires ffmpeg):
ffmpeg -i "M3U8_URL" -c copy -bsf:a aac_adtstoasc "~/Downloads/output.mp4"

# Background mode for long downloads:
terminal(command='yt-dlp "URL" -o "~/Downloads/output.mp4"', 
         background=True, notify_on_complete=True)
```

### Phase 6: Verification

```bash
# Check file exists and has reasonable size
ls -lh ~/Downloads/output.mp4
file ~/Downloads/output.mp4

# Verify it's a valid video (check duration)
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 \
  ~/Downloads/output.mp4
```

## COOKIE HANDLING — COMPLETE REFERENCE

### Xiaoeknow Cookie Structure

Required cookies for authenticated access:

| Cookie Name | Purpose | Critical? |
|---|---|---|
| `ko_token` | Session authentication token | **Yes** |
| `ctx_user_id` | User identity | **Yes** |
| `colla_login` | Login status flag | Yes |
| `logintime` | Last login timestamp | No |
| `regtime` | Registration timestamp | No |
| `requestId` | Request tracking ID | No |
| `sensorsdata2015jssdkcross` | Analytics tracking | No |

**Minimum viable cookie set**: Only `ko_token` and `ctx_user_id` are strictly required for API calls. Other cookies are optional but recommended for full compatibility.

### Converting Browser Cookie Export to Netscape Format

When user pastes Chrome/Edge cookie JSON (from DevTools → Application → Cookies):

```python
import json, datetime

# User provides JSON array of cookie objects
cookies_json = '''[PASTED_JSON]'''
cookies = json.loads(cookies_json)

with open('cookies.txt', 'w') as f:
    f.write('# Netscape HTTP Cookie File\n')
    f.write('# This file is generated by Hermes video-download-pipeline\n\n')
    
    for c in cookies:
        domain = c['domain']
        secure = 'TRUE' if c.get('secure', False) else 'FALSE'
        path = c.get('path', '/')
        
        if c.get('session', False):
            expires = 0
        else:
            expires = int(c.get('expirationDate', 0))
        
        httponly = 'TRUE' if c.get('httpOnly', False) else 'FALSE'
        
        f.write(f'{domain}\t{secure}\t{path}\t{expires}\t{httponly}\t{c["name"]}\t{c["value"]}\n')

print(f'Written {len(cookies)} cookies to cookies.txt')
```

## COMMON ERROR PATTERNS AND FIXES

### Error: "Unsupported URL" from yt-dlp
**Cause**: Platform redirects to login page; yt-dlp generic extractor fails.  
**Fix**: Use cookie-based approach (Phase 3) or browser extraction (Phase 4).

### Error: Cookie expired / session invalid
**Cause**: Cookies have short expiration (typically 2-8 hours).  
**Fix**: Ask user for fresh cookies from their browser.

### Error: HLS stream token expired
**Cause**: m3u8 URLs contain time-limited tokens.  
**Fix**: Extract and download immediately — don't store URLs for later.

### Error: Gateway timeout during ffmpeg download
**Cause**: Large files take longer than `gateway_timeout`.  
**Fix**: Use `terminal(background=True, notify_on_complete=True)` pattern.

### Error: "Failed to connect" / network error
**Cause**: Chinese platforms may block non-Chinese IPs or require specific User-Agent.  
**Fix**: Add proper headers and consider proxy settings.

## OUTPUT CONVENTIONS

- Download directory: `~/Downloads/` (or user-specified)
- File naming: Use video title when available, fallback to timestamp-based name
- Format: Preserve original format (MP4 preferred); use `-c copy` for HLS to avoid re-encoding
- Report: Always confirm download completion with file size and duration

## INTEGRATION WITH EXISTING SKILLS

This skill **complements** `chinese-video-download` by:
1. Adding gateway stability safeguards
2. Providing a complete end-to-end workflow (not just extraction)
3. Including cookie handling automation
4. Adding verification steps

Load both skills when working with Chinese video platforms for best results.
