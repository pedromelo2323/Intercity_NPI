# Intercity Bolt NPI — Project Context

## Goal

Build a daily automated price checker for **Bolt ride-hailing** between two fixed points:

- **Origin:** Amsterdam Schiphol Airport
- **Destination:** Rotterdam Centraal Station

The target is to capture the **actual fare the user sees in the app** — not an estimate, the real displayed price — and track it over time (and potentially alert if it drops below a threshold).

---

## Why this is non-trivial

Bolt has **no public API**. Unlike Marktplaats or the Amsterdam bike depot (which expose clean JSON APIs), Bolt prices are:

- Generated dynamically (depend on GPS location, time of day, surge pricing, account)
- Only visible inside the app or after login on the web
- Not available on any public-facing search page

---

## Approaches evaluated

### Option 1 — Reverse-engineer the mobile app API ✅ Recommended
**Difficulty: Medium**

Use **mitmproxy** to intercept HTTPS traffic from the Bolt app on a phone. This reveals the exact private API endpoint, headers, and auth token that the app uses to fetch prices.

Once captured, replicating the call in a Python script is straightforward — identical in complexity to the bike depot scraper already built in this repo.

**Risks:**
- Bolt may use **certificate pinning** (blocks mitmproxy interception) — only discoverable by trying
- Auth token may expire frequently, requiring re-capture
- Technically violates Bolt's ToS (use at own risk)

**Next step:** Set up mitmproxy on Mac, configure phone proxy, open Bolt app, request a price quote for Schiphol → Rotterdam Centraal, and inspect the intercepted request.

### Option 2 — Browser automation (Selenium/Playwright)
**Difficulty: Hard**

Automate a browser to log into bolt.eu, input origin/destination, wait for price to render, and scrape it. Rejected because:
- Requires persistent login session
- 4+ points of failure (login, map load, price render, DOM scraping)
- Bolt UI changes break it silently

### Option 3 — Third-party data
**Difficulty: Very Hard / Unlikely**

No reliable public Bolt price API exists.

---

## Chosen approach: mitmproxy intercept

### One-time setup (to be done next session)

1. Install mitmproxy on Mac:
   ```bash
   brew install mitmproxy
   ```

2. Start the proxy:
   ```bash
   mitmproxy --listen-port 8080
   ```

3. On your phone, set HTTP proxy to your Mac's local IP on port 8080

4. Install the mitmproxy CA certificate on the phone (visit `mitm.it` from the phone browser while proxy is active)

5. Open Bolt app, enter **Schiphol → Rotterdam Centraal**, request a price

6. In mitmproxy, find the request that returns price data — look for JSON responses containing fare/price fields

7. Note down:
   - Full request URL
   - All headers (especially `Authorization`)
   - Request body/parameters
   - Response structure (which field contains the price)

### After intercept

Build a Python script (similar to `check_tenways.py` and `check_marktplaats.py`) that:
- Replicates the captured API call
- Extracts the displayed fare
- Logs it daily
- Optionally sends a Telegram alert if price is below a set threshold

Schedule via GitHub Actions (same pattern as Bike Finder project).

---

## Related project

**Bike Finder** — [github.com/pedromelo2323/bike-finder](https://github.com/pedromelo2323/bike-finder)

Already built and running. Uses the same architecture (Python stdlib + GitHub Actions + Telegram). Good reference for the script structure and GitHub Actions workflow pattern to reuse here.

---

## Open questions for next session

1. iPhone or Android? (affects mitmproxy setup — Android is easier, iPhone requires trusting a profile)
2. Is certificate pinning present on Bolt? (unknown until tried)
3. What to do if token expires? (manual re-capture vs. automating login)
4. Price threshold for alert? Or just daily log of the price?
5. Should this also track **surge pricing patterns** over time (e.g. which hours are cheapest)?
