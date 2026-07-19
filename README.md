# Internship job watcher

Polls public job-board feeds for internship-related titles, remembers what you have already seen in **SQLite**, and sends **Telegram** notifications for new US-matching postings (see `config.json` for keywords and filters).

---

## Telegram setup

1. **Create a bot**  
   In Telegram, open a chat with [@BotFather](https://t.me/BotFather), send `/newbot`, and follow the prompts. BotFather gives you a **bot token** (looks like `123456789:AAH…`).

2. **Get your chat ID**  
   - Easiest: message [@userinfobot](https://t.me/userinfobot) or [@getidsbot](https://t.me/getidsbot); it replies with your numeric **user id** (use that as `TELEGRAM_CHAT_ID`).  
   - Or: add your bot to a group and use a group chat id if you want notifications there (group ids are often negative).

3. **Start the bot**  
   Open a private chat with your bot and tap **Start** so the bot can message you.

4. **Put secrets in `.env`** (on your Mac and again on the server — never commit this file):

   ```env
   TELEGRAM_BOT_TOKEN=paste_token_from_botfather
   TELEGRAM_CHAT_ID=paste_numeric_chat_id
   ```

---

## Local setup (Mac or any machine)

```bash
cd /path/to/job
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/playwright install chromium
cp config.example.json config.json   # optional; repo already has config.json
nano .env                             # add TELEGRAM_* as above
.venv/bin/python main.py --once     # one poll, then exit
```

- **`main.py --once`** — single poll (good for cron or testing).  
- **`main.py`** (no flag) — loops forever using `poll_interval_seconds` from `config.json`.

Optional environment variables (defaults match the systemd unit on Oracle):

| Variable | Default | Purpose |
|----------|---------|---------|
| `CONFIG_PATH` | `./config.json` | Path to job source config |
| `STATE_DB_PATH` | `./state.db` | SQLite file for “already seen” job ids |

---

## Oracle Cloud VM (production)

Full step-by-step (account, VM shape, SSH, security lists, rsync) is in:

**[`deploy/oracle/STEPS.txt`](deploy/oracle/STEPS.txt)**

Short version:

1. Create an **Always Free** Ubuntu **ARM64** instance with a **public IPv4** and your **SSH key**.
2. Security list: allow **TCP 22** from **your IP /32** (avoid `0.0.0.0/0` unless you know the risk).
3. SSH in as **`ubuntu`** (Ubuntu image) or **`opc`** (Oracle Linux).
4. Install `python3`, `python3-venv`, `git`, copy the project (`rsync` or `git clone`).
5. On the VM: `python3 -m venv .venv`, `pip install -r requirements.txt`, **`playwright install chromium`**, and on Linux **`sudo playwright install-deps`** (system libraries for headless Chrome). Create **`~/job/.env`** with the same Telegram variables as locally.
6. Test: `cd ~/job && .venv/bin/python main.py --once`
7. Install systemd:

   ```bash
   sudo cp ~/job/deploy/oracle/internship-watcher.service /etc/systemd/system/
   # If your user is opc or paths differ, edit the unit file first (User, WorkingDirectory, paths).
   sudo systemctl daemon-reload
   sudo systemctl enable internship-watcher
   sudo systemctl start internship-watcher
   sudo systemctl status internship-watcher
   journalctl -u internship-watcher -f
   ```

The bundled unit file sets **`PYTHONUNBUFFERED=1`** so log lines from Python show up in `journalctl` immediately.

**Updating code on the VM:** rsync or `git pull` on `~/job`, then `sudo systemctl restart internship-watcher`.

**First run:** If `state.db` is new or empty, the first successful poll can send **many** Telegram messages (everything looks “new”). After that, only new job ids per source are notified.

---

## Configuration

- **`config.json`** — list of companies (`sources`), poll interval, title keywords (`intern`, `internship`, `student`), and **`location_filter`** (`countries: ["US"]`) with heuristics in `main.py`.
- **`scripts/build_config.py`** — regenerates `config.json` from the `SOURCES` list in that script. Run:  
  `python3 scripts/build_config.py`

---

## Companies included (current `config.json`)

**300** employers are polled (Meta, Apple, Tesla, Coinbase, GitLab, Toast, Autodesk, and many more via Playwright/Phenom/Workday/Greenhouse/Ashby, etc.). Regenerate the list from config:

```bash
.venv/bin/python -c "import json; print(len(json.load(open('config.json'))['sources']), 'sources')"
```

**Feed types in use:** `greenhouse`, `workday`, `ashby`, `lever`, `smartrecruiters`, `pcsx`, `oracle_careers`, `phenom`, `playwright`, `avature`, `successfactors`, `amazon`, `google_careers`, `workable`, `valve`, `usajobs`, `bofa`.

---

## Intern Tracker webapp (optional)

Companion UI for marking companies applied, calendar deadlines, and Telegram sync.

| What | Where |
|------|-------|
| **Live app** | https://webapp-two-peach.vercel.app |
| **Source** | https://github.com/nehanataraj/internship-notifs |

Use the **same** `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` in the webapp Settings as in this repo's `.env`.

**One-time pin bootstrap** (creates the `JTRACK::` pinned message the watcher and webapp share):

```bash
.venv/bin/python scripts/bootstrap_pinned.py
```

**Regenerate `companies.json`** for the webapp repo:

```bash
.venv/bin/python scripts/build_webapp_data.py
# copy output into internship-notifs repo if you maintain your own fork
```

## License / ops

- Do not commit **`.env`** or **private SSH keys**.  
- Ensure the VM allows **HTTPS outbound** (443) to job boards and `api.telegram.org`.
