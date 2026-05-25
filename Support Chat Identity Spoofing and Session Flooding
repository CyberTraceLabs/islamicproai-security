# Bug Report: Support Chat Identity Spoofing and Session Flooding

**Project:** IslamicProAI – Help Desk  
**Component:** Support messaging system  
**Type:** Business Logic / Abuse  
**OWASP:** A04:2021  
**Severity:** Medium  
**Status:** Fixed  
**CWE:** CWE-284

---

## 1. Summary
The support chat system identified users by mutable display name instead of immutable user ID. Changing the display name created a new support session each time, allowing a single user to flood the admin inbox with unlimited sessions. Additionally, there was no rate limit on messages.

> **Safe disclosure:** Implementation details of ID generation are omitted.

## 2. Root Cause
1. Session keyed by `username` (changeable) not `user_id`
2. No rate limiting on `/support/message`
3. No deduplication of active sessions

## 3. Impact
- Admin dashboard spam
- Loss of conversation continuity
- Potential impersonation if names collided

## 4. Fix
1. **Immutable identifier:** Introduced permanent `account_uuid` generated at signup, never changes even if email/name changes. All support messages now link to this ID.
```python
# abstract
message = SupportMessage(
    account_id=current_user.permanent_id,  # immutable
    content=text
)
```

2. **Session deduplication:** One active support thread per `account_id`. Name changes update display only, do not create new thread.

3. **Rate limiting:**
```python
@limiter.limit("50 per hour", key_func=lambda: current_user.permanent_id)
def send_support_message(): ...
```

4. **Admin view:** Now groups by permanent ID, shows full history regardless of name/email changes.

## 5. Verification
- Change username 5 times → same support thread continues
- 51st message in hour → HTTP 429
- Admin can track user across email changes via permanent ID

## 6. Lessons
Use immutable internal IDs for all security and support functions. Never key sensitive flows by mutable user-controlled fields, and always rate-limit abuse-prone endpoints.
