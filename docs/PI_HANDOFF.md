# Raspberry Pi Handoff: Daily Instagram Followers Snapshot to Google Sheets

Goal: once a day, a Raspberry Pi (desktop OS) opens Instagram in Chromium, logs in to **your own account**, reads the full followers and following lists, and POSTs them to a Google Apps Script web app. The script writes the results into your Google Sheet and records who joined and who left.

Scope and limits (agreed):
- Schedule: **once a day**, Pi local time.
- Join time: Instagram does not expose when someone followed you. The sheet records the **first time the job saw them**, which is the day of the run. Accurate to one day.
- First login: done by you, on a monitor connected to the Pi. No 2FA on the account.
- Data: username, status, first seen, last seen, left at. Plus counts per run.

This doc is for a local session on the Pi (or with SSH to it). Work through it top to bottom.

---

## 1. Current status

| Item | State |
|---|---|
| Google Sheet | You create it (step 2). |
| Apps Script web app | Code in section 4. You paste and deploy it. |
| Pi scraper | **Not written yet.** The local session writes `scripts/follower-snapshot.mjs` per section 5. |
| Cron | Not set up (section 7). |
| `instagram-private-api` (MCP server) | Login broken (`400 out of date`). Not used by this job. |

---

## 2. Create the spreadsheet (you)

1. Create a new Google Sheet. Name it e.g. `IG Followers`.
2. Note nothing else yet. The script creates its own tabs on first run: `Snapshots`, `Followers`, `Following`, `Events`.

---

## 3. Pi prerequisites

- Raspberry Pi OS **with desktop** (confirmed). Chromium is usually preinstalled; if not, see 6.1.
- Network access.
- Monitor connected for the first run (section 8).
- Confirm the Pi's local time zone:
  ```bash
  timedatectl        # look for "Time zone" and "Local time"
  ```
  If wrong: `sudo raspi-config` → Localisation → Timezone. The cron schedule and timestamps use this.

---

## 4. Apps Script (you paste this into the sheet)

1. In the sheet: **Extensions → Apps Script**.
2. Replace the default `Code.gs` content with the code below.
3. **Project Settings (gear) → Script properties → Add**: name `INGEST_TOKEN`, value = a long random string you generate (e.g. `openssl rand -hex 32` on the Pi). Keep it; the Pi needs the same value.
4. **Deploy → New deployment → type: Web app**. Execute as: **Me**. Who has access: **Anyone**. Deploy and **copy the Web app URL** (ends in `/exec`). Authorize when asked.

```javascript
// Code.gs — receives snapshots from the Pi and updates the sheet.
const TOKEN_PROP = 'INGEST_TOKEN';

function doPost(e) {
  const props = PropertiesService.getScriptProperties();
  let body;
  try {
    body = JSON.parse(e.postData.contents);
  } catch (err) {
    return out({ ok: false, error: 'bad json' });
  }
  if (!body.token || body.token !== props.getProperty(TOKEN_PROP)) {
    return out({ ok: false, error: 'unauthorized' });
  }

  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const ts = String(body.timestamp_local); // e.g. 2026-11-22T09:00:12+05:30

  // One row per run
  const snap = tab(ss, 'Snapshots', ['timestamp', 'follower_count', 'following_count', 'followers_scraped', 'following_scraped']);
  snap.appendRow([ts, body.follower_count, body.following_count, body.followers.length, body.following.length]);

  const events = tab(ss, 'Events', ['timestamp', 'list', 'username', 'event']);
  const newEvents = [];
  syncList(ss, 'Followers', body.followers, ts, 'followers', newEvents);
  syncList(ss, 'Following', body.following, ts, 'following', newEvents);
  if (newEvents.length) {
    events.getRange(events.getLastRow() + 1, 1, newEvents.length, 4).setValues(newEvents);
  }

  return out({ ok: true, events: newEvents.length });
}

// Keeps a tab with columns: username, status, first_seen, last_seen, left_at
function syncList(ss, name, usernames, ts, listLabel, newEvents) {
  const sh = tab(ss, name, ['username', 'status', 'first_seen', 'last_seen', 'left_at']);
  const data = sh.getDataRange().getValues().slice(1);
  const current = new Set(usernames);
  const seen = new Set();
  const rows = [];

  for (const r of data) {
    const [u, status, first, last, left] = r;
    if (!u) continue;
    seen.add(u);
    if (current.has(u)) {
      if (status === 'left') {
        rows.push([u, 'active', first, ts, '']);
        newEvents.push([ts, listLabel, u, 'rejoined']);
      } else {
        rows.push([u, 'active', first, ts, left]);
      }
    } else {
      if (status === 'active') {
        rows.push([u, 'left', first, last, ts]);
        newEvents.push([ts, listLabel, u, 'left']);
      } else {
        rows.push([u, status, first, last, left]);
      }
    }
  }
  for (const u of usernames) {
    if (!seen.has(u)) {
      rows.push([u, 'active', ts, ts, '']);
      newEvents.push([ts, listLabel, u, 'joined']);
    }
  }

  sh.getRange(2, 1, Math.max(sh.getMaxRows() - 1, 1), 5).clearContent();
  if (rows.length) sh.getRange(2, 1, rows.length, 5).setValues(rows);
}

function tab(ss, name, headers) {
  let sh = ss.getSheetByName(name);
  if (!sh) {
    sh = ss.insertSheet(name);
    sh.appendRow(headers);
    sh.setFrozenRows(1);
  }
  return sh;
}

function out(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj)).setMimeType(ContentService.MimeType.JSON);
}
```

