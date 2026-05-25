# Bug Report: Broken Access Control – Cross-User Conversation Access (IDOR)

**Project:** IslamicProAI  
**Component:** Chat API – `/ask_stream` and conversation history  
**Type:** Broken Access Control  
**OWASP:** A01:2021  
**Severity:** High  
**Status:** Fixed  
**CWE:** CWE-639 (Authorization Bypass Through User-Controlled Key)

---

## 1. Summary

Users could access chat conversations belonging to other accounts by supplying a different `session_id` in the API request. The backend retrieved the conversation by primary key without verifying that the requesting user owned that session. This allowed both free and premium users to read and continue conversations that were not theirs.

> **Safe disclosure:** This write-up describes the vulnerability class and the architectural fix only. Specific request payloads, exact parameter names beyond `session_id`, and any bypass techniques have been intentionally omitted to prevent misuse against similar systems.

## 2. Where it existed

- **File:** `app.py`
- **Endpoint:** `POST /ask_stream` (AI chat streaming)
- **Related:** `GET /api/chat-history`, `DELETE /api/session`
- **Original code location:** approximately lines 538–610 (session retrieval block)
- **Affected models:** `ChatSession`, `ChatMessage`

## 3. Root Cause

The original implementation followed this pattern:

```python
# BEFORE (simplified and redacted)
session_id = request_data.get("session_id")
chat_session = ChatSession.query.get(session_id)  # direct lookup

if not chat_session:
    chat_session = create_new_session(current_user)
    
# continue processing...
```

Missing controls:
1. **No ownership check** – `chat_session.user_id` was never compared to `current_user.id`
2. **Predictable IDs** – early versions used incremental integers, later UUIDs but still unprotected
3. **Same code path for all tiers** – premium and free users shared the identical lookup, so tier checks did not help

This is a classic Insecure Direct Object Reference (IDOR).

## 4. Impact

- **Confidentiality breach:** User A could read User B's private religious questions and AI responses
- **Integrity breach:** User A could append messages to User B's history, polluting their context
- **Premium bypass risk:** A free user supplying a premium user's active session could inherit premium features tied to that session (rate limits, model access)
- **Privacy violation:** Violates GDPR/CCPA expectations for data isolation

No evidence of mass exploitation was found in logs, but the flaw was reproducible in testing with two test accounts.

## 5. How I Discovered It

During a security review for my portfolio, I created two test users (User-A and User-B). While inspecting network traffic, I noticed the frontend sent `session_id` in plain JSON. I manually changed the value to a session belonging to the other test account (using my own data only). The API returned the other user's history instead of rejecting the request – confirming missing authorization.

## 6. Fix – Authorization Enforcement

I implemented a strict ownership check at the data-access layer, applied to every session-related endpoint.

### Key changes

1. **Centralized verification function** (added, not shown in detail)
   - Accepts `session_id` and `current_user`
   - Returns session only if `session.user_id == current_user.id`
   - Otherwise returns `None` and logs the attempt

2. **Updated retrieval logic**
```python
# AFTER (abstract, safe version)
session_id = request_data.get("session_id")
chat_session = get_owned_session(session_id, current_user)

if not chat_session:
    # either invalid ID or not owned – create fresh session
    chat_session = create_new_session(current_user)
    # do NOT fall back to requested ID
```

3. **Enforced at all entry points**
   - `/ask_stream` – added ownership check before loading history
   - `/api/chat-history` – now filters by `user_id` in the SQL query, not just by ID
   - `/api/session DELETE` – returns 403 if not owner

4. **Defense in depth**
   - Changed session IDs to UUIDv4 (non-sequential)
   - Added database index on `(id, user_id)` for fast ownership lookup
   - Added audit log for failed ownership attempts (rate-limited to prevent log flooding)

### Lines changed (approximate)
- `app.py`: +24 lines (new helper + checks), -9 lines (removed unsafe direct get)
- `models.py`: added composite index (no logic change)

## 7. Verification

**Before fix (test environment):**
- User-A with session `S1`, User-B with session `S2`
- Request with User-A credentials + `session_id=S2` → returned User-B data (FAIL)

**After fix:**
- Same request → API creates new session for User-A, returns 200 with new empty history
- Response no longer contains User-B data
- Server logs show: `WARN: ownership_check_failed user_id=A session_id=S2`
- Automated test suite now includes 12 IDOR test cases (all pass)

## 8. Lessons Learned

1. **Always check ownership, not just authentication** – `@login_required` is not enough; you need `@owns_resource`
2. **Never trust client-supplied IDs** – treat every ID as untrusted input
3. **Apply checks at the data layer** – filtering in SQL (`WHERE user_id = :current`) is safer than in-memory checks
4. **Use unpredictable IDs** – UUIDs don't fix IDOR, but they reduce enumeration risk
5. **Portfolio-safe disclosure** – I can demonstrate I fixed a critical IDOR without publishing an exploit recipe

This fix ensures strict tenant isolation between free and premium users, and makes the chat system safe to demonstrate publicly in my portfolio.
