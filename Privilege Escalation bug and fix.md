# Bug Report: Privilege Escalation via Insecure Admin Bootstrap

**Project:** IslamicProAI  
**Component:** Authentication – `app.py` login flow  
**Type:** Privilege Escalation  
**Severity:** Critical  
**Status:** Fixed and Removed  
**fix:** By Cyber trace labs |  Manage and Control  by  Hussain Ahmed (Toushin)
**CVE-style:** CWE-798 (Use of Hard-coded Credentials), CWE-269 (Improper Privilege Management)

---

## 1. Summary

A development-time admin bootstrap routine was left inside the production `/login` route. The code performed a direct credential comparison and would auto-create a super-admin account on first use. Because the check lived in the login handler instead of a one-time setup script, it created an unintended privilege escalation path.

> **Responsible disclosure note:** Specific credential values, exact comparison logic, and email addresses have been redacted from this public write-up to prevent misuse. The fix is described at an architectural level only.

## 2. Location

- **File:** `app.py`
- **Function:** `login()`
- **Original lines:** ~1176–1193
- **Related code:** Admin auto-creation also existed in `__main__` startup block (lines ~3450–3462) – this is the correct place

## 3. Root Cause

1. **Hard-coded bootstrap in request path** – Instead of running once at deployment, the admin creation logic executed on every login attempt
2. **Mixed authentication flows** – Normal password verification was bypassed for a specific identity check
3. **Role assignment by email comparison** – Privilege was granted based on a string match rather than a secure database flag

This violates the principle of least privilege and creates two separate ways to become admin.

## 4. Impact (if exploited)

- An attacker who discovered the bootstrap condition could obtain `super_admin` role without database access
- Bypasses normal password hashing verification path
- Makes security audits fail (hard-coded secrets in source)
- If repository became public, the condition would be trivially discoverable

**No evidence of exploitation in production logs.**

## 5. How I Found It

During a code review for my portfolio, I noticed the `login()` function was over 40 lines long and contained a conditional block before the standard `check_password_hash()` call. Static analysis flagged "hard-coded credential" pattern.

## 6. Fix – What Changed

I completely removed the insecure bootstrap from the request path and consolidated admin creation into a single, secure startup routine.

### Before (redacted)
```python
@app.route('/login', methods=['POST'])
def login():
    email = request.form.get('email')
    password = request.form.get('password')
    
    # INSECURE: special-case admin creation inside login
    if email == ADMIN_IDENTIFIER and password == [REDACTED]:
        user = get_or_create_super_admin()  # bypasses normal auth
        login_user(user)
        return redirect(...)
    
    # normal flow continues...
```

### After
```python
@app.route('/login', methods=['POST'])
@limiter.limit("10 per minute")
def login():
    email = request.form.get('email')
    password = request.form.get('password')
    
    # Single, standard authentication path for ALL users
    user = User.query.filter_by(email=email).first()
    if user and verify_password(user.password, password):
        login_user(user)
        return redirect(...)
    
    flash('Invalid credentials', 'danger')
```

### Additional hardening
1. **Removed** lines 1176–1193 entirely (-18 lines)
2. **Kept** only the safe startup bootstrap in `if __name__ == '__main__':` which now reads credentials from environment variables:
   ```python
   ADMIN_EMAIL = os.environ.get('ADMIN_EMAIL')
   ADMIN_INIT_PASSWORD = os.environ.get('ADMIN_INIT_PASSWORD')
   # hashed with bcrypt before storage
   ```
3. **Enforced RBAC** – All privilege checks now use `user.role` from database only, no email comparisons in decorators
4. **Added** `.env.example` with placeholder values, and added real `.env` to `.gitignore`

## 7. Verification

- Attempting the old bootstrap condition now follows the normal password check and fails with "Invalid credentials"
- `git grep -n "HARDCODED\|Toushin469"` returns zero results
- Bandit security scan: previously flagged `B105:hardcoded_password_string`, now clean
- All admin routes still work when logging in with properly created super-admin account from startup script

## 8. Lessons for Portfolio

- **Never put setup code in request handlers** – initialization belongs in migrations or CLI commands
- **Single authentication path** – every user, including admins, must go through the same `verify_password()` function
- **Secrets belong in environment, not code** – even for portfolio projects, use `os.getenv()`
- **Principle of least privilege** – removed email-based privilege and rely solely on database role field with hierarchy check

This fix reduced the attack surface and made the codebase safe to open-source for my portfolio without exposing any escalation technique.
