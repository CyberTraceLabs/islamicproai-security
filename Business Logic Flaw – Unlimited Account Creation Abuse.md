# Bug Report: Business Logic Flaw – Unlimited Account Creation Abuse

**Project:** IslamicProAI  
**Component:** User Registration & Anti-Abuse  
**Type:** Business Logic  
**OWASP:** A04:2021 – Insecure Design  
**Severity:** Medium-High (Business Impact)  
**Status:** Fixed with Device Fingerprinting  
**CWE:** CWE-840 (Business Logic Errors)
**Find and FIX:** By Cybertrace labs owned by Hussain Ahmed (Toushin)

---

## 1. Summary

The original signup flow had **no restriction on the number of accounts a single person could create**. A user could register unlimited free accounts from the same device to abuse free credits, bypass rate limits, manipulate referral rewards, and evade bans.

This is not a code crash – it's a business logic flaw. The system trusted email uniqueness alone, which is trivial to bypass with disposable emails.

> **Safe disclosure note:** This document describes the vulnerability class and the defensive design only. Specific fingerprinting algorithms, hash salts, storage schema details, and bypass detection heuristics are intentionally omitted to protect the live system.

## 2. Where it existed

- **File:** `app.py` – `signup()` route
- **Original lines:** ~1120–1160
- **Database:** `User` table only checked `email` uniqueness
- **Missing:** Any device, IP, or browser-level correlation

## 3. Root Cause

1. **Identity = Email only** – The business assumed one email = one person
2. **No velocity checks** – No limit on signups per IP/hour or per device/day
3. **Free tier abuse vector** – Each new account received fresh daily AI credits and free trial premium
4. **No correlation layer** – No way to link multiple accounts to the same physical device

In testing, I was able to create 27 accounts in 15 minutes using temporary email services from the same laptop – all accounts worked normally.

## 4. Business Impact

- **Revenue loss:** Users cycled free trials instead of paying for premium
- **Resource abuse:** Each account consumed AI API credits (direct cost)
- **Rate limit bypass:** After hitting 30 requests/hour, attacker switched accounts
- **Spam & reputation risk:** Fake accounts used for promotional abuse in community features
- **Support overhead:** Manual cleanup of duplicate accounts

Estimated potential loss: ~$120/month in API costs during early beta if exploited at scale.

## 5. How I Discovered It

While building my portfolio analytics dashboard, I noticed the `User` table had 5 accounts with identical `created_at` timestamps within 2 minutes, different emails, same IP in logs. This pattern indicated the same person testing the system – confirming the missing control.

## 6. Fix – Device Fingerprinting with Account Cap

I implemented a privacy-preserving device fingerprint system with a hard business rule: **maximum 3 accounts per device lifetime**.

### Design principles (no sensitive details disclosed)
- **No PII stored** – fingerprint is a one-way hash, not raw device data
- **Multi-signal** – combines network and browser characteristics (exact signals redacted)
- **Server-side enforcement** – check happens before user record is created
- **Graceful failure** – user sees generic "limit reached" message, not technical details

### Before (abstract)
```python
@app.route('/signup')
def signup():
    email = request.form['email']
    if User.query.filter_by(email=email).first():
        return "email exists"
    
    # create user immediately – no device check
    create_user(email, password)
```

### After (abstract, safe version)
```python
MAX_ACCOUNTS_PER_DEVICE = 3  # business rule

@app.route('/signup')
def signup():
    email = request.form['email']
    
    # 1. Generate privacy-safe device identifier (details omitted)
    device_id = generate_anonymous_fingerprint(request)
    
    # 2. Enforce business rule
    existing_count = count_accounts_for_device(device_id)
    if existing_count >= MAX_ACCOUNTS_PER_DEVICE:
        log_abuse_attempt(device_id)
        return render_template('limit_reached.html'), 429
    
    # 3. Normal creation
    user = create_user(email, password)
    link_device_to_user(device_id, user.id)
```

### Additional hardening (architecture only)
1. **Database table** `device_fingerprints` stores only hashed identifiers + user_id + timestamp (no raw data)
2. **Rate limiting** on signup endpoint (already implemented via Flask-Limiter)
3. **Monitoring** – dashboard alerts when >10 devices hit the cap in 1 hour (possible attack)
4. **Privacy compliance** – fingerprint expires after 12 months of inactivity, documented in privacy policy

### Lines changed (approximate, no secrets)
- `app.py`: +35 lines (fingerprint generation wrapper + check)
- `models.py`: +1 new model (schema redacted)
- Migration added composite index for performance

## 7. Verification

**Before fix:**
- Test script created 10 accounts from same browser → all succeeded

**After fix:**
- Accounts 1-3: success
- Account 4: blocked with HTTP 429 and friendly message: "You've reached the maximum number of accounts for this device"
- Attempting with different email but same device → still blocked
- Using different device → allowed (proves device-level, not IP-only)

**Business metrics after 30 days:**
- Duplicate account creation dropped 94%
- No legitimate user complaints (3-account limit matches real family-use case)
- API costs stabilized

## 8. Lessons for Portfolio

1. **Business logic bugs cost money, not just data** – this flaw had direct $ impact
2. **Email ≠ identity** – always add a second, privacy-safe correlation factor for abuse prevention
3. **Design limits early** – it's cheaper to add a cap at MVP than to clean up abuse later
4. **Balance security and privacy** – used hashing and expiration to avoid storing PII
5. **Don't disclose the fingerprint recipe** – in public write-ups, describe *that* you fingerprint, not *how* exactly, to prevent evasion

This fix demonstrates ability to think beyond code vulnerabilities to protect business model integrity – a key skill for production SaaS products.
