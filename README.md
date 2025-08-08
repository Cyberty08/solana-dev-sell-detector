# solana-dev-sell-detector
This repository contains a ready-to-run Solana Dev-Sell Detector that watches SPL token developer accounts for sells and sends Telegram alerts
# solana-dev-sell-detector (GitHub-ready)

This repository contains a ready-to-run **Solana Dev-Sell Detector** that watches SPL token developer accounts for sells and sends Telegram alerts. Copy the files below into a GitHub repo (or download) and follow the README to run via Docker.

---

## File tree

```
solana-dev-sell-detector/
├─ README.md
├─ multi_dev_sell_detector.py
├─ Dockerfile
├─ requirements.txt
├─ tokens.txt
├─ .env.example
├─ .gitignore
└─ detector_state.db   # created at runtime (do NOT commit)
```

---

## README.md

```markdown
# Solana Dev-Sell Detector

Monitors SPL token developer accounts (top token accounts) and sends Telegram alerts when balances drop beyond a threshold.

## Features
- Multi-token support (list tokens in `TOKENS` env or `tokens.txt`)
- Heuristic detection of likely dev/treasury accounts (top-N largest token accounts)
- SQLite state storage to deduplicate alerts and persist balances
- Telegram alerting
- Dockerized for easy deployment

## Quickstart
1. Fill `.env` (see `.env.example`).
2. Optionally edit `tokens.txt` (one mint per line) or set `TOKENS` env var.
3. Build: `docker build -t solana-multi-dev-bot .`
4. Run: `docker run --env-file .env -v $(pwd)/detector_state.db:/app/detector_state.db solana-multi-dev-bot`

## Important
- Use a reliable RPC provider if you monitor many tokens or short intervals (QuickNode, GenesysGo, etc.)
- `detector_state.db` keeps state across restarts — mount it into the container.

## Extending
- Add swap-confirmation by parsing recent transactions for known AMM program IDs.
- Add Discord or webhook support.
- Convert to fully async with websocket subscriptions.
```

````

---

## multi_dev_sell_detector.py

