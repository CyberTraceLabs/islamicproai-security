# Bug Report: OTP Validation Bypass Leading to Account Takeover

**Project:** IslamicProAI  
**Component:** Password Reset / Email Change Flow  
**Type:** Broken Access Control  
**OWASP:** A01:2021  
**Severity:** Critical  
**Status:** Fixed  
**CWE:** CWE-640, CWE-284

---

## 1. Summary
The OTP verification endpoint validated the OTP code correctly but did not bind the OTP to the target account. An attacker could use a valid OTP generated for their own account, intercept the request with a proxy, and change the `email` or `user_id` parameter to reset another user's password.

> **Safe disclosure:** Specific parameter names and OTP length are omitted. No step-by-step exploit is provided.

## 2. Root Cause
Original flow:
1. User requests OTP for their email → OTP stored with `user_id = A`
2. Verification endpoint checked: `if otp == stored_otp` → valid
3. Then performed update using `request.target_email` without re-checking ownership

Missing binding between OTP and the account being modified.

## 3. Impact
Full account takeover of any user with known email address, using attacker's own OTP.

## 4. Fix
Implemented cryptographic binding:
```python
# AFTER (abstract)
otp_record = OTP.query.filter_by(
    code=provided_otp,
    user_id=current_user.id,  # or target_user_id from secure session
    purpose='password_reset',
    expires_at > now()
).first()

if not otp_record:
    reject

# perform update ONLY for otp_record.user_id
```

Changes:
- OTP now stored with `user_id`, `purpose`, `expires_at`, and single-use flag
- Verification requires all three to match
- Removed `target_email` from client request – server derives from OTP record

## 5. Verification
- Attempt to reuse own OTP for another account → rejected
- OTP single-use enforced, deleted after success

## 6. Lessons
Never trust client-supplied identifiers in sensitive flows. Bind OTPs to user ID and purpose server-side.
