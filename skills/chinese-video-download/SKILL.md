---
name: chinese-video-download
description: "Download videos from Chinese education and content platforms (小鹅通/Xiaoeknow, Bilibili, Tencent Video, etc.) — handling SMS authentication, API extraction, and stream URL discovery."
category: media
tags: [video-download, chinese-platforms, xiaoe, sms-auth, yt-dlp]
---

# Chinese Video Download — Auth-Gated Platforms

Download videos from Chinese education platforms, course sites, and content providers that require authentication (SMS login, WeChat auth, etc.). Covers 小鹅通 (Xiaoeknow), Bilibili paid courses, Tencent Video VIP, and similar platforms.

## TRIGGERS
- User shares a link to a Chinese video/education platform
- URL contains `xetslk.com`, `xiaoe`, `xiaoeknow`, or redirects to phone login
- User asks to download a course video, paid content, or live recording from China
- yt-dlp fails with "Unsupported URL" on a Chinese platform

## PLATFORM IDENTIFICATION

### Short-link redirectors (common patterns)
- `gnud0.xetslk.com/sl/XXXX` → redirects to 小鹅通 login page
- `t.cn`, `url.cn` → WeChat/Tencent short links, often point to course content
- `h5.xiaoeknow.com` → direct 小鹅通 H5 pages

### After redirect, identify the actual platform:
1. Check the redirect URL with `curl -sL -o /dev/null -w "%{url_effective}" <URL>`
2. The redirect target reveals the platform and content ID
3. Extract `app_id`, `course_id`/`live_id` from query parameters

## APPROACH HIERARCHY (try in order)

### 1. yt-dlp (fastest, works for public content)
```bash
yt-dlp --no-check-certificates "URL"
```
- Works if the video is publicly accessible or you have cookies
- Add `--cookies-from-browser chrome` if logged in on desktop browser
- Falls back to generic extractor and fails on auth-gated pages

### 2. Direct API probing (for platforms with public APIs)
Some platforms expose course/video details without full authentication:
```bash
# Xiaoeknow patterns — try these API endpoints
curl -sL "https://APPID.h5.xiaoeknow.com/api/h5/course/getCourseDetail?course_id=COURSE_ID&app_id=APPID" \
  -H "User-Agent: Mozilla/5.0 ..." \
  -H "Accept: application/json" \
  -H "Referer: https://APPID.h5.xiaoeknow.com/"

# Common API patterns to try:
/api/h5/course/getCourseDetail
/api/pc/live/getLiveDetail
/api/h5/live/getLiveDetail
/v4/course/alive/COURSE_ID
```
**Pitfall**: Most return 302 redirects without auth cookies. If all APIs redirect, move to browser approach.

### 3. Browser-based SMS login + video extraction (most reliable)
When content is behind phone verification:

#### Step-by-step flow:
1. **Navigate** to the URL with `browser_navigate(url)`
2. **Identify login form**: look for phone number field (`textbox "请输入手机号"`) and SMS button
3. **Enter phone number**: `browser_type(ref="PHONE_FIELD", text="NUMBER")`
4. **Request SMS code**: click the SMS button (usually next to verification input)
5. **Wait for user**: ask them to provide the SMS code from their phone
6. **Submit login**: enter code in verification field, click "登录"
7. **Navigate to video page**: after login, navigate to the actual content URL
8. **Extract video stream**: use `browser_console` to find video elements:
   ```javascript
   // Find video source URLs
   document.querySelectorAll('video source').forEach(s => console.log(s.src))
   
   // Check for HLS/m3u8 streams
   document.querySelector('video')?.src
   
   // Look for stream data in page JS variables
   window.__INITIAL_STATE__ || window.__NEXT_DATA__ || window.GlobalState
   ```
9. **Download**: once you have the direct video URL, use `yt-dlp` or `curl`:
   ```bash
   yt-dlp "VIDEO_URL" -o "output.mp4"
   # or for HLS streams:
   ffmpeg -i "STREAM_URL" -c copy output.mp4
   ```

### 4. Cookie-based approach (if user has existing session)

