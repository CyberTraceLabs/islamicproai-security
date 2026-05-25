# Bug Report: Broken Access Control – Chat Session Hijacking (IDOR)

**Project:** IslamicProAI  
**Component:** AI Chat API (`/ask_stream`)  
**Type:** Insecure Direct Object Reference  
**OWASP:** A01:2021  
**Severity:** High  
**Status:** Fixed  
**CWE:** CWE-639

---

## 1. Summary
Authenticated users could supply any `session_id` in the chat API request and the server would load that conversation without verifying ownership. This allowed User A to read and write messages in User B's private chat history.

> **Disclosure note:** Exact request parameters and session ID formats are described at a high level only. No working exploit payload is provided.

## 2. Root Cause
The original code retrieved the session by primary key only:
```python
# BEFORE (abstract)
session = ChatSession.query.get(request.session_id)
# no check: session.user_id == current_user.id
```

Missing authorization layer after authentication.

## 3. Impact
- Confidentiality: private religious questions exposed
- Integrity: attacker could pollute victim's chat context
- Premium bypass: free user could continue a premium user's session

## 4. Fix
Implemented strict ownership enforcement:
```python
# AFTER (abstract)
session = ChatSession.query.filter_by(
    id=session_id, 
    user_id=current_user.id
).first()

if not session:
    abort(403)  # or create new owned session
```

Additional hardening:
- All session lookups now include `user_id` in WHERE clause
- Added audit logging for ownership failures
- Session IDs remain UUIDv4 (non-sequential)

## 5. Verification
- Test with two accounts: User-A supplying User-B's session → now receives 403
- Automated tests cover 15 IDOR scenarios

## 6. Lessons
Always enforce authorization at data layer, not just authentication. Never trust client-supplied resource IDs.
