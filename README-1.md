# Badho_Bingo — Web + Telegram Bot

Real-time lottery platform. **Backend is just a Python Telegram bot. No Firebase.**
Data lives in a private Telegram channel that the bot owns.

```
Browser (GitHub Pages)  ──HTTPS──▶  Bot's HTTP API  ──▶  Telegram Bot API
                                                       │
                                                       └─▶  Storage Channel (private)
```

The bot token sits in the bot server only. The browser never sees it. The
browser authenticates by sending a `X-Web-User` header containing a random
web user_id it generated on first visit.

---

## 1. Prerequisites

- A Telegram bot (already done — `@badhobingo_bot`).
- A **private** Telegram channel where the bot is admin (Post + Delete messages).
- The bot token, the channel chat_id (negative, starts with `-100`), and your
  Telegram user ID.

## 2. Bot setup

### Local (Termux / your laptop)

```bash
cd Badho_Bingo_Web
python3 -m venv venv
source venv/bin/activate
pip install python-telegram-bot==20.7 aiohttp httpx

export BOT_TOKEN="8908230709:AAH-vzosCTwfkEmVYBkeRBkGw9WN_Vd6dZ4"
export STORAGE_CHANNEL_ID="-1004327682401"
export ADMIN_TG_ID="6591945625"

python bot.py
```

The bot starts polling Telegram and the HTTP API on `:8080`.

### PythonAnywhere

1. Upload the folder via "Files" tab.
2. Open a "Bash console".
3. `mkvirtualenv badho --python=python3.11`
4. `pip install python-telegram-bot==20.7 aiohttp httpx`
5. Edit `~/.bashrc` to add your three env vars (or use a `.env` file).
6. Run via "Tasks" → "Always-on task":
   `/home/<you>/.virtualenvs/badho/bin/python /home/<you>/Badho_Bingo_Web/bot.py`
7. Make `:8080` reachable from the public web: PA free tier doesn't expose
   arbitrary ports, so use **ngrok** or **cloudflared**:
   ```
   ./cloudflared tunnel --url http://localhost:8080
   ```
   That gives you a public HTTPS URL like `https://random.trycloudflare.com`.

## 3. Web app setup

`index.html` is a single static file. Host it anywhere that serves HTML.

### GitHub Pages (free)

1. Create a new GitHub repo, e.g. `badho-bingo-web`.
2. Add `index.html` to the repo root.
3. Settings → Pages → Source: `Deploy from a branch` → `main` / `(root)` → Save.
4. Wait ~1 min. Your site is at `https://<your-username>.github.io/badho-bingo-web/`.

### Tell the web app where the bot lives

When you open the site, open the browser console and run:

```js
localStorage.setItem('bb_api_base', 'https://your-tunnel-or-host.example');
location.reload();
```

Replace the URL with whatever your bot's public endpoint is (ngrok, cloudflare
tunnel, your VPS, etc.).

For local testing, if your phone is on the same wifi as your laptop, you can
use `http://192.168.x.x:8080` after running the bot.

## 4. First run

1. Visit your GitHub Pages URL.
2. Open the **Profile** tab. Note your `web_user_id` (e.g. `w_abc123...`).
3. In Telegram, open `@badhobingo_bot` and send `/link w_abc123...`.
4. The bot replies "Linked!". Now your web account is bound to your Telegram.
5. As admin (`ADMIN_TG_ID`), in Telegram send:
   ```
   /new_lottery Badho Bingo #1 | First round | 200
   ```
   The bot creates the lottery + 200 numbers.
6. Refresh the web app — the lottery appears under "Iskuɗii".

## 5. Admin actions

Admin commands all run inside Telegram with `@badhobingo_bot`:

| Command | What it does |
|---|---|
| `/new_lottery Title \| Description \| MaxNumber` | Create a lottery + numbers 1..N |
| `/list_lotteries` | Show all lotteries |
| `/delete_lottery <id>` | Delete a lottery + its numbers |
| `/complete_lottery <id>` | Mark completed (broadcasts message) |
| `/post_winner <lid> \| <1,5,12> \| <prize>` | Save winners doc |
| `/pending_payments` | List pending payments with Approve/Reject buttons |

The web app's "Admin" tab is only visible if you set
`localStorage.setItem('bb_is_admin', '1')` in the browser. Even then, the
**real** admin gate is `ADMIN_TG_ID` on the bot — the web app only builds the
command strings for you to copy-paste into Telegram.

To enable the admin tab visually:

```js
// in browser console
localStorage.setItem('bb_is_admin', '1'); location.reload();
```

## 6. Architecture notes

- **Storage** is a private Telegram channel. Each record is a message with
  content `<!--bb:{...json...}-->`. Updates are `editMessageText`, deletes
  are `deleteMessage`. The bot mirrors state in memory and persists to
  `badho_bingo_state.json` for cold-start recovery.
- **Race protection** on number claims: per-`asyncio.Lock` keyed on
  `(lottery_id, number_value)` inside the bot. The browser cannot bypass
  this because it only calls `/api/claim` over HTTP.
- **Real-time** on the web: simple 5-second polling of `/api/lottery/:id`.
  Adequate for hundreds of users. For thousands, swap polling for SSE.
- **Payment screenshots** are uploaded to the storage channel via
  `sendPhoto`. The web app stores the Telegram `file_id` and uses
  `/api/screenshot/:fid` to fetch them via `getFile`.

## 7. Limitations vs the Firebase version

- No native mobile app — web only.
- No sub-second real-time (polling at 5s).
- Atomic claim is in-memory, not transactional like Firestore. If the bot
  restarts mid-claim there's a small window where two claims could race;
  the post-restart state reload from disk catches it but only on next boot.
- Screenshots live in Telegram storage (max 20MB/file, 10GB total per bot).

## 8. Files

- `bot.py` — Python Telegram bot + HTTP API. Single file.
- `index.html` — single-file web app, Tailwind via CDN.
- `requirements.txt` — Python deps.
- `README.md` — this file.