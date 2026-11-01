```bash
curl $URL
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>CorpNet — Network Operations Centre</title>
<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --orange:      #e8640a;
  --orange-dark: #c4530a;
  --orange-pale: #fff4ec;
  --orange-mid:  #fde8d8;
  --blue:        #1a5fa8;
  --blue-hi:     #2272c3;
  --blue-pale:   #edf4fc;
  --white:       #ffffff;
  --off-white:   #f8f9fb;
  --border:      #e2e6ed;
  --border-lo:   #edf0f5;
  --text:        #1a2535;
  --text-mid:    #4a5a6e;
  --text-dim:    #8a96a8;
  --sans:        "Segoe UI", -apple-system, Arial, sans-serif;
}

body {
  font-family: var(--sans);
  background: var(--off-white);
  color: var(--text);
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* ── Topbar ── */
.topbar {
  background: var(--blue);
  height: 54px;
  display: flex;
  align-items: center;
  padding: 0 2.5rem;
  position: relative;
  flex-shrink: 0;
}
.topbar::after {
  content: "";
  position: absolute;
  bottom: 0; left: 0; right: 0; height: 3px;
  background: var(--orange);
}
.brand {
  display: flex;
  align-items: center;
  gap: 10px;
}
.brand-mark {
  width: 32px; height: 32px;
  background: var(--orange);
  display: flex; align-items: center; justify-content: center;
}
.brand-mark svg { width: 17px; height: 17px; fill: #fff; }
.brand-name {
  font-size: 0.82rem;
  font-weight: 700;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: #fff;
}
.brand-name span {
  font-weight: 400;
  color: rgba(255,255,255,0.55);
  margin-left: 8px;
  letter-spacing: 0.04em;
  font-size: 0.75rem;
}
.topbar-right {
  margin-left: auto;
  display: flex;
  align-items: center;
  gap: 6px;
}
.status-pill {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 0.65rem;
  color: rgba(255,255,255,0.6);
  letter-spacing: 0.06em;
  padding: 3px 10px;
  border: 1px solid rgba(255,255,255,0.15);
}
.status-pill .dot {
  width: 6px; height: 6px;
  border-radius: 50%;
  background: #4ade80;
  box-shadow: 0 0 5px #4ade80;
}

/* ── Hero ── */
.hero {
  background: linear-gradient(135deg, var(--blue) 0%, #0e3d70 100%);
  padding: 4.5rem 2.5rem;
  text-align: center;
  position: relative;
  overflow: hidden;
}
.hero::before {
  content: "";
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  background-image:
    linear-gradient(rgba(255,255,255,0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255,255,255,0.03) 1px, transparent 1px);
  background-size: 40px 40px;
}
.hero-inner { position: relative; max-width: 640px; margin: 0 auto; }
.hero-tag {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: rgba(232,100,10,0.15);
  border: 1px solid rgba(232,100,10,0.35);
  color: #fba96e;
  font-size: 0.65rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  padding: 4px 12px;
  margin-bottom: 1.2rem;
}
.hero-tag::before {
  content: "";
  width: 6px; height: 6px;
  border-radius: 50%;
  background: var(--orange);
}
.hero h1 {
  font-size: 2rem;
  font-weight: 300;
  color: #fff;
  line-height: 1.2;
  margin-bottom: 0.8rem;
  letter-spacing: -0.01em;
}
.hero h1 strong { font-weight: 700; }
.hero p {
  font-size: 0.9rem;
  color: rgba(255,255,255,0.6);
  line-height: 1.7;
  max-width: 480px;
  margin: 0 auto;
}

/* ── Divider strip ── */
.strip {
  height: 4px;
  background: linear-gradient(90deg, var(--orange) 0%, var(--blue-hi) 50%, var(--off-white) 100%);
}

/* ── Cards ── */
.cards-section {
  padding: 3rem 2.5rem;
  max-width: 960px;
  margin: 0 auto;
  width: 100%;
}
.cards-title {
  font-size: 0.65rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.14em;
  color: var(--text-dim);
  margin-bottom: 1.2rem;
}
.cards-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
}
.card {
  background: var(--white);
  border: 1px solid var(--border);
  padding: 1.4rem;
  position: relative;
  overflow: hidden;
  box-shadow: 0 1px 3px rgba(0,0,0,0.05);
  transition: box-shadow 0.15s;
}
.card::before {
  content: "";
  position: absolute;
  top: 0; left: 0; right: 0; height: 3px;
  background: var(--orange);
}
.card:nth-child(2)::before { background: var(--blue); }
.card:nth-child(3)::before { background: #0e7b4a; }
.card:hover { box-shadow: 0 4px 12px rgba(0,0,0,0.08); }
.card-icon {
  width: 36px; height: 36px;
  display: flex; align-items: center; justify-content: center;
  margin-bottom: 0.9rem;
}
.card:nth-child(1) .card-icon { background: var(--orange-pale); }
.card:nth-child(2) .card-icon { background: var(--blue-pale); }
.card:nth-child(3) .card-icon { background: #ecfdf5; }
.card:nth-child(1) .card-icon svg { fill: var(--orange); }
.card:nth-child(2) .card-icon svg { fill: var(--blue); }
.card:nth-child(3) .card-icon svg { fill: #0e7b4a; }
.card-icon svg { width: 18px; height: 18px; }
.card h3 {
  font-size: 0.88rem;
  font-weight: 700;
  color: var(--text);
  margin-bottom: 0.4rem;
}
.card p {
  font-size: 0.76rem;
  color: var(--text-mid);
  line-height: 1.6;
}

/* ── Stats bar ── */
.stats-bar {
  background: var(--white);
  border-top: 1px solid var(--border);
  border-bottom: 1px solid var(--border);
  display: flex;
  max-width: 100%;
}
.stat-item {
  flex: 1;
  padding: 1.1rem 2rem;
  border-right: 1px solid var(--border);
  text-align: center;
}
.stat-item:last-child { border-right: none; }
.stat-n {
  font-size: 1.5rem;
  font-weight: 700;
  color: var(--blue);
  line-height: 1;
}
.stat-item:nth-child(2) .stat-n { color: var(--orange); }
.stat-l {
  font-size: 0.67rem;
  color: var(--text-dim);
  margin-top: 3px;
  text-transform: uppercase;
  letter-spacing: 0.07em;
}

/* ── Footer ── */
.page-footer {
  margin-top: auto;
  background: var(--blue);
  padding: 1rem 2.5rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 0.5rem;
}
.page-footer p {
  font-size: 0.65rem;
  color: rgba(255,255,255,0.4);
  letter-spacing: 0.06em;
}
.footer-mark {
  display: flex;
  align-items: center;
  gap: 6px;
}
.footer-mark .dot {
  width: 5px; height: 5px;
  border-radius: 50%;
  background: var(--orange);
}
.footer-mark span {
  font-size: 0.62rem;
  color: rgba(255,255,255,0.35);
  letter-spacing: 0.1em;
  text-transform: uppercase;
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
    <div class="status-pill">
      <span class="dot"></span>
      <span>ALL SYSTEMS NOMINAL</span>
    </div>
  </div>
</header>

<div class="hero">
  <div class="hero-inner">
    <div class="hero-tag">Infrastructure Monitoring</div>
    <h1><strong>Network Operations</strong><br>Centre Portal</h1>
    <p>Centralised infrastructure monitoring, host health verification, and operator audit logging for the CorpNet internal network.</p>
  </div>
</div>

<div class="strip"></div>

<div class="stats-bar">
  <div class="stat-item">
    <div class="stat-n">14</div>
    <div class="stat-l">Monitored Hosts</div>
  </div>
  <div class="stat-item">
    <div class="stat-n">3</div>
    <div class="stat-l">Open Alerts</div>
  </div>
  <div class="stat-item">
    <div class="stat-n">99.4%</div>
    <div class="stat-l">Uptime (30d)</div>
  </div>
  <div class="stat-item">
    <div class="stat-n">4</div>
    <div class="stat-l">Network Segments</div>
  </div>
</div>

<div class="cards-section">
  <div class="cards-title">Platform Capabilities</div>
  <div class="cards-grid">
    <div class="card">
      <div class="card-icon">
        <svg viewBox="0 0 16 16"><path d="M8 1a7 7 0 100 14A7 7 0 008 1zm0 2a5 5 0 110 10A5 5 0 018 3zm-.5 2v4l3 1.5-.5.9L7 9.5V5h.5z"/></svg>
      </div>
      <h3>Uptime Monitoring</h3>
      <p>Continuous ICMP and TCP service checks across all managed network segments with automated alerting on state changes.</p>
    </div>
    <div class="card">
      <div class="card-icon">
        <svg viewBox="0 0 16 16"><path d="M8 1l7 13H1L8 1zm0 3L3 12h10L8 4zm-.5 3h1v3h-1V7zm0 4h1v1.5h-1V11z"/></svg>
      </div>
      <h3>Incident Alerting</h3>
      <p>Threshold-based alert routing to on-call operators with escalation policies and suppression windows for planned maintenance.</p>
    </div>
    <div class="card">
      <div class="card-icon">
        <svg viewBox="0 0 16 16"><path d="M1 2h14v1H1zM1 5h10v1H1zM1 8h14v1H1zM1 11h10v1H1zM1 14h14v1H1z"/></svg>
      </div>
      <h3>Operator Audit Trail</h3>
      <p>All operator actions are logged with timestamp and session context. Audit records are retained for compliance and forensic review.</p>
    </div>
  </div>
</div>

<footer class="page-footer">
  <p>CORPNET INFRASTRUCTURE &nbsp;&mdash;&nbsp; INTERNAL USE ONLY &nbsp;&mdash;&nbsp; UNAUTHORISED ACCESS IS PROHIBITED</p>
  <div class="footer-mark">
    <span class="dot"></span>
    <span>NOC Portal v2.4.1</span>
  </div>
</footer>

</body>
</html>
```