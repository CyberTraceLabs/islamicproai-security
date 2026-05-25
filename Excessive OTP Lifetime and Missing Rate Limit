# Bug Report: Excessive OTP Lifetime and Missing Rate Limit

**Project:** IslamicProAI  
**Component:** OTP System  
**Type:** Security Misconfiguration  
**OWASP:** A07:2021  
**Severity:** Medium  
**Status:** Fixed  
**CWE:** CWE-307, CWE-613

---

## 1. Summary
OTP codes were valid for an excessively long period (originally >60 minutes) and the verification endpoint had no rate limiting, allowing brute-force guessing attacks.

## 2. Root Cause
- `expires_at` set far in future
- No counter for failed attempts
- No throttling on `/verify-otp`

## 3. Impact
Attacker with automated tooling could brute-force 6-digit OTP within validity window.

## 4. Fix
1. Reduced OTP lifetime to **10 minutes**
```python
expires_at = now() + timedelta(minutes=10)
```
2. Added rate limiting:
```python
@limiter.limit("5 per minute")
@limiter.limit("20 per hour")
def verify_otp(): ...
```
3. Implemented progressive backoff and account lockout after 5 failures
4. OTP marked as consumed after first successful use

## 5. Verification
- OTP expires after 10 minutes (tested)
- 6th attempt within minute → HTTP 429
- Brute-force simulation blocked

## 6. Lessons
Short-lived, single-use OTPs with strict rate limits are essential for any auth flow.