```python
#!/usr/bin/env python3
"""
Multi-token Solana dev-sell detector (same script provided earlier).
Place this file at the repo root.
"""

import os
import time
import sqlite3
import threading
from decimal import Decimal, getcontext
from solana.rpc.api import Client
from solana.publickey import PublicKey
import requests

getcontext().prec = 40

# ---------------- CONFIG (via env) ----------------
RPC_URL = os.getenv("RPC_URL", "https://api.mainnet-beta.solana.com")
TOKENS_CSV = os.getenv("TOKENS", "")  # comma-separated mints, e.g. "Mint1,Mint2"
TOKENS_FILE = os.getenv("TOKENS_FILE", "tokens.txt")  # fallback file
TOP_N = int(os.getenv("TOP_N", "6"))  # top largest token accounts to consider as devs
THRESHOLD_PCT = Decimal(os.getenv("THRESHOLD_PCT", "0.5"))  # percent decrease to alert (e.g., 0.5)
CHECK_INTERVAL = int(os.getenv("CHECK_INTERVAL", "30"))  # seconds between polls
AUTO_REFRESH_INTERVAL = int(os.getenv("AUTO_REFRESH_INTERVAL", "3600"))  # seconds to refresh top-N accounts
SQLITE_DB = os.getenv("SQLITE_DB", "detector_state.db")

TELEGRAM_TOKEN = os.getenv("TELEGRAM_TOKEN")
TELEGRAM_CHAT_ID = os.getenv("TELEGRAM_CHAT_ID")
TELEGRAM_ENABLED = bool(TELEGRAM_TOKEN and TELEGRAM_CHAT_ID)

# ----------------- INIT -----------------
client = Client(RPC_URL)

# ----------------- DB -----------------
conn = sqlite3.connect(SQLITE_DB, check_same_thread=False)
cur = conn.cursor()
cur.execute("""
CREATE TABLE IF NOT EXISTS tracked_accounts (
    token TEXT,
    account TEXT,
    balance TEXT,
    PRIMARY KEY (token, account)
)
""")
cur.execute("""
CREATE TABLE IF NOT EXISTS alerts (
    token TEXT,
    account TEXT,
    detected_at INTEGER,
    prev_balance TEXT,
    new_balance TEXT,
    sold_amount TEXT,
    UNIQUE(token, account, detected_at)
)
""")
conn.commit()

# ----------------- Helpers -----------------
def send_telegram(text: str):
    if not TELEGRAM_ENABLED:
        print("[TELEGRAM] not configured, skipping send. Message:\n", text)
        return
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
    payload = {"chat_id": TELEGRAM_CHAT_ID, "text": text}
    try:
        r = requests.post(url, json=payload, timeout=10)
        if r.status_code != 200:
            print("[TELEGRAM] non-200:", r.status_code, r.text)
    except Exception as e:
        print("[TELEGRAM] send error:", e)


def load_tokens():
    tokens = []
    if TOKENS_CSV:
        tokens = [t.strip() for t in TOKENS_CSV.split(",") if t.strip()]
    if not tokens and os.path.exists(TOKENS_FILE):
        with open(TOKENS_FILE, "r") as f:
            for line in f:
                s = line.strip()
                if s:
                    tokens.append(s)
    return tokens


def get_top_token_accounts(mint: str, top_n=6):
    try:
        resp = client.get_token_largest_accounts(PublicKey(mint))
        vals = resp.get("result", {}).get("value", []) or []
        accounts = [v["address"] for v in vals[:top_n]]
        return accounts
    except Exception as e:
        print(f"[ERROR] get_token_largest_accounts for {mint}: {e}")
        return []


def get_token_account_balance(account_pubkey: str):
    try:
        resp = client.get_token_account_balance(PublicKey(account_pubkey))
        val = resp.get("result", {}).get("value")
        if not val:
            return Decimal(0)
        amount = Decimal(val.get("amount", "0"))
        return amount
    except Exception as e:
        print(f"[ERROR] balance fetch {account_pubkey}: {e}")
        return Decimal(0)


def store_tracked_account(token: str, account: str, balance: Decimal):
    cur.execute("INSERT OR REPLACE INTO tracked_accounts (token, account, balance) VALUES (?, ?, ?)",
                (token, account, str(balance)))
    conn.commit()


def get_prev_balance(token: str, account: str) -> Decimal:
    cur.execute("SELECT balance FROM tracked_accounts WHERE token=? AND account=?", (token, account))
    r = cur.fetchone()
    if not r:
        return Decimal(-1)
    return Decimal(r[0])


def record_alert(token, account, prev_bal, new_bal, sold_amount):
    ts = int(time.time())
    cur.execute("INSERT OR IGNORE INTO alerts (token, account, detected_at, prev_balance, new_balance, sold_amount) VALUES (?, ?, ?, ?, ?, ?)",
                (token, account, ts, str(prev_bal), str(new_bal), str(sold_amount)))
    conn.commit()


# ----------------- Core monitoring -----------------
class TokenMonitor:
    def __init__(self, mint: str):
        self.mint = mint
        self.lock = threading.Lock()
        self.accounts = []
        self.refresh_accounts()

    def refresh_accounts(self):
        print(f"[{self.mint}] Refreshing top {TOP_N} token accounts...")
        try:
            top = get_top_token_accounts(self.mint, TOP_N)
            if not top:
                print(f"[{self.mint}] No top accounts found.")
                return
            with self.lock:
                self.accounts = top
            for acct in top:
                bal = get_token_account_balance(acct)
                prev = get_prev_balance(self.mint, acct)
                if prev == Decimal(-1):
                    store_tracked_account(self.mint, acct, bal)
                    print(f"[{self.mint}] Added tracked account {acct} balance={bal}")
        except Exception as e:
            print(f"[{self.mint}] Error refreshing accounts: {e}")

    def poll_once(self):
        with self.lock:
            accts = list(self.accounts)
        for acct in accts:
            try:
                new_bal = get_token_account_balance(acct)
                prev_bal = get_prev_balance(self.mint, acct)
                if prev_bal == Decimal(-1):
                    store_tracked_account(self.mint, acct, new_bal)
                    continue
                if new_bal < prev_bal:
                    if prev_bal == 0:
                        pct = Decimal(100)
                    else:
                        pct = (prev_bal - new_bal) / prev_bal * Decimal(100)
                    if pct >= THRESHOLD_PCT:
                        sold_amount = prev_bal - new_bal
                        msg = (
                            f"🚨 Dev Sell Detected\nToken: {self.mint}\nAccount: {acct}\n"
                            f"Prev (base units): {prev_bal}\nNew (base units): {new_bal}\n"
                            f"Sold (base units): {sold_amount}\nDrop: {pct:.6f}%\n"
                            f"RPC: {RPC_URL}"
                        )
                        print(msg)
                        send_telegram(msg)
                        record_alert(self.mint, acct, prev_bal, new_bal, sold_amount)
                        store_tracked_account(self.mint, acct, new_bal)
                    else:
                        print(f"[{self.mint}] Small decrease {acct}: {prev_bal} -> {new_bal} ({pct:.6f}%)")
                        store_tracked_account(self.mint, acct, new_bal)
                elif new_bal > prev_bal:
                    store_tracked_account(self.mint, acct, new_bal)
                else:
                    pass
            except Exception as e:
                print(f"[{self.mint}] error polling acct {acct}: {e}")


# ----------------- Runner -----------------
def run_monitor(tokens):
    monitors = {}
    for t in tokens:
        m = TokenMonitor(t)
        monitors[t] = m

    def loop():
        last_refresh = time.time()
        while True:
            for m in monitors.values():
                try:
                    m.poll_once()
                except Exception as e:
                    print("Polling error:", e)
            if AUTO_REFRESH_INTERVAL > 0 and (time.time() - last_refresh) >= AUTO_REFRESH_INTERVAL:
                print("Auto-refreshing all token tracked accounts...")
                for m in monitors.values():
                    try:
                        m.refresh_accounts()
                    except Exception as e:
                        print("Refresh error:", e)
                last_refresh = time.time()
            time.sleep(CHECK_INTERVAL)

    th = threading.Thread(target=loop, daemon=True)
    th.start()
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("Exiting...")


if __name__ == "__main__":
    def load_tokens_simple():
        tokens = []
        env = os.getenv("TOKENS", "")
        if env:
            tokens = [t.strip() for t in env.split(",") if t.strip()]
        if not tokens and os.path.exists("tokens.txt"):
            with open("tokens.txt", "r") as f:
                for line in f:
                    s = line.strip()
                    if s:
                        tokens.append(s)
        return tokens

    tokens = load_tokens_simple()
    if not tokens:
        print("No tokens specified. Set TOKENS env var or tokens.txt file.")
        exit(1)
    print("Monitoring tokens:", tokens)
    print(f"Threshold_pct={THRESHOLD_PCT}%, check_interval={CHECK_INTERVAL}s, top_n={TOP_N}")
    if TELEGRAM_ENABLED:
        print("Telegram alerts enabled.")
    else:
        print("Telegram NOT enabled (set TELEGRAM_TOKEN & TELEGRAM_CHAT_ID).")
    run_monitor(tokens)
````

---

## Dockerfile

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
COPY . /app
ENV PYTHONUNBUFFERED=1
CMD ["python", "multi_dev_sell_detector.py"]
```