#### 4a. Browser cookie export → Netscape format for yt-dlp
If the user is already logged in on their desktop browser:

**Export from Chrome DevTools:**
1. Open Application tab → Cookies → select domain
2. Copy all cookies as JSON array (each entry has: domain, expirationDate, name, path, secure, value)
3. Convert to Netscape format using this EXACT column order that yt-dlp requires:

```
Domain<TAB>IncludeSubdomains<TAB>Path<TAB>Secure<TAB>Expires<TAB>Name<TAB>Value
```

**CRITICAL**: The `IncludeSubdomains` column must be `TRUE` if `hostOnly=false`, and `FALSE` if `hostOnly=true`. The `Secure` column is position 4 (NOT position 5). Many converters get this wrong.

**Cookie file MUST have Unix line endings (`\n`)** — Windows `\r\n` causes yt-dlp to reject all entries with "invalid expires at FALSE".

```bash
# After creating cookies.txt, fix line endings:
sed -i 's/\r$//' cookies.txt
```

Then use with yt-dlp:
```bash
yt-dlp --cookies cookies.txt "URL"
```

#### 4b. Direct API probing with curl (for Xiaoeknow specifically)
Xiaoeknow's `_alive/v3/base_info` endpoint returns stream URLs when called with proper auth cookies via curl:

```bash
AUTH_COOKIES="ko_token=XXX; ctx_user_id=u_XXX; colla_login=1; regtime=XXX; logintime=XXX; newuserdays=90; olduserdays=180; requestId=req_XXX; shop_version_type=4; xenbyfpfUnhLsdkZbX=0"

curl -sL "https://APPID.h5.xiaoeknow.com/_alive/v3/base_info?resource_id=LIVE_ID&product_id=&type=12&is_direct=1&file_tag=1" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -H "Accept: application/json" \
  -H "Referer: https://APPID.h5.xiaoeknow.com/v4/course/alive/LIVE_ID?app_id=APPID" \
  -H "X-Requested-With: XMLHttpRequest" \
  -b "$AUTH_COOKIES" | python -m json.tool
```

The response contains `data.alive_play` with all stream URLs (M3U8, FLV) across multiple CDN lines.

**⚠️ CRITICAL LIMITATION — Live vs Recording**: 
- The `_alive/v3/base_info` endpoint returns **LIVE stream URLs only**
- For ended sessions (`alive_state: 3`) with replay enabled (`is_lookback: 1`), these live URLs return **404 Not Found**
- The actual recording/VOD URL is served through a DIFFERENT mechanism that requires browser-based extraction (see section 5)

#### 4c. When cookies fail with curl but work in browser
If `ko_token` is marked as `secure: true`, curl may not send it correctly over HTTP stream URLs. The Xiaoeknow API accepts these cookies, but the actual video CDN (`liveplay.xiaoe-live.com`) does NOT require auth — it just returns 404 for ended streams.

### 5. Browser-based recording extraction (for ended live sessions with replay)
When `alive_state: 3` (ended) and `is_lookback: 1` (replay available), the recording URL is NOT in `_alive/v3/base_info`. It requires browser interaction:

**Method A — Network tab capture (RECOMMENDED):**
1. User opens video page in desktop Chrome while logged in
2. Opens DevTools → Network tab → filter by `m3u8` or `play`
3. Clicks play button on the video player
4. Captures the actual recording stream URL from network requests
5. Downloads with: `ffmpeg -i "STREAM_URL" -c copy output.mp4`

**Method B — Console extraction:**
Run in Chrome DevTools Console on the video page:
```javascript
performance.getEntriesByType('resource').filter(e => e.name.includes('m3u8') || e.name.includes('.flv')).map(e => e.name)
```

### 6. Browser automation limitations (IMPORTANT for headless approach)
When using `browser_navigate` + cookie injection:
- **Each `browser_navigate` resets all cookies** — fresh session every time
- Cookies set via JS on one page are LOST when navigating to another URL
- Workaround: Set cookies, then use `window.location.href = 'URL'` (JS navigation preserves cookies)
- **BUT**: Some Xiaoeknow pages block cookie access entirely (`SecurityError: Access is denied`) — especially login subdomains like `/p/t/free/...`
- Synchronous XMLHttpRequest from console can call APIs with injected cookies, but async `fetch` results are not captured by the console evaluator

