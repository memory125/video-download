# Xiaoeknow (小鹅通) API Patterns and Auth Flow

## Session Reference: 2026-06-08

### Short-link to actual URL mapping
```
Short link: https://gnud0.xetslk.com/sl/1y3LBw
Redirects to: https://appswddscls8484.h5.xiaoeknow.com/p/t/free/v1/basic-platform/h5_basic/login/auth?redirect_url=ENCODED_URL

Decoded redirect target:
https://appswddscls8484.h5.xiaoeknow.com/v4/course/alive/l_6a23ddbde4b0694c35150191?app_id=appswddscls8484&alive_mode=0&pro_id=&type=2
```

### Extracted parameters from redirect URL:
- `app_id`: appswddscls8484 (store identifier)
- `course_id`/`live_id`: l_6a23ddbde4b0694c35150191
- `alive_mode`: 0 (live recording mode)
- `type`: 2 (content type indicator)

### API endpoints tested (all require auth cookies):
| Endpoint | Result without auth |
|----------|-------------------|
| `/api/pc/course/getCourseDetail` | 302 redirect to homepage |
| `/api/h5/course/getCourseDetail` | 302 redirect |
| `/api/pc/live/getLiveDetail` | 302 redirect |
| `/api/h5/live/getLiveDetail` | 302 redirect |
| `/v4/course/alive/{id}` | Returns HTML login page (200) |

### Login form structure (from browser snapshot):
```
- textbox "请输入手机号" [phone input]
- textbox "请输入验证码" [SMS code input]  
- clickable element next to SMS field [get verification code button]
- button "登录" [login/submit button]
```

### Page JavaScript variables available before login:
```javascript
window.APPID = 'appswddscls8484'
window.USERID = ''                    // Empty until logged in
window.__anony_logon = '0'           // Login status flag
window.TAGNAME = '1.5.31'            // Frontend JS version
window.__aegis_id = "QV3PrsJEVJ1ZPgZoOv"  // Bot detection ID
window.GlobalState = { ... }         // App configuration object
```

### Store name (from page source):
```javascript
window.SHOP_NAME = 'N维咨询小店'
```

### CDN domains for static assets:
```javascript
window.__cdn_retry_domains = {
  "cdn": [
    "https://assets.cdn.xiaoeknow.com",
    "https://assets-tx.xet.tech", 
    "https://assets-tx.cdn.xiaoeknow.com"
  ],
  "micro_comp": [
    "https://static-resource-cos-1252524126.cdn.xiaoeknow.com",
    "https://static-resource-cos-tx.xet.tech",
    "https://assets-micro-tx.cdn.xiaoeknow.com"
  ]
}
```

### Bot detection notes:
- Uses Aegis RUM SDK for frontend monitoring
- Has white-screen detection that retries assets on CDN failover domains
- `window.__anony_logon` controls whether anonymous access is allowed (set to '0' = requires login)