---

## requirements.txt

```
solana
requests
python-dotenv
```

---

## tokens.txt (example)

```
So11111111111111111111111111111111111111112
<another-mint-address>
```

---

## .env.example

```env
RPC_URL=https://api.mainnet-beta.solana.com
TOKENS=So11111111111111111111111111111111111111112
TOP_N=6
THRESHOLD_PCT=0.5
CHECK_INTERVAL=30
AUTO_REFRESH_INTERVAL=3600
TELEGRAM_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
TELEGRAM_CHAT_ID=987654321
SQLITE_DB=detector_state.db
```

---

## .gitignore

```
detector_state.db
.env
__pycache__/
```

---

## Push to GitHub (quick steps)

1. `git init`
2. `git add .`
3. `git commit -m "Initial commit - Solana dev-sell detector"`
4. Create a repo on GitHub and follow their push instructions, e.g.:

   ```bash
   git remote add origin git@github.com:yourusername/solana-dev-sell-detector.git
   git branch -M main
   git push -u origin main
   ```

---

## Next steps I can do for you

* Add swap-confirmation (Raydium/Orca/Jupiter program IDs) for high-confidence sells.
* Convert to async websocket subscriptions for instant alerts.
* Add a GitHub Actions workflow to build and push a Docker image.

If you want me to implement any of those, tell me which one and I will update the repository.
