# Bug Report #1: Missing Rate Limiting (DoS & Brute-Force Vulnerability)

**Project:** IslamicProAI – Flask AI Chat + E-commerce Backend  
**Component:** `app.py` – Authentication & AI API routes  
**Severity:** High (Security)  
**Status:** ✅ Fixed  by cybertrace labs |order by Toushin
**Date Fixed:** May 2024

---

## 1. What was the bug?

The application had **no rate limiting** on any HTTP endpoint. Attackers (or even buggy clients) could send unlimited requests to:

- `/login` – brute-force passwords
- `/ask_stream` – expensive AI API calls (Gemini/Mistral/DeepSeek)
- `/signup` – spam account creation

This is a classic OWASP API4:2023 – Unrestricted Resource Consumption vulnerability.

## 2. Where was it?

**File:** `app.py`  
**Before fix:** Lines 1-50 had no reference to Flask-Limiter at all.

The entire app initialization looked like this:

```python
# BEFORE (lines 1-35)
from flask import Flask, render_template, request, jsonify, ...
from flask_login import LoginManager, ...
import bcrypt, requests, os, uuid, random, secrets, json, re
from flask_migrate import Migrate
from dotenv import load_dotenv
load_dotenv()

app = Flask(__name__)
app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', 'islamic_pro_ai_vip_key_2025')
# ... database config ...
# NO RATE LIMITER INITIALIZED
```

Critical routes were completely open:

```python
# BEFORE (line ~1170)
@app.route('/login', methods=['GET', 'POST'])
def login():  # <-- no protection
    ...

# BEFORE (line ~535)
@app.route('/ask_stream', methods=['POST'])
@login_required
def ask_stream():  # <-- expensive AI call, no limit
    ...
```

## 3. Impact

- **Cost blow-up:** One user could trigger 1000+ AI calls/hour → $50+ API bill
- **Brute-force:** Unlimited login attempts → credential stuffing risk
- **DoS:** Simple `while true; do curl` could crash the Postgres pool
- **Poor UX:** Legitimate users suffered when server was overloaded

During testing I was able to send 342 requests to `/login` in 60 seconds with no blocking.

## 4. How I fixed it

I integrated **Flask-Limiter** with a tiered strategy:

### Step 1 – Add imports (exact lines added)
```python
# app.py:15
from flask_limiter import Limiter

# app.py:18
from flask_limiter.util import get_remote_address
```

### Step 2 – Initialize limiter (lines 36-43 added)
```python
# ================================================================
# RATE LIMITING
# ================================================================
limiter = Limiter(
    get_remote_address,          # identify client by IP
    app=app,
    default_limits=["200 per hour"],  # global safety net
    storage_uri="memory://"      # TODO: switch to Redis in production
)
```

### Step 3 – Protect high-risk endpoints

**a) Login – brute force protection**
```python
# app.py:1170 (ADDED decorator)
@app.route('/login', methods=['GET', 'POST'])
@limiter.limit("10 per minute")   # <-- NEW
def login():
```

**b) AI streaming – cost protection**
```python
# app.py:537 (ADDED decorator)
@app.route('/ask_stream', methods=['POST'])
@login_required
@limiter.limit("30 per hour")    # <-- NEW
def ask_stream():
```

## 5. Exact code change summary

| Action |                    Location |           Lines Changed |
|----------------|-------------------------|----------------------------|
| Add import |            app.py:15 |                   +1 |
| Add import |            app.py:18 |                   +1 |
| Create limiter |        app.py:36-43 |                +8 |
| Decorate login |        app.py:1170 |                 +1 |
| Decorate ask_stream |   app.py:537 |                  +1 |

| **Total 12 lines** |

Git diff preview:
```diff
+ from flask_limiter import Limiter
+ from flask_limiter.util import get_remote_address
+
+ limiter = Limiter(
+     get_remote_address,
+     app=app,
+     default_limits=["200 per hour"],
+     storage_uri="memory://"
+ )
...
 @app.route('/login', methods=['GET', 'POST'])
+@limiter.limit("10 per minute")
 def login():
...
 @app.route('/ask_stream', methods=['POST'])
 @login_required
+@limiter.limit("30 per hour")
 def ask_stream():
```

## 6. Verification

After fix:
```bash
for i in {1..15}; do curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/login; done
# Output: 200 200 ... 200 (10 times) then 429 429 429 429 429
```

Response headers now include:
```
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 0
Retry-After: 53
```

AI endpoint test: 31st request in same hour returns:
```json
{"error": "ratelimit exceeded 30 per 1 hour"}
```

## 7. What I learned

- Always set a **global default limit** first, then tighten per-route
- `memory://` is fine for dev, but production needs Redis to share state across workers
- Rate limiting is not just security – it's **cost control** for AI apps
- Document limits in API docs so frontend can show proper warnings

---

**Next improvement:** Migrate `storage_uri` to Redis and add user-based limits (not just IP) using `key_func=lambda: current_user.id` for authenticated routes.

