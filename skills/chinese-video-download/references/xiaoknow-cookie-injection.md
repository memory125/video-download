# Xiaoeknow Cookie Injection & Stream Extraction

## Session: 2026-06-08 — Video Download Attempt for appswddscls8484

### Problem Statement
User provided raw cookie JSON from Chrome DevTools. Goal: download a past live recording from Xiaoeknow platform.

### Cookie Injection Pattern That Works

**CRITICAL**: `browser_navigate()` resets all cookies. Use JS navigation to preserve auth state.

```javascript
// Step 1: Navigate to login page (establishes domain context)
browser_navigate("https://appswddscls8484.h5.xiaoeknow.com/login")

// Step 2: Inject ALL auth cookies via document.cookie
document.cookie = 'ko_token=VALUE;path=/';
document.cookie = 'ctx_user_id=VALUE;path=/';
document.cookie = 'colla_login=1;path=/';
document.cookie = 'regtime=VALUE;path=/';
// ... inject all remaining cookies

// Step 3: Navigate to video page using JS (preserves cookies!)
window.location.href = 'https://appswddscls8484.h5.xiaoeknow.com/v4/course/alive/...';
```

**DO NOT** use `browser_navigate()` for step 3 — it creates a fresh session and loses all injected cookies.

### Key Auth Cookies (from this session)
- `ko_token` — PRIMARY auth token (marked secure=true, HTTPS only)
- `ctx_user_id` — User identifier (format: `u_69f9a8106ce7d_xDKmJEusCw`)
- `colla_login` — Login status flag (value: `1`)
- `regtime`, `logintime` — Timestamps for session validation

### API Endpoint Discovered
```
GET https://APPID.h5.xiaoeknow.com/_alive/v3/base_info?resource_id=ALIVE_ID&product_id=&type=12&is_direct=1&file_tag=1
```
Returns: `alive_play` object with stream URLs (m3u8, flv) in multiple quality levels and CDN lines.

### Live vs Recording Issue
- API returns LIVE stream URLs even for ended sessions (`alive_state: 3`)
- These return **404** when session has ended, despite `is_lookback: 1`
- Actual recording URL is served via blob wrapper in browser player — NOT directly accessible from API
- Workaround needed: intercept network requests in authenticated browser OR find VOD-specific endpoint

### yt-dlp Cookie File Bug
All cookies rejected with: `WARNING: skipping cookie file entry due to invalid expires at FALSE`
- Root cause: yt-dlp misparses Netscape format `FALSE` discard column
- Fix: Use Unix line endings, proper tabs (NOT heredoc in bash)
- Alternative: curl with `-b "cookie1=val; cookie2=val"` string

### Video Info from This Session
- Course: 实战心法 闭门训练营 (Practical Mindset Closed-Door Training Camp)
- Instructor: 刘伯涛 (Liu Bota)
- Duration: ~2.7 hours (9722 seconds)
- Platform: 小鹅通 (Xiaoeknow) appswddscls8484