Test it from the Pi later with a tiny payload (section 9). Set `INGEST_TOKEN` before testing.

Note: each run rewrites the `Followers` and `Following` tabs. Fine for a few thousand rows. Apps Script has a per-execution time limit (about 6 minutes); if the lists get very large, tell the local session and we will switch to a smarter diff.

---

## 5. Pi scraper (local session writes this)

Create `scripts/follower-snapshot.mjs`. Requirements:

**Setup**
- Node 20+, ESM (`"type": "module"` already set in `package.json`).
- Dependencies: `playwright-core` (use system Chromium), `dotenv` (already present).
- Launch: `chromium.launchPersistentContext(~/.ig-profile, { executablePath: '/usr/bin/chromium', headless: process.env.HEADLESS !== '0', viewport: {width: 1280, height: 900} })`. Use the persistent profile so the login survives between runs.
- `.env` (chmod 600) holds: `INSTAGRAM_USERNAME`, `INSTAGRAM_PASSWORD`, `INGEST_TOKEN`, `APPS_SCRIPT_URL`, and optional `MAX_SCROLLS` (default 400).

**Login**
- Go to `https://www.instagram.com/`. If the login form is present, fill username/password, submit, wait for the logged-in home page.
- If a "Save login info" or "Turn on notifications" prompt appears, dismiss it (not Save login info is fine either way; just be consistent).
- If a challenge, checkpoint, or 2FA page appears: **stop, take a screenshot to `~/ig-debug.png`, log it, exit non-zero.** You handle it on the monitor, then re-run.

**Counts and lists**
- Open `https://www.instagram.com/<INSTAGRAM_USERNAME>/`.
- Read follower and following counts. Prefer the header links `a[href$="/followers/"]` and `a[href$="/following/"]`; parse `1,234`, `12.3K`, `1.2M` into integers. Fallback: `meta[name="description"]`.
- Click the followers link, wait for `div[role="dialog"]`. Find the scrollable child (`scrollHeight > clientHeight`). Scroll it repeatedly, collecting usernames from anchors whose `href` matches `/^\/[A-Za-z0-9._]+\/$/`, skipping reserved paths (`/explore/`, `/accounts/`, etc.).
- Stop when the collected count reaches the header count, or when no new usernames appear for about 5 consecutive scrolls, or when `MAX_SCROLLS` is hit. Log which condition ended it.
- Close the dialog and repeat for following.
- Random small delays (about 1–3 s) between scrolls and between the two lists. Do not run more than once per day.

**Output**
- Build the JSON payload: `{ token, timestamp_local, follower_count, following_count, followers: [...], following: [...] }`. `timestamp_local` is `new Date()` formatted with the Pi's local offset, e.g. `2026-11-22T09:00:12+05:30` (use `Intl`/manual formatting, not `toISOString()`, which is UTC).
- Save a copy to `~/ig-snapshots/YYYY-MM-DD-HHMM.json` before posting. Keep it even if posting fails.
- POST to `APPS_SCRIPT_URL` with `Content-Type: text/plain;charset=utf-8` and the JSON as body (Apps Script reads the raw body). Node `fetch` follows the Apps Script redirect; check the response JSON has `"ok": true`.