**Bottom line**: For recording/VOD extraction on Xiaoeknow, **desktop browser Network tab capture (Method A above) is the most reliable approach**. Browser automation works for initial auth but struggles with dynamic content extraction.

## XIAOKNOW (小鹅通) SPECIFICS

### Platform structure
- Host: `APPID.h5.xiaoeknow.com` (each store has unique app_id)
- Content types: courses (`course`), live recordings (`alive`/`live`)
- Auth: phone number + SMS verification code
- After login, session stored in cookies

### Key page variables (from HTML source):
```javascript
window.APPID = 'appswddscls8484'        // Store ID
window.USERID = ''                       // Empty before login
window.__anony_logon = '0'              // Login status
window.GlobalState                       // App state object
```

### Video delivery:
- Videos are typically delivered via HLS (m3u8) streams
- Stream URLs are generated server-side after authentication
- Look for video player initialization in page JavaScript

## COMMON PITFALLS

### Bot detection on Xiaoeknow
- The platform uses Aegis/RUM monitoring (`window.__aegis_id`)
- Browser automation may trigger bot detection — use stealth headers
- If blocked, try accessing from a different user agent or wait before retrying

### yt-dlp generic extractor failure
- When yt-dlp hits a login page instead of video content, it falls back to the generic extractor and raises `UnsupportedError`
- This is expected behavior for auth-gated platforms — don't retry with different yt-dlp flags

### Empty browser snapshot after navigation
- Some pages return `(empty page)` from snapshot due to JS rendering requirements
- Recovery: check actual URL with `browser_console(expression="window.location.href")`, then use `browser_vision` for visual confirmation

### HLS stream expiration
- Video stream URLs often have short-lived tokens (minutes)
- Download immediately after extraction — don't store URLs for later use

### Browser cookie loss on navigation (CRITICAL)
- **Each `browser_navigate()` starts a FRESH session** — all cookies set via JS in the previous page are LOST
- Workaround: Set cookies AFTER landing on the target domain, then navigate WITHIN that session using `window.location.href = 'URL'` (JS navigation preserves cookies)
- Pattern that works:
  ```javascript
  // 1. Navigate to login page first
  browser_navigate("https://APPID.h5.xiaoeknow.com/login")
  // 2. Inject all auth cookies via document.cookie
  browser_console(expression="document.cookie='ko_token=...'; document.cookie='ctx_user_id=...'")
  // 3. Navigate to video page using JS (NOT browser_navigate)
  browser_console(expression="window.location.href='https://APPID.h5.xiaoeknow.com/v4/course/alive/...'")
  ```

### Live stream vs Recording URL mismatch
- Xiaoeknow's `_alive/v3/base_info` API returns LIVE stream URLs even for ended sessions (`alive_state: 3`)
- These live URLs return **404 Not Found** when the session has ended, even if `is_lookback: 1` (replay enabled)
- The actual recording/VOD URL is served through a DIFFERENT mechanism — often wrapped in blob URLs by the browser player
- To get the recording URL: intercept network requests in the authenticated browser session, or look for VOD-specific API endpoints

### yt-dlp cookie file parsing bug
- yt-dlp may reject ALL cookies with "skipping cookie file entry due to invalid expires at FALSE"
- Root cause: yt-dlp misinterprets the Netscape format's `FALSE` discard column as an expiration date
- Fix: Ensure cookie file uses Unix line endings (no `\r\n`) and proper tab separation
- **MUST start with header line:** `# Netscape HTTP Cookie File` — Python's `MozillaCookieJar` rejects files without it
- Use `printf` or Python to generate the file, NOT heredoc in bash (heredoc mangles tabs)
- Alternative: use curl with `-b "cookie1=val1; cookie2=val2"` string directly instead of cookie files

