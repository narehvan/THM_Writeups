# Contents of the failed sign-in page
```html
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 8973
Server: Werkzeug/2.0.2 Python/3.10.12
Date: Sun, 01 Nov 2026 10:49:17 GMT

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>CorpNet NOC — Sign In</title>
<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --orange:      #e8640a;
  --orange-dark: #c4530a;
  --orange-pale: #fff4ec;
  --blue:        #1a5fa8;
  --blue-hi:     #2272c3;
  --blue-pale:   #edf4fc;
  --blue-glow:   rgba(26,95,168,0.15);
  --white:       #ffffff;
  --off-white:   #f8f9fb;
  --border:      #e2e6ed;
  --border-mid:  #cdd3dc;
  --text:        #1a2535;
  --text-mid:    #4a5a6e;
  --text-dim:    #8a96a8;
  --text-faint:  #c8d0da;
  --sans:        "Segoe UI", -apple-system, Arial, sans-serif;
  --mono:        "Consolas", "Courier New", monospace;
}

html, body { height: 100%; }
body {
  font-family: var(--sans);
  background: var(--off-white);
  color: var(--text);
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background-image:
    linear-gradient(var(--border) 1px, transparent 1px),
    linear-gradient(90deg, var(--border) 1px, transparent 1px);
  background-size: 36px 36px;
  background-color: #f0f3f7;
}

/* ── Topbar ── */
.topbar {
  background: var(--blue);
  height: 54px;
  display: flex; align-items: center;
  padding: 0 2rem;
  flex-shrink: 0;
  position: relative;
}
.topbar::after {
  content: "";
  position: absolute; bottom: 0; left: 0; right: 0; height: 3px;
  background: var(--orange);
}
.brand { display: flex; align-items: center; gap: 10px; }
.brand-mark {
  width: 30px; height: 30px;
  background: var(--orange);
  display: flex; align-items: center; justify-content: center;
}
.brand-mark svg { width: 15px; height: 15px; fill: #fff; }
.brand-name {
  font-size: 0.8rem; font-weight: 700;
  letter-spacing: 0.1em; text-transform: uppercase; color: #fff;
}
.brand-name span { font-weight: 400; color: rgba(255,255,255,0.5); margin-left: 6px; font-size: 0.72rem; }
.topbar-right { margin-left: auto; }
.status-pill {
  display: flex; align-items: center; gap: 5px;
  font-size: 0.62rem; color: rgba(255,255,255,0.5);
  letter-spacing: 0.06em; padding: 3px 9px;
  border: 1px solid rgba(255,255,255,0.12);
}
.status-pill .dot { width: 5px; height: 5px; border-radius: 50%; background: #4ade80; }

/* ── Page body ── */
.page-body {
  flex: 1; display: flex;
  align-items: center; justify-content: center;
  padding: 2.5rem 1rem;
}

.login-wrap { width: 100%; max-width: 420px; }

.login-eyebrow {
  font-size: 0.62rem; font-weight: 700;
  letter-spacing: 0.16em; text-transform: uppercase;
  color: var(--orange);
  margin-bottom: 0.45rem;
  display: flex; align-items: center; gap: 8px;
}
.login-eyebrow::before {
  content: ""; display: block;
  width: 18px; height: 2px; background: var(--orange);
}
.login-title {
  font-size: 1.45rem; font-weight: 300;
  color: var(--text); letter-spacing: -0.01em;
  margin-bottom: 0.2rem; line-height: 1.2;
}
.login-title strong { font-weight: 700; }
.login-sub { font-size: 0.76rem; color: var(--text-dim); margin-bottom: 1.8rem; }

/* ── Card ── */
.card {
  background: var(--white);
  border: 1px solid var(--border);
  box-shadow: 0 1px 3px rgba(0,0,0,0.06), 0 4px 16px rgba(0,0,0,0.05);
  overflow: hidden;
}
.card-stripe {
  height: 4px;
  background: linear-gradient(90deg, var(--orange) 0%, var(--blue) 60%, var(--blue-hi) 100%);
}
.card-body { padding: 1.8rem; }

/* ── Alert ── */
.alert {
  display: flex; align-items: flex-start; gap: 9px;
  padding: 0.65rem 0.85rem;
  background: #fff7f0; border: 1px solid #fdd0a8;
  border-left: 3px solid var(--orange);
  margin-bottom: 1.3rem;
  font-size: 0.77rem; color: #7a3300; line-height: 1.5;
}
.alert svg { width: 13px; height: 13px; fill: var(--orange); flex-shrink: 0; margin-top: 1px; }

/* ── Fields ── */
.field { margin-bottom: 1rem; }
.field label {
  display: block; font-size: 0.7rem; font-weight: 700;
  text-transform: uppercase; letter-spacing: 0.06em;
  color: var(--text-mid); margin-bottom: 0.35rem;
}
.input-wrap { position: relative; }
.input-wrap svg {
  position: absolute; left: 10px; top: 50%; transform: translateY(-50%);
  width: 13px; height: 13px; fill: var(--text-faint); pointer-events: none;
  transition: fill 0.15s;
}
.input-wrap input {
  width: 100%; padding: 0.58rem 0.75rem 0.58rem 2rem;
  font-size: 0.87rem; font-family: var(--sans);
  border: 1px solid var(--border-mid);
  background: var(--white); color: var(--text);
  outline: none;
  transition: border-color 0.15s, box-shadow 0.15s;
}
.input-wrap input::placeholder { color: var(--text-faint); }
.input-wrap input:focus { border-color: var(--blue); box-shadow: 0 0 0 3px var(--blue-glow); }
.input-wrap:focus-within svg { fill: var(--blue); }

/* ── Submit ── */
.btn-submit {
  width: 100%; margin-top: 0.3rem;
  padding: 0.62rem 1rem;
  background: var(--orange); border: none;
  color: #fff; font-family: var(--sans);
  font-size: 0.85rem; font-weight: 700;
  letter-spacing: 0.04em; cursor: pointer;
  display: flex; align-items: center; justify-content: center; gap: 7px;
  transition: background 0.15s, box-shadow 0.15s;
}
.btn-submit svg { width: 13px; height: 13px; fill: rgba(255,255,255,0.75); }
.btn-submit:hover { background: var(--orange-dark); box-shadow: 0 4px 12px rgba(232,100,10,0.3); }
.btn-submit:active { background: #a84408; }

/* ── Card footer ── */
.card-foot {
  padding: 0.65rem 1.8rem;
  background: #fafbfc; border-top: 1px solid var(--border);
  display: flex; justify-content: space-between; gap: 1rem; flex-wrap: wrap;
}
.foot-item {
  display: flex; align-items: center; gap: 4px;
  font-size: 0.62rem; color: var(--text-dim); letter-spacing: 0.04em;
}
.foot-item svg { width: 9px; height: 9px; fill: var(--text-faint); }

/* ── Page footer ── */
.page-footer {
  padding: 0.9rem 2rem; text-align: center;
  font-size: 0.62rem; color: var(--text-faint); letter-spacing: 0.06em;
  background: var(--white); border-top: 1px solid var(--border);
}
</style>
</head>
<body>

<header class="topbar">
  <div class="brand">
    <div class="brand-mark">
      <svg viewBox="0 0 16 16"><path d="M2 2h5v5H2zM9 2h5v5H9zM2 9h5v5H2zM11 9h2v2h2v2h-2v2h-2v-2H9v-2h2z"/></svg>
    </div>
    <div class="brand-name">CorpNet <span>Network Operations Centre</span></div>
  </div>
  <div class="topbar-right">
    <div class="status-pill"><span class="dot"></span><span>SYSTEMS NOMINAL</span></div>
  </div>
</header>

<div class="page-body">
  <div class="login-wrap">
    <div class="login-eyebrow">Operator Access</div>
    <h1 class="login-title"><strong>Sign In</strong> to NOC Portal</h1>
    <p class="login-sub">Restricted to authorised personnel. All sessions are monitored and logged.</p>

    <div class="card">
      <div class="card-stripe"></div>
      <div class="card-body">

        
        <div class="alert">
          <svg viewBox="0 0 16 16"><path d="M8 1a7 7 0 100 14A7 7 0 008 1zM7 4h2v5H7V4zm0 6h2v2H7v-2z"/></svg>
          Invalid username or password.
        </div>
        

        <form method="POST" action="/internal" autocomplete="off">
          <div class="field">
            <label for="username">Username</label>
            <div class="input-wrap">
              <input type="text" id="username" name="username"
                     placeholder="operator id" autofocus
                     autocomplete="off" spellcheck="false">
              <svg viewBox="0 0 16 16"><path d="M8 8a3 3 0 100-6 3 3 0 000 6zm-5 5a5 5 0 0110 0H3z"/></svg>
            </div>
          </div>
          <div class="field">
            <label for="password">Password</label>
            <div class="input-wrap">
              <input type="password" id="password" name="password"
                     placeholder="••••••••••••"
                     autocomplete="current-password">
              <svg viewBox="0 0 16 16"><path d="M11 6V5a3 3 0 00-6 0v1H3v8h10V6h-2zM7 5a1 1 0 012 0v1H7V5zm1 6a1 1 0 110-2 1 1 0 010 2z"/></svg>
            </div>
          </div>
          <button type="submit" class="btn-submit">
            <svg viewBox="0 0 16 16"><path d="M10 3h3v10h-3v-2H8v-1h2V6H8V5h2V3zM1 5h6v1H3v4h4v1H1V5z"/></svg>
            Authenticate
          </button>
        </form>

      </div>
      <div class="card-foot">
        <span class="foot-item">
          <svg viewBox="0 0 16 16"><path d="M8 1l6 3v4c0 3.5-2.5 6.5-6 7.5C2.5 14.5 0 11.5 0 8V4l8-3z"/></svg>
          Sec-Level: RESTRICTED
        </span>
        <span class="foot-item">
          <svg viewBox="0 0 16 16"><path d="M8 1a7 7 0 100 14A7 7 0 008 1zm.5 7.5v3h-1v-3h1zm0-4v2h-1V4.5h1z"/></svg>
          Session timeout: 30 min
        </span>
        <span class="foot-item">v2.4.1</span>
      </div>
    </div>
  </div>
</div>

<footer class="page-footer">
  CORPNET INFRASTRUCTURE &nbsp;|&nbsp; INTERNAL USE ONLY &nbsp;|&nbsp; UNAUTHORISED ACCESS IS PROHIBITED
</footer>

</body>
</html>
```