**Logging and failure**
- Append to `~/ig-snapshot.log`: start time, counts, scrape stop reason, post result, errors.
- On any failure: screenshot to `~/ig-debug.png`, log, exit non-zero. No retry loop.

**Do not**
- Put the password, token, or URL in code or in git. `.env` only.
- Scrape any account other than `INSTAGRAM_USERNAME`.

---

## 6. Install Chromium and Node (if missing)

### 6.1 Chromium
```bash
sudo apt update
sudo apt install -y chromium git curl
which chromium chromium-browser
```
Use whichever path exists for `executablePath`.

### 6.2 Node 20
```bash
node -v   # need v20+
# if missing or too old:
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

### 6.3 Repo
```bash
cd ~
git clone https://github.com/joshuadaniel-8090/insta-mcp-followers-handle.git
cd insta-mcp-followers-handle
git checkout ccr-e0215717-oul1ny
npm install
npm install playwright-core
```

---

## 7. Credentials file

```bash
cd ~/insta-mcp-followers-handle
nano .env
```

```
INSTAGRAM_USERNAME=your_username
INSTAGRAM_PASSWORD=your_password
INGEST_TOKEN=the_same_token_as_apps_script
APPS_SCRIPT_URL=https://script.google.com/macros/s/.../exec
MAX_SCROLLS=400
```

```bash
chmod 600 .env
git check-ignore -v .env || echo "NOT IGNORED - add .env to .gitignore"
```

---

## 8. First run (you, on the monitor)

```bash
cd ~/insta-mcp-followers-handle
HEADLESS=0 node scripts/follower-snapshot.mjs
```

- A Chromium window opens. Watch it log in.
- If Instagram shows a challenge or approval screen, handle it in the window or in the Instagram app, then re-run.
- Confirm: the lists scrape, a row appears in `Snapshots`, and the `Followers` and `Following` tabs fill in.
- After a successful run, the profile in `~/.ig-profile` keeps the session. Later cron runs can be headless.

---

## 9. Quick test of the Apps Script endpoint (before the first real run)

```bash
curl -sL -X POST "$APPS_SCRIPT_URL" \
  -H "Content-Type: text/plain;charset=utf-8" \
  --data '{"token":"'"$INGEST_TOKEN"'","timestamp_local":"2026-11-22T00:00:00+00:00","follower_count":1,"following_count":1,"followers":["testuser"],"following":["testuser"]}'
```

Expect `{"ok":true,...}`. Then delete that test row from `Snapshots`, `Followers`, `Following`, and `Events` so real data starts clean. (The `Events` tab will show two `joined` rows for `testuser`; delete those too.)

---

## 10. Cron (daily)

```bash
crontab -e
```

Add (09:00 Pi local time):

```
0 9 * * * cd /home/pi/insta-mcp-followers-handle && /usr/bin/node scripts/follower-snapshot.mjs >> /home/pi/ig-cron.log 2>&1
```

Adjust `/home/pi` if your user is not `pi`. Check with `crontab -l`. Cron uses the Pi's system time zone (step 3).

Note: on a desktop Pi, cron runs headless with the persistent profile. If it hits a challenge, it exits non-zero, and you fix it on the monitor with `HEADLESS=0`.

---

## 11. Known risks

- **Join and leave times are per run.** A follower who joins and leaves between two runs is not seen.
- **Account flags.** Scrolling the full list once a day is more than a normal user does. Keep it to once a day and do not add retries.
- **Breakage.** Instagram changes its markup. Selectors in section 5 will eventually break. `~/ig-debug.png` and the log are the first place to look.
- **Apps Script quotas.** Large lists or many runs can hit Google quotas. Watch the `Snapshots` tab.
- **Secrets.** `.env` stays on the Pi, `chmod 600`. Not in git, not in chat. The token and script URL are secrets too.
- **Scope.** Your own account only.

---

## 12. Open items

1. **Follower count** is unknown. The default is to scroll the full list, capped by `MAX_SCROLLS=400`. If the count is much higher, tell the local session the number and it will tune the cap or switch to a lighter daily check.

---

## 13. Done when

- `HEADLESS=0 node scripts/follower-snapshot.mjs` completes and fills the sheet.
- `crontab -l` shows the daily job.
- The next day, `Snapshots` has a new row, and `Events` shows any joins or leaves since the previous run.
