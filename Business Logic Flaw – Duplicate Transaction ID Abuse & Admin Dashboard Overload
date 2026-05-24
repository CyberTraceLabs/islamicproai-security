# Bug Report: Business Logic Flaw – Duplicate Transaction ID Abuse & Admin Dashboard Overload

**Project:** IslamicProAI – Premium Subscription System  
**Component:** Manual Payment Flow & Admin Dashboard  
**Type:** Business Logic / Data Integrity  
**OWASP:** A04:2021 – Insecure Design  
**Severity:** Medium (Financial & Operational Impact)  
**Status:** Fixed with Idempotency + Search  
**CWE:** CWE-840, CWE-696 (Incorrect Behavior Order)

---

## 1. Summary

Users upgrading to premium submitted a manual payment transaction ID (referred to internally as "TDX ID" – e.g., bKash/Nagad TRX). The original implementation accepted the same TDX ID multiple times with no deduplication check. This allowed:

1. A single payment to be claimed by multiple accounts
2. The same user to spam the same TDX ID repeatedly
3. Admin dashboard to fill with thousands of duplicate rows, with no search function – causing operational "jam"

> **Safe disclosure:** Exact payment provider names, database column names, and validation regex patterns are redacted. This write-up describes the design flaw and the architectural fix only.

## 2. Where it existed

- **File:** `app.py` – premium request handler
- **Original lines:** ~1135–1142 (ManualPayment lookup)
- **Admin view:** `/admin` dashboard – rendered full `ManualPayment.query.all()` with no pagination or search
- **Database:** `manual_payments` table had no unique constraint on transaction identifier

## 3. Root Cause

**Before (simplified):**
```python
# User submits premium request
tdx_id = request.form.get('trx_id')

# System only checked if THIS email had an approved payment
existing = ManualPayment.query.filter_by(email=user.email, status='Approved').first()

# Then created a NEW record every time, even with same tdx_id
new_payment = ManualPayment(email=user.email, tdx_id=tdx_id, status='Pending')
db.session.add(new_payment)
```

Missing controls:
1. **No idempotency** – same `tdx_id` could be inserted 100 times
2. **No cross-user check** – User A and User B could both claim the same transaction
3. **No admin tooling** – dashboard loaded all records, no search, no filter by TDX

This is a classic business logic error: trusting client-supplied transaction IDs without server-side uniqueness.

## 4. Business Impact

- **Financial risk:** One real payment could unlock premium for multiple accounts
- **Operational jam:** Admin dashboard took 12+ seconds to load with 4,000+ duplicate rows; staff had to manually scroll to find a TDX
- **Support burden:** Users complained "my payment not approved" while admin couldn't locate their TDX quickly
- **Data integrity:** Reconciliation with payment provider became impossible due to duplicates

During internal testing, submitting the same test TDX ID 50 times created 50 pending records – confirming the flaw.

## 5. How I Discovered It

While reviewing premium conversion metrics, I noticed the `manual_payments` table had a 3:1 ratio of pending to approved records. Sampling showed identical TDX IDs across different emails and timestamps. Simultaneously, the admin team reported dashboard freezes.

## 6. Fix – Idempotency + Search-First Admin UX

I implemented a two-part fix: enforce one-time use of transaction IDs, and rebuild admin tooling.

### Part A: Server-side idempotency (no secrets disclosed)

1. **Database constraint** – Added unique index on normalized transaction ID (exact DDL redacted)
2. **Pre-insert check** – Before creating a payment request:
```python
# AFTER (abstract, safe)
normalized_tdx = normalize_tdx(user_input)  # trimming, case-folding – logic omitted

if payment_exists(normalized_tdx):
    return error_response("This transaction ID has already been submitted")
    
create_payment_request(user, normalized_tdx)
```

3. **Atomic operation** – Check and insert wrapped in single transaction to prevent race conditions

### Part B: Admin search engine

Replaced full table load with searchable, paginated view:

```python
# AFTER (abstract)
@app.route('/admin')
@require_roles('admin')
def admin_dashboard():
    search_query = request.args.get('q', '')
    # server-side filtering, never loads full table
    payments = query_payments(search=search_query, page=page, limit=50)
```

Features added (implementation details omitted):
- Search by TDX ID, email, or status
- Instant results with debounce (no page reload)
- Pagination (50 per page)
- One-click approve/reject from search results

### Changes summary
- **Database:** +1 unique constraint (migration)
- **Backend:** +~40 lines (idempotency check + search API)
- **Frontend:** Replaced static table with search component
- **Removed:** Old `ManualPayment.query.all()` pattern

## 7. Verification

**Before fix:**
- Submit same TDX 5 times → 5 records created
- Admin dashboard load time: 11.8s for 4,200 records

**After fix:**
- Submit same TDX 2nd time → immediate error: "Transaction already submitted" (HTTP 409)
- Different user tries same TDX → same error, with audit log entry
- Admin search "TDX123" → returns 1 record in 180ms
- Dashboard initial load: 420ms (paginated)

**Business metrics after 2 weeks:**
- Duplicate TDX submissions: 0
- Premium fraud attempts blocked: 37
- Admin time spent on payment verification: reduced by 78%

## 8. Lessons for Portfolio

1. **Idempotency is a business requirement, not a feature** – any external ID (payment, webhook, order) must be unique server-side
2. **Build admin tools for scale from day one** – loading full tables works for 100 rows, fails at 4,000
3. **Never trust client-supplied transaction identifiers** – always normalize and deduplicate
4. **Safe public write-ups** – I can demonstrate fixing a financial logic bug without revealing validation regex, provider APIs, or exact database schema

This fix protected revenue integrity and made operations sustainable – exactly the kind of business-logic thinking SaaS companies value.
