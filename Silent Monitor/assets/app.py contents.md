```python
cat app.py 
#!/usr/bin/env python3
import os
import re
import sqlite3
import subprocess
import hashlib
from functools import wraps
from flask import (
    Flask, request, render_template, redirect,
    url_for, session, g
)

app = Flask(__name__)
app.secret_key = os.environ.get("FLASK_SECRET")
if not app.secret_key:
    raise RuntimeError("FLASK_SECRET environment variable not set")

DATABASE = "/opt/netops/netops.db"


def get_db():
    db = getattr(g, "_database", None)
    if db is None:
        db = g._database = sqlite3.connect(DATABASE)
        db.row_factory = sqlite3.Row
    return db


@app.teardown_appcontext
def close_db(exc):
    db = getattr(g, "_database", None)
    if db is not None:
        db.close()


def init_db():
    db = sqlite3.connect(DATABASE)
    db.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id       INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE NOT NULL,
            password TEXT NOT NULL,
            role     TEXT NOT NULL DEFAULT 'viewer'
        )
    """)
    db.execute("""
        CREATE TABLE IF NOT EXISTS audit_log (
            id       INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT,
            action   TEXT,
            detail   TEXT,
            ts       TEXT
        )
    """)
    pw = hashlib.md5(b"C0rp$N3tS3cur3!").hexdigest()
    db.execute(
        "INSERT OR IGNORE INTO users (username, password, role) VALUES (?,?,?)",
        ("netops", pw, "operator")
    )
    db.commit()
    db.close()


def log_action(username, action, detail=""):
        pass


def login_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if not session.get("user"):
            return redirect(url_for("login"))
        return f(*args, **kwargs)
    return decorated


# need to modify this filter.
BLOCK_RE = re.compile(
    r"(union\s+select|drop\s+table|insert\s+into|xp_cmdshell)",
    re.IGNORECASE
)

FAKE_AUDIT = [
    ("jmartin",  "LOGIN_OK",     "",           "2026-01-14 08:03:11"),
    ("jmartin",  "HEALTH_CHECK", "10.0.1.4",   "2026-01-14 08:04:02"),
    ("jmartin",  "HEALTH_CHECK", "10.0.1.7",   "2026-01-14 08:04:28"),
    ("jmartin",  "LOGOUT",       "",           "2026-01-14 08:31:55"),
    ("netops",   "LOGIN_OK",     "",           "2026-01-14 09:17:40"),
    ("netops",   "HEALTH_CHECK", "10.0.0.1",   "2026-01-14 09:18:03"),
    ("netops",   "HEALTH_CHECK", "10.0.2.12",  "2026-01-14 09:19:47"),
    ("netops",   "LOGOUT",       "",           "2026-01-14 09:44:22"),
    ("svc-mon",  "LOGIN_OK",     "",           "2026-01-14 10:00:01"),
    ("svc-mon",  "HEALTH_CHECK", "10.0.1.1",   "2026-01-14 10:00:04"),
    ("svc-mon",  "HEALTH_CHECK", "10.0.1.2",   "2026-01-14 10:00:07"),
    ("svc-mon",  "HEALTH_CHECK", "10.0.1.3",   "2026-01-14 10:00:10"),
    ("svc-mon",  "HEALTH_CHECK", "10.0.1.4",   "2026-01-14 10:00:13"),
    ("svc-mon",  "LOGOUT",       "",           "2026-01-14 10:00:16"),
    ("jmartin",  "LOGIN_FAIL",   "",           "2026-01-14 11:02:38"),
    ("jmartin",  "LOGIN_OK",     "",           "2026-01-14 11:02:51"),
    ("jmartin",  "HEALTH_CHECK", "10.0.2.5",   "2026-01-14 11:03:14"),
    ("jmartin",  "LOGOUT",       "",           "2026-01-14 11:28:07"),
    ("netops",   "LOGIN_OK",     "",           "2026-01-15 07:55:30"),
    ("netops",   "HEALTH_CHECK", "10.0.0.254", "2026-01-15 07:56:01"),
]


@app.route("/")
def index():
    return render_template("index.html")


@app.route("/internal", methods=["GET", "POST"])
def login():
    error = None

    if request.method == "POST":
        username = request.form.get("username", "")
        password = request.form.get("password", "")

        if BLOCK_RE.search(username) or BLOCK_RE.search(password):
            log_action(username, "LOGIN_BLOCKED")
            error = "Invalid username or password."
        else:
            pw_hash = hashlib.md5(password.encode()).hexdigest()
            # Vulnerable: direct string interpolation — no parameterisation
            query = (
                "SELECT id, username, role FROM users "
                "WHERE username = '%s' AND password = '%s'"
            ) % (username, pw_hash)

            try:
                db = get_db()
                row = db.execute(query).fetchone()
                if row:
                    session["user"] = row["username"]
                    session["role"] = row["role"]
                    log_action(row["username"], "LOGIN_OK")
                    return redirect(url_for("dashboard"))
                else:
                    error = "Invalid username or password."
                    log_action(username, "LOGIN_FAIL")
            except Exception:
                error = "Invalid username or password."
                log_action(username, "LOGIN_ERROR")

    return render_template("login.html", error=error)


@app.route("/internal/dashboard")
@login_required
def dashboard():
    db = get_db()
    real_logs = db.execute(
        "SELECT username, action, detail, ts FROM audit_log "
        "ORDER BY ts DESC LIMIT 8"
    ).fetchall()

    combined = [(r["username"], r["action"], r["detail"], r["ts"])
                for r in real_logs]
    combined += list(FAKE_AUDIT)

    return render_template(
        "dashboard.html",
        current_user=session["user"],
        role=session["role"],
        logs=combined
    )


# vulnerable to newline injection (\n / %0a), fix soon.
PING_BLOCK_RE = re.compile(r"[;|`$&]")


@app.route("/internal/health", methods=["GET", "POST"])
@login_required
def health_check():
    output  = None
    cmd_err = None
    target  = ""

    if request.method == "POST":
        target = request.form.get("target", "").strip()

        if not target:
            cmd_err = "No target specified."
        elif len(target) > 100:
            cmd_err = "Invalid hostname or IP address."
        elif PING_BLOCK_RE.search(target):
            cmd_err = "Invalid hostname or IP address."
        else:
            try:
                proc = subprocess.Popen(
                    "ping -c 2 -W 1 " + target,
                    shell=True,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.STDOUT,
                    text=True
                )
                try:
                    output, _ = proc.communicate(timeout=8)
                except subprocess.TimeoutExpired:
                    proc.stdout.close()
                    output = "[process running in background]"

                log_action(session["user"], "HEALTH_CHECK",
                           target.split("\n")[0])
            except Exception:
                cmd_err = "An error occurred."

    return render_template(
        "health.html",
        current_user=session["user"],
        role=session["role"],
        output=output,
        error=cmd_err,
        target=target
    )


@app.route("/internal/logout")
def logout():
    user = session.get("user", "unknown")
    session.clear()
    log_action(user, "LOGOUT")
    return redirect(url_for("login"))


@app.errorhandler(404)
def not_found(e):
    return render_template("404.html"), 404


@app.errorhandler(403)
def forbidden(e):
    return render_template("404.html"), 403


if __name__ == "__main__":
    init_db()
    app.run(host="0.0.0.0", port=5050, debug=False)

```