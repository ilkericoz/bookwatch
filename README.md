# bookwatch

Watches a bookings grid website (set `base_url` in `config.json`) for new photoshoot bookings in the configured cities, alerts subscribers on Telegram, and auto-claims ("Assign myself") bookings that fall inside configured date/time windows.

> **Independent project.** Not affiliated with, endorsed by, or sponsored by any booking platform or brand it can be pointed at. Use it only with accounts you're authorized to use, and in line with the target site's terms of service.

## How it works

The bot attaches to a real, user-launched Chrome over CDP (port 9223) and reuses its logged-in site session for all HTTP calls — no headless browser, no automation fingerprint.

- **Grid watcher** — polls the month-view bookings grid every few seconds, diffs booking IDs, and for each new booking: Telegram alert → autobook decision (keywords → date/time window → listing types → schedule conflict → tier delay) → claim.
- **Re-verifier** — every 30 min re-checks each claimed future booking: canceled, time changed, or no longer assigned to us → one alert (intentional drops don't repeat).
- **Crew sync** — grid bookings already claimed by our crew member are recorded into `crew_schedule.json` automatically.
- **Flask UI** (`ui.py`) — crew schedule viewer/editor and booking-window config editor, with heartbeat-based bot liveness indicator.

## Setup

**Requirements:** Python 3.10+, Chrome

```bash
pip install -r requirements.txt
playwright install chromium   # only the playwright driver is used (CDP attach)
```

1. Copy `.env.example` to `.env` and fill in the Telegram bot token + owner chat ID.
2. Launch Chrome with `--remote-debugging-port=9223` and a persistent `--user-data-dir`, and log into the site. The bot never starts Chrome itself — it attaches to this one.
3. Start the bot: `python main.py` (or `run_logged.bat` for a console-less logged run).
4. Optional: `python ui.py` for the web UI (`setup_auth.py` generates the login hash).

## Configuration — `config.json`

| Key | Description |
|-----|-------------|
| `base_url` | **Required.** Site root, e.g. `https://example.com` — the bot has no built-in site |
| `cdp_endpoint` | Chrome remote-debugging endpoint, e.g. `http://localhost:9223` |
| `sanity_phrase` | Optional text the grid page must contain; a miss is treated as logged-out/blocked |
| `crew_name` | Crew member name to match against assigned crew |
| `urls[].cities` | Cities to watch (omit to watch every city the account can see) |
| `urls[].keywords` | Listing names that trigger alerts |
| `urls[].autobook` | Master switch for auto-claiming |
| `urls[].autobook_date_ranges` | Windows (`from`/`to` dates, `time_from`/`time_to`, per-window `listing_types`, buffers) inside which bookings are claimed |
| `check_interval_min/max_seconds` | Randomized poll interval |

Config is hot-reloaded; no restart needed for window changes.

## Telegram commands

`/start` `/stop` — subscribe/unsubscribe to alerts · `/status` — bot state · `/screenshot` `/fast` `/normal` `/interval <min> <max>` `/subscribers` — admin only.

## Tests

```bash
python test_payload.py     # synthetic end-to-end claim flow (+ --live read-only parse)
python test_reverifier.py  # reverifier alert scenarios across cycles
```
