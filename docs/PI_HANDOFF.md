# Raspberry Pi Handoff: Daily Instagram Follower Count

Goal: a Raspberry Pi runs a daily job (cron) that opens Instagram in a headless browser, logs in to **your own account**, reads the follower and following counts, and appends them to a CSV.

This doc is for a local session (on the Pi or on a laptop with SSH access to it). It is self-contained.

---

## 1. Current status

| Item | State |
|---|---|
| MCP server (`src/`, `dist/`) | Builds (`npm run build`). Login via `instagram-private-api` **fails**: Instagram returns `400 Your version of Instagram is out of date`, even with the version override (`456.1.0.56.78` / code `386215068`). |
| Browser-based counter | **Not written yet.** This is the job for the local session. |
| Pi | Not set up. No SSH access from the cloud session. |
| Credentials | Not stored anywhere in the repo. Must be placed on the Pi by the owner, never committed. |

Why a browser instead of the API: the API library's emulated app version is rejected. A real Chromium session on the Pi sidesteps the app-version check. It may still hit Instagram's security checks (see section 6).

---

## 2. Prerequisites on the Pi

- Raspberry Pi OS (64-bit recommended), with internet access.
- A user account (examples below use `pi`; substitute yours).
- The owner's Instagram login available on a phone, for approving a new-device check.

---

## 3. Setup steps

Run these on the Pi.

### 3.1 System packages

```bash
sudo apt update
sudo apt install -y chromium git curl
# Node.js 20 LTS (or newer) from NodeSource, if `node -v` is missing or < 18:
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v && npm -v && chromium --version
```

`chromium` is the Debian package, so the binary is `/usr/bin/chromium` (on some Pi OS builds it is `chromium-browser`; check with `which chromium chromium-browser`).

### 3.2 Clone the repo and check out the working branch

```bash
cd ~
git clone https://github.com/joshuadaniel-8090/insta-mcp-followers-handle.git
cd insta-mcp-followers-handle
git checkout ccr-e0215717-oul1ny
npm install
npm run build
```

Note: the build only proves the TypeScript compiles. The browser counter in section 4 is separate.

### 3.3 Create the credentials file (do this yourself)

```bash
cp .env.example .env 2>/dev/null || touch .env
nano .env
```

Put this in `.env` (values are yours, do not paste them into chat or commit them):

```
INSTAGRAM_USERNAME=your_username
INSTAGRAM_PASSWORD=your_password
```

Then lock it down:

```bash
chmod 600 .env
```

Confirm `.env` is ignored by git before going further:

```bash
git check-ignore -v .env || echo "NOT IGNORED - add .env to .gitignore"
```

### 3.4 Install the browser library

Use the system Chromium instead of downloading one:

```bash
npm install --save-dev playwright-core
```

Launch with `executablePath: '/usr/bin/chromium'` (or `chromium-browser`, whichever `which` found).

---

## 4. What the browser job must do (to build in the local session)

Create `scripts/follower-count.mjs` (or similar). Required behaviour:

1. Load `.env` (`dotenv` is already a dependency).
2. Launch headless Chromium at the system path, with a realistic user agent and a persistent profile dir (e.g. `~/.ig-profile`) so the session is reused between runs.
3. If not logged in, go to `https://www.instagram.com/accounts/login/`, fill username/password, submit, and wait for the home page. Save the session via the persistent profile.
4. Open `https://www.instagram.com/<username>/`.
5. Read the **followers** and **following** counts. Prefer the `meta[name="description"]` content or the header counts; take the number text, convert `1,234` / `12.3K` / `1.2M` to an integer.
6. Append a row to `~/ig-counts.csv`: `timestamp_iso,followers,following`. Write the header on first run.
7. Exit non-zero on failure, and log to `~/ig-counts.log`. Do not retry in a tight loop. One attempt per run.
8. Take a screenshot to `~/ig-debug.png` on failure, to help diagnose selector breakage.

Keep the run to **once per day**. Do not add retries or other accounts.

Test it once by hand (step 5) before adding cron.

---

## 5. First run (manual, required)

```bash
cd ~/insta-mcp-followers-handle
node scripts/follower-count.mjs
```

Expect Instagram to ask for one of:
- a **new device / suspicious login** approval: approve it in the Instagram app on the phone, then re-run;
- a **verification code** (email or SMS): enter it, if the script supports it, or complete it once in a browser on the Pi with a display;
- a **checkpoint** ("Confirm it's you"): complete it in the Instagram app, then re-run.

Once the first login works, the persistent profile should keep the session.

---

## 6. Cron schedule

```bash
crontab -e
```

Add one line (09:00 daily, adjust path and user):

```
0 9 * * * cd /home/pi/insta-mcp-followers-handle && /usr/bin/node scripts/follower-count.mjs >> /home/pi/ig-cron.log 2>&1
```

Check with `crontab -l`. Confirm the next day that `~/ig-counts.csv` has a new row.

---

## 7. Known risks

- **Account security flags.** Automated logins can trigger checkpoints or temporary locks. One run a day from one device is the lowest-risk pattern, but no guarantee.
- **Breakage.** Instagram changes its markup. Selectors in the script will break eventually; `ig-debug.png` and the log are for that.
- **API library is not used here.** `instagram-private-api` login is broken on current Instagram (see section 1). Do not switch back to it for this job without a fresh version check.
- **Terms of service.** Reading your own account's counts is the scope. Do not scrape other accounts.
- **Secrets.** `.env` stays local with `chmod 600`. Never commit it, never paste the password into a chat or issue.

---

## 8. Open questions for the owner

1. Is the Instagram account protected by 2FA? If yes, the script needs a way to enter the code (interactive first run is the simplest).
2. Do you want the CSV kept on the Pi only, or pushed somewhere (e.g. a Google Sheet)? Default: Pi only.
3. Is the Pi headless (no monitor)? If yes, the first login can't be done visually; use the approval-in-app route or run once on a laptop with the same profile copied over.

---

## 9. Done when

- `node scripts/follower-count.mjs` runs cleanly by hand and appends a row to `~/ig-counts.csv`.
- `crontab -l` shows the daily job.
- Next day, a second row appears in the CSV with no manual step.