### Bilibili CSP blocks document.cookie injection
- On Bilibili pages, `document.cookie = 'SESSDATA=...; domain=.bilibili.com'` throws `SecurityError: Access is denied for this document`
- This prevents browser automation from injecting auth cookies via JS on Bilibili domains
- Workaround: Use Python `urllib.request.HTTPCookieProcessor` with a pre-built `http.cookiejar.CookieJar`, or use curl's `-b` flag directly

### yt-dlp 412 on Bilibili even with valid cookies
- yt-dlp may get `HTTP Error 412: Precondition Failed` on the WBI-signed playurl API, even with correct SESSDATA cookies
- Causes: proxy interference (e.g. `http://127.0.0.1:19828`), or yt-dlp's WBI signing not matching Bilibili's current implementation
- Workaround: Use Python `urllib.request` directly with cookie jar instead of yt-dlp for Bilibili authenticated requests

### Cookie domain matching for curl vs browser
- Cookies set with `domain=.appswddscls8484.h5.xiaoeknow.com` (leading dot = subdomain wildcard) work in browsers but may NOT be sent by curl
- For curl: use exact hostname without leading dot: `domain=appswddscls8484.h5.xiaoeknow.com`
- The `secure: true` flag on `ko_token` means it only sends over HTTPS — curl must use `https://` URLs

### UTF-8 encoding errors in browser console
- Xiaoeknow pages contain non-UTF-8 bytes that cause `'utf-8' codec can't decode byte 0xb2` errors
- This affects `browser_snapshot` and large HTML extraction via `browser_console`
- Workaround: use `browser_vision` for visual inspection instead of text-based snapshot when encoding fails

## GATEWAY STABILITY — CRITICAL FOR LONG DOWNLOADS

Long-running video downloads can cause Hermes Gateway to become unresponsive. Always:

1. **Use background mode** for downloads >1 minute: `terminal(background=True, notify_on_complete=true)`
2. **Check gateway status** before starting: `hermes gateway status`
3. **Increase terminal timeout** in config.yaml if needed: `terminal.timeout: 300`
4. **Prefer cookie-based auth** over browser sessions — faster and more stable

See `video-download-pipeline` skill for the complete workflow with stability safeguards.

## BILIBILI-SPECIFIC WORKAROUNDS

### yt-dlp 412 Precondition Failed
yt-dlp frequently fails with `HTTP Error 412: Precondition Failed` on Bilibili, even with custom user-agent headers. This is a persistent issue — don't waste retries adding more flags.

**Working workaround — browser-based stream extraction:**
1. Navigate to the video page with `browser_navigate(url)`
2. Extract the direct MP4 stream URL from the `<video>` element:
   ```javascript
   (function(){ const v = document.querySelector('video'); return JSON.stringify({src: v?.src, currentSrc: v?.currentSrc}); })()
   ```
3. Download immediately with ffmpeg (the token expires quickly):
   ```bash
   ffmpeg -y -referer "https://www.bilibili.com/video/BVxxxxx" \
     -user_agent "Mozilla/5.0 ..." \
     -i "STREAM_URL" -c copy "output.mp4"
   ```

### Charging/Paid Member Videos (充电会员专属视频) — CDN-LEVEL RESTRICTION

Some Bilibili videos require a paid "充电" subscription. The page shows: "该视频为「XXX」专属视频，试看中・开通「68元档包月充电」即可观看".

**⚠️ HARD LIMITATION (confirmed 2026-06):** Charging member exclusive videos (`is_upower_exclusive: True`) return **ONLY the ~5 min trial preview** from ALL API endpoints, even with valid authenticated cookies (SESSDATA, DedeUserID). The restriction is enforced at Bilibili's CDN level — `timelength` metadata shows full duration but actual `durl[0].length` is capped at ~300s. No known bypass exists without the specific creator's charging membership.

**Detection:** Check `is_upower_exclusive` in the view API response:
```bash
curl -sL "https://api.bilibili.com/x/web-interface/view?bvid=BVxxxxx" | python -c "import sys,json; d=json.load(sys.stdin)['data']; print('upower_exclusive:', d.get('is_upower_exclusive'))"
```

