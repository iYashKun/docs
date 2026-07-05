# Captcha API Documentation

**API URL:** `https://usecaptcha.vercel.app`

---

## 📋 Table of Contents
1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Endpoints](#endpoints)
4. [Response Codes](#response-codes)
5. [Examples](#examples)
6. [Security Features](#security-features)
7. [Troubleshooting](#troubleshooting)

---

## Overview

A secure, production-ready CAPTCHA API built with Ruby/Rack, featuring:
- JWT-based token authentication
- IP-locked verification
- Attempt limiting (3 attempts per CAPTCHA)
- Secure HttpOnly cookies
- AWS Lambda compatible
- Full security headers (CSP, HSTS, etc.)

**Workflow:**
```
1. GET /captcha.rb (GENERATE) → Get tokens and image
2. GET /captcha.rb?verify=TOKEN&captcha=ANSWER (VERIFY) → Validate answer
```

---

## Authentication

### Token Format
- **Type:** JWT (JSON Web Token)
- **Algorithm:** HS256
- **Secret:** `JWT_SECRET` environment variable
- **Payload Example:**
```json
{
  "captcha": "ABC12",
  "attempts_left": 3,
  "ip": "192.168.1.1",
  "iat": 1688000000,
  "exp": 1688000300
}
```

### Security Features
- ✅ **IP Lock:** Token bound to requesting IP
- ✅ **Time Expiry:** 5 minutes validity
- ✅ **Attempt Limiting:** Max 3 verification attempts
- ✅ **Secure Cookies:** HttpOnly + Secure + SameSite=Strict
- ✅ **No Token Reuse:** Invalid after first wrong answer in AWS Lambda

---

## Endpoints

### 1. **Generate CAPTCHA** 
📝 Create a new CAPTCHA challenge

**Request:**
```http
GET /captcha.rb HTTP/1.1
Host: your-domain.com
```

**Response:** `200 OK`
```json
{
  "success": true,
  "csrf_token": "eyJhbGciOiJIUzI1NiJ9.eyJjYXB0Y2hhIjoiQUJDMTIiLCJhdHRlbXB0c19sZWZ0IjozLCJpcCI6IjE5Mi4xNjguMS4xIiwiaWF0IjoxNjg4MDAwMDAwLCJleHAiOjE2ODgwMDAzMDB9.signature",
  "captcha_id": "eyJhbGciOiJIUzI1NiJ9.eyJjYXB0Y2hhIjoiQUJDMTIiLCJpcCI6IjE5Mi4xNjguMS4xIiwiZXhwIjoxNjg4MDAwMzAwfQ.signature",
  "client_ip": "192.168.1.1",
  "total_attempts": 3,
  "generated_at_ist": "2026-07-04 15:30:00 IST",
  "expires_at_ist": "2026-07-04 15:35:00 IST",
  "png_url": "/captcha.rb?png=eyJhbGciOiJIUzI1NiJ9...",
  "data": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAKoAAAA3...",
  "timestamp": 1688000000,
  "error_code": 200
}
```

**Cookies Set:**
```
Set-Cookie: csrf_token=...; Path=/; Max-Age=300; HttpOnly; Secure; SameSite=Strict
Set-Cookie: captcha_id=...; Path=/; Max-Age=300; HttpOnly; Secure; SameSite=Strict
```

**Use Cases:**
- Page load → Show CAPTCHA form
- User clicks "Refresh CAPTCHA" → Generate new one

---

### 2. **View CAPTCHA Image**
🖼️ Retrieve the PNG image (with fallback support)

**Request:**
```http
GET /captcha.rb?png=eyJhbGciOiJIUzI1NiJ9.eyJjYXB0Y2hhIjoiQUJDMTIiLCJpcCI6IjE5Mi4xNjguMS4xIiwiZXhwIjoxNjg4MDAwMzAwfQ.signature HTTP/1.1
Host: your-domain.com
```

**Response:** `200 OK`
```json
{
  "success": true,
  "image": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAKoAAAA3CAYAAACNrR...",
  "type": "image/png",
  "size": 6745,
  "timestamp": 1688000010,
  "error_code": 200
}
```

**Error - IP Mismatch:** `403 Forbidden`
```json
{
  "success": false,
  "message": "IP Mismatch! Image view forbidden.",
  "error_code": 403
}
```

**Error - Expired:** `400 Bad Request`
```json
{
  "success": false,
  "message": "Captcha Expired! Generate new one.",
  "error_code": 400
}
```

**HTML Usage (Fallback):**
```html
<!-- Primary: Inline display -->
<img id="captcha" src="data:image/png;base64,..." alt="CAPTCHA">

<!-- Fallback: If data URL fails -->
<img src="/captcha.rb?png=TOKEN" alt="CAPTCHA" onerror="handleImageError()">
```

---

### 3. **Verify CAPTCHA Answer**
✅ Submit and verify the user's answer

**Request:**
```http
GET /captcha.rb?verify=eyJhbGciOiJIUzI1NiJ9.eyJjYXB0Y2hhIjoiQUJDMTIiLCJhdHRlbXB0c19sZWZ0IjozLCJpcCI6IjE5Mi4xNjguMS4xIiwiaWF0IjoxNjg4MDAwMDAwLCJleHAiOjE2ODgwMDAzMDB9.signature&captcha=ABC12 HTTP/1.1
Host: your-domain.com
```

**Parameters:**
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `verify` | string | ✅ | CSRF token from generation endpoint |
| `captcha` | string | ✅ | User's answer (case-insensitive, trimmed) |

**Response - Success:** `200 OK`
```json
{
  "success": true,
  "message": "Verification Successful! ✅",
  "error_code": 200,
  "attempts_used": 1
}
```

**Response - Wrong Answer (Attempts Left):** `400 Bad Request`
```json
{
  "success": false,
  "message": "Invalid Captcha! Try again.",
  "error_code": 400,
  "attempts_left": 2,
  "next_verify_token": "eyJhbGciOiJIUzI1NiJ9.eyJjYXB0Y2hhIjoiQUJDMTIiLCJhdHRlbXB0c19sZWZ0IjoyLCJpcCI6IjE5Mi4xNjguMS4xIiwiaWF0IjoxNjg4MDAwMDA1LCJleHAiOjE2ODgwMDAzMDB9.signature"
}
```

**Response - Attempts Exhausted:** `400 Bad Request`
```json
{
  "success": false,
  "message": "Wrong Captcha. Attempts exhausted!",
  "error_code": 429,
  "attempts_left": 0
}
```

**Response - IP Mismatch:** `403 Forbidden`
```json
{
  "success": false,
  "message": "IP Mismatch! Security violation suspected.",
  "error_code": 403
}
```

**Response - Expired Token:** `400 Bad Request`
```json
{
  "success": false,
  "message": "Captcha expired! 5 minutes limit crossed.",
  "error_code": 400
}
```

---

## Response Codes

| Code | Status | Meaning |
|------|--------|---------|
| **200** | ✅ OK | Success (generation/verification) |
| **400** | ⚠️ Bad Request | Invalid token, expired, wrong answer, attempts exceeded |
| **403** | 🚫 Forbidden | IP mismatch, security violation |
| **404** | ❌ Not Found | Invalid route |
| **429** | 🔒 Too Many Requests | Max attempts exceeded (attempt limiting) |
| **500** | 💥 Server Error | Unexpected error |

---

## Examples

### Complete Workflow (JavaScript)

```javascript
// Step 1: Generate CAPTCHA
async function generateCaptcha() {
  const res = await fetch('/captcha.rb');
  const data = await res.json();
  
  if (!data.success) throw new Error(data.message);
  
  // Store tokens
  window.tokens = {
    csrf: data.csrf_token,
    captcha: data.captcha_id
  };
  
  // Display image
  document.getElementById('captcha-image').src = data.data;
  
  // Show form
  document.getElementById('captcha-form').style.display = 'block';
  
  console.log('CAPTCHA Generated! Expires:', data.expires_at_ist);
}

// Step 2: Submit answer
async function verifyCaptcha() {
  const userAnswer = document.getElementById('captcha-input').value;
  
  const res = await fetch(
    `/captcha.rb?verify=${window.tokens.csrf}&captcha=${userAnswer}`
  );
  const data = await res.json();
  
  if (data.success) {
    alert('✅ CAPTCHA Verified!');
    // Proceed with form submission
    submitMainForm();
  } else {
    if (data.attempts_left > 0) {
      alert(`❌ Wrong! ${data.attempts_left} attempts left.`);
      // Update token for next attempt
      window.tokens.csrf = data.next_verify_token;
      document.getElementById('captcha-input').value = '';
    } else {
      alert('❌ Attempts exhausted! Please refresh and try again.');
      generateCaptcha(); // Generate new captcha
    }
  }
}

// Step 3: Refresh CAPTCHA
function refreshCaptcha() {
  generateCaptcha();
}

// Initialize on page load
document.addEventListener('DOMContentLoaded', generateCaptcha);
```

### HTML Form

```html
<!DOCTYPE html>
<html>
<head>
  <title>CAPTCHA Test</title>
  <style>
    #captcha-image {
      border: 1px solid #ccc;
      padding: 10px;
      width: 200px;
      height: auto;
    }
    .form-group { margin: 15px 0; }
    input, button { padding: 8px; font-size: 14px; }
    button { cursor: pointer; }
  </style>
</head>
<body>
  <h1>🔐 CAPTCHA Verification</h1>
  
  <div id="captcha-form" style="display: none;">
    <div class="form-group">
      <label>Enter the text from image:</label><br>
      <img id="captcha-image" alt="CAPTCHA Image">
      <button type="button" onclick="refreshCaptcha()">🔄 Refresh</button>
    </div>
    
    <div class="form-group">
      <input 
        type="text" 
        id="captcha-input" 
        placeholder="Enter CAPTCHA..." 
        maxlength="5"
        autocomplete="off"
      >
    </div>
    
    <button type="button" onclick="verifyCaptcha()">✅ Verify</button>
  </div>

  <script src="captcha.js"></script>
</body>
</html>
```

### cURL Examples

**Generate:**
```bash
curl -X GET http://localhost:3000/captcha.rb \
  -H "User-Agent: Mozilla/5.0" \
  -v
```

**View Image:**
```bash
curl -X GET "http://localhost:3000/captcha.rb?png=eyJhbGc..." \
  -H "User-Agent: Mozilla/5.0"
```

**Verify:**
```bash
curl -X GET "http://localhost:3000/captcha.rb?verify=eyJhbGc...&captcha=ABC12" \
  -H "User-Agent: Mozilla/5.0"
```

---

## Security Features

### 🛡️ Headers Sent
```
X-Content-Type-Options: nosniff                    ← MIME sniffing protection
X-Frame-Options: SAMEORIGIN                       ← Clickjacking protection
X-XSS-Protection: 1; mode=block                   ← XSS protection
Strict-Transport-Security: max-age=31536000       ← HSTS (1 year)
Content-Security-Policy: default-src 'self'       ← CSP
Referrer-Policy: strict-origin-when-cross-origin  ← Referrer control
Permissions-Policy: geolocation=(), microphone=() ← Feature policy
Cache-Control: no-store, no-cache, max-age=0      ← No caching
```

### 🍪 Cookie Security
```
HttpOnly       → JavaScript cannot access (XSS protection)
Secure         → HTTPS only
SameSite=Strict → CSRF protection
Max-Age=300    → Expires in 5 minutes
```

### 🔒 IP Locking
- Token bound to **requesting IP address**
- Different IP = **403 Forbidden**
- Prevents token theft/reuse from other IPs

### ⏱️ Time Limits
- **5 minute expiry** from generation
- Token automatically invalid after expiry

### 🔢 Attempt Limiting
- **3 attempts maximum** per CAPTCHA
- Each wrong attempt decrements counter
- New token issued on failed attempt (for audit trail)
- After 3 failures → Must generate new CAPTCHA

---

## Troubleshooting

### Problem: "IP Mismatch! Image view forbidden."
**Cause:** Requesting from different IP than token generation
**Solution:** 
- Use same network/IP for all requests
- Check proxy/VPN settings
- Verify X-Forwarded-For header if behind load balancer

### Problem: "Captcha Expired!"
**Cause:** Token older than 5 minutes
**Solution:** Call generation endpoint again

### Problem: "source sequence is illegal/malformed utf-8"
**Cause:** AWS Lambda UTF-8 marshalling error (FIXED in v2.0)
**Solution:** Already handled - returns base64 in JSON

### Problem: Image not loading
**Cause:** Data URL not supported or image token invalid
**Solution:** 
- Use fallback PNG route: `/captcha.rb?png=TOKEN`
- Check browser console for errors
- Verify token hasn't expired

### Problem: CAPTCHA always wrong
**Cause:** Case sensitivity or special characters
**Solution:** 
- Answer is case-insensitive (auto-converted to uppercase)
- Only alphanumeric characters (0-9, A-Z, a-z)
- No special chars or 'j' letter (by design)

### Problem: Cookies not being set
**Cause:** HTTPS required for Secure flag in production
**Solution:**
- Locally: Works on `http://localhost:3000`
- Production: Must use HTTPS
- Or set `JWT_SECRET` env var

### Problem: CORS errors
**Cause:** Cross-origin request blocked
**Solution:** Headers already set:
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, OPTIONS, POST
Access-Control-Allow-Credentials: true
```
If issue persists, set `CORS_ORIGIN` env var:
```bash
export CORS_ORIGIN="https://your-frontend.com"
```

---

## Environment Variables

```bash
# JWT Secret (REQUIRED)
export JWT_SECRET="your_super_secret_key_min_32_chars"

# CORS Origin (Optional, defaults to *)
export CORS_ORIGIN="https://yourdomain.com"

# Port (Optional, defaults to 3000)
export PORT=3000
```

---

## API Status Endpoint (Optional Add)

Check API health:
```http
GET /health HTTP/1.1
```

Future enhancement: Could add health check for monitoring.

---

## Rate Limiting (Not Implemented Yet)

Consider adding:
- Redis-based rate limiting
- Max N CAPTCHA generations per IP/hour
- Max N verification attempts per IP/hour

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | 2026-07-04 | AWS Lambda UTF-8 fix, security hardening, cookies, headers |
| 1.0 | 2026-07-04 | Initial release |

---

## Support

For issues or questions:
1. Check [Troubleshooting](#troubleshooting) section
2. Review security headers in browser DevTools
3. Check server logs for JWT decode errors
4. Verify IP matches in token payload

---

**Last Updated:** 2026-07-04  
**API Version:** 2.0  
**Status:** ✅ Production Ready
