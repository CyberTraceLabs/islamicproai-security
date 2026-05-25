# Bug Report: Critical Business Logic – Client-Side Price Manipulation

**Project:** IslamicProAI – E-commerce Shop  
**Component:** Order API (`/api/place-order`)  
**Type:** Business Logic Flaw  
**OWASP:** A04:2021  
**Severity:** Critical  
**Status:** Fixed  
**CWE:** CWE-840

---

## 1. Summary
The `/api/place-order` endpoint trusted the `total` and `items[].price` values sent by the client. An attacker could intercept the request and change a 1000 BDT product to 10 BDT, and the server would create the order at the manipulated price.

> **Safe disclosure:** Exact JSON structure is generalized. No working payload is included.

## 2. Root Cause
Original code:
```python
# BEFORE (abstract)
order_total = request.json['total']
items = request.json['items']
create_order(user, items, order_total)  # trusts client
```

No server-side price lookup or recalculation.

## 3. Impact
- Direct revenue loss
- Inventory sold below cost
- Financial reconciliation failure

## 4. Fix
Implemented server-side authoritative pricing:
```python
# AFTER (abstract)
server_total = 0
validated_items = []

for item in request.items:
    product = Product.query.get(item['id'])
    if not product or not product.is_active:
        abort(400)
    
    # ignore client price, use DB price
    line_total = product.price * item['qty']
    server_total += line_total
    validated_items.append({...})

# Optional: compare with client total for tampering detection
if abs(server_total - client_total) > 0.01:
    log_tampering_attempt(user.id)

create_order(user, validated_items, server_total)
```

Additional controls:
- Prices fetched with row-level locking during checkout
- HMAC signature on cart (optional defense-in-depth)
- Audit log for price mismatches

## 5. Verification
- Attempt to send manipulated price → order created at correct DB price
- Tampering attempts logged and alerted

## 6. Lessons
Never trust client-supplied monetary values. Always recalculate totals server-side from trusted data.