**Verification (always do this):** Compare `data.timelength` vs `data.durl[0].length`:
- If `durl[0].length << timelength`, you only got the trial preview
- Example: `timelength=2373548ms` (~40min) but `durl[0].length=299980ms` (~5min) = TRIAL ONLY

#### Method A: playurl API bypass (works for PUBLIC videos; TRIAL-ONLY for charging member exclusives)

1. **Get CID from view API**:
   ```bash
   curl -sL "https://api.bilibili.com/x/web-interface/view?bvid=BVxxxxx" \
     -H "User-Agent: Mozilla/5.0 ..." > /tmp/bv_info.json
   # Extract CID from response: data.cid
   ```

2. **Request playurl with CID**:
   ```bash
   curl -sL "https://api.bilibili.com/x/player/playurl?bvid=BVxxxxx&cid=CID&qn=80" \
     -H "User-Agent: Mozilla/5.0 ..." \
     -H "Referer: https://www.bilibili.com/video/BVxxxxx" > /tmp/playurl.json
   # Extract URL from response: data.durl[0].url
   ```

3. **⚠️ CRITICAL — Verify you got the FULL video, not just trial preview**:
   - Compare `data.timelength` (full duration in ms) vs `data.durl[0].length` (actual stream length in ms)
   - If `durl[0].length << timelength`, you only got the ~5 min trial preview
   - Example: `timelength=2373548ms` (~40min) but `durl[0].length=299980ms` (~5min) = TRIAL ONLY
   - **Charging member exclusive videos (充电会员专属视频) ALWAYS return trial-only streams without auth** — the CDN enforces this, not just the frontend UI

4. **Download with Python urllib** (RECOMMENDED — handles JSON-escaped URLs correctly):
   - The playurl JSON contains `\u0026` (escaped `&`) in the URL — shell tools like sed/grep struggle to unescape these reliably
   - Use a Python script that reads the JSON, extracts the URL natively, and downloads via urllib
   - See `references/bilibili-playurl-bypass.md` for the complete working script

5. **Key facts**:
   - API returns `result: "suee"` even without login cookies — this does NOT guarantee full video access
   - Quality is capped at 720P (quality=64) for unauthenticated requests
   - Stream tokens have short deadlines (~30 min) — download immediately after extraction
   - **Charging member videos (`is_upower_exclusive: True`) return TRIAL-ONLY even with authenticated cookies** — the CDN enforces ~300s limit regardless of auth status
   - DASH format is NOT available for charging member videos — only MP4 trial streams are returned
   - This bypass may be patched by Bilibili at any time; fall back to cookie-based auth if it stops working

#### Method B: Browser trial view extraction (trial portion only, ~5 min)

If the API bypass fails, extract from browser `<video>` element as fallback — this gives only the trial preview.

### Chinese filenames fail in MSYS/bash ffmpeg output
When using ffmpeg on Windows with MSYS bash, Chinese characters in output filenames cause "No such file or directory" errors. Use ASCII-safe filenames:
```bash
# BAD — fails silently
ffmpeg -i "URL" -c copy "/c/Users/david/Downloads/中文标题.mp4"

# GOOD — use pinyin or English
ffmpeg -i "URL" -c copy "/c/Users/david/Downloads/Zhongwen-Biaoti.mp4"
```

## TOOLS AND DEPENDENCIES

```bash
# yt-dlp (handles most video formats including HLS)
pip install yt-dlp

# ffmpeg (for HLS/m3u8 streams and format conversion)
# Usually pre-installed on Windows via Gyan build
ffmpeg -version
```

## REFERENCE FILES
- See `references/xiaoknow-api-patterns.md` for Xiaoeknow API endpoint patterns and authentication flow details
- See `references/xiaoknow-cookie-injection.md` for cookie injection techniques, browser navigation pitfalls, and live-vs-recording URL issues
- See `references/bilibili-playurl-bypass.md` for Bilibili playurl API bypass technique — downloads full videos without charging membership
