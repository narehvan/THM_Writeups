```
cat netops.db 
��4�7tableaudit_logaudit_logCREATE TABLE audit_log (
    id       INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT,
    action   TEXT,
    detail   TEXT,
    ts       TEXT
)P++Ytablesqlite_sequencesqlite_sequenceCREATE TABLE sqlite_sequence(name,seq)�H�otableusersusersCREATE TABLE users (
    id       INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL,
    role     TEXT NOT NULL DEFAULT 'viewer'
��3Mnetopsb9b3917940feb033ae62c693d3967ba4operator
���tK��=%13netopsHEALTH_CHECK127.0.0.1%0awhoami2026-05-19 03:16:04=%13netopsHEALTH_CHECK127.0.0.10%awhoami2026-05-19 03:15:524%3netops3admin' --LOGIN_FAIL2026-05-19 03:14:09
```