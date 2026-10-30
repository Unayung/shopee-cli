# shopee-cli

Agent-native **buyer** CLI for Shopee Taiwan (`shopee.tw`, TWD). Zero dependencies (Python 3.8+ stdlib).
Search, product, price checks, login, and a read-only cart view. **No checkout, no cart writes.**

> [!WARNING]
> **Shopee rate-limits hard: budget a few dozen requests per account per day, not hundreds.**
> Measured 2026-10-30: `429 Too Many Requests` after ~20 requests in ~10 min, and after ~40 requests in
> ~45 min Shopee refused even its own search page. The CLI paces requests (≥ 9 s, shared across processes),
> caches result pages for 10 min, and on a block exits 4 with a cool-down instead of retrying.
> Run one `shopee` command at a time, and don't work around the pacing. Details: [Rate limits](#rate-limits-built-in-please-dont-work-around-them).

```bash
shopee doctor
shopee search 衛生紙 [--sort relevance|price|price-desc|sales|newest] [--page 2] [--limit 20] [--min 100 --max 500]
shopee product 14191115.115552656           # alias: item; also accepts shopee.tw product URLs
shopee price 14191115.115552656 50662979.24830833184
shopee raw '/api/v4/pdp/get_pc?item_id=...&shop_id=...'   # GET, for debugging
```

Product ids are `<shopid>.<itemid>` (the same pair Shopee puts in `...-i.<shopid>.<itemid>` URLs).

## Transport: everything goes through a background tab in your Edge

Shopee signs its web API calls in the page (`af-ac-enc-dat`, `x-sap-sec`, ... minted by its
anti-fraud SDK). Measured 2026-10-30:

| what | endpoint | transport | needs |
|---|---|---|---|
| search | `GET /api/v4/search/search_items` (issued by `shopee.tw/search?keyword=`) | page's own request, read via CDP | Edge signed in |
| product / price | `GET /api/v4/pdp/get_pc` (issued by `shopee.tw/product/<shopid>/<itemid>`) | page's own request | Edge signed in |
| login / whoami | `GET /api/v4/account/basic/get_account_info` | **plain HTTPS** with the saved cookies | saved session |
| cart list | `POST /api/v4/cart/get` (issued by `shopee.tw/cart`) | page's own request | login |
| cart add / qty / rm | `POST /api/v4/cart/add_to_cart`, `/api/v4/cart/update` | **disabled**: in-page XHR refused 3/3, see Cart | n/a |

- Plain HTTPS to `search_items` gets HTTP 403 `{"error":90309999}`; so does `fetch()` inside the page.
  Shopee's SDK hooks `XMLHttpRequest` only, and even those calls were refused about 2 times in 3, so
  reads use the page's own requests (200 on ~20 loads in a row, until the escalation described below).
- The tab is created with `Target.createTarget(background=true)`, never focused, and closed afterwards
  (if it is the last tab left, it is blanked instead, so Edge does not quit). User tabs are never touched.
- Signed out, Shopee blocks product pages (traffic-verify redirect) and paging; a captcha appeared once
  after ~9 guest requests. The CLI never solves captchas.

## Rate limits (built in, please don't work around them)

Measured 2026-10-30: Shopee answered a page load with an HTML `429 Too Many Requests` after
~12 page loads + ~8 XHRs in ~10 minutes (about 9 s apart). A 10-minute pause cleared it. After that,
~20 reads at ≥ 25 s apart went through without a 429.

- Later in the same session, after ~40 requests in ~45 min, Shopee refused even the real search
  page's own `search_items` (`90309999`) on 5 loads in a row, while the cart page still loaded.
  Expect a daily budget of a few dozen requests per account, not hundreds.
- **Pacing:** at least `SHOPEE_MIN_INTERVAL` seconds (default **9**) between Shopee requests, shared by
  every `shopee` process through a timestamp file (`~/.config/shopee/.last_request`). Agents running
  many commands should raise it (e.g. `SHOPEE_MIN_INTERVAL=25`).
- **No retry loops.** On a 429, anti-bot refusal (`90309999`), captcha or traffic-verify page, the
  command stops with **exit 4**, says how long to wait, and records a cool-down
  (`~/.config/shopee/.cooldown_until`: 10 min, 3 min for `90309999`). Until it expires every later
  `shopee` call refuses to send anything (`shopee doctor` shows the time left).
- **Up-front cost:** `price` with several ids and price-sorted pools that are not cached
  print "needs N Shopee requests, about M min" to stderr before starting.
- **Cache:** each Shopee result page is cached for 10 minutes (`SHOPEE_CACHE_TTL`; `--fresh` bypasses),
  so paging and switching between `--page` values does not re-hit Shopee.
- `shopee doctor` sends at most one request (plain-HTTPS `get_account_info`); `--live` adds one search page.

## Prices (read this before comparing)

Shopee API prices are integers = TWD × 100000. Listings often have variants with different prices.

- `price` = the **lowest variant price right now** (`price_min`, after product promotions, before vouchers).
  `price_max` is the highest variant; `product` lists every variant (`models[]`) with its own price.
- `card_price` (search) / `final_price` (product) = what Shopee's card shows, which can subtract a
  recommended shop voucher ("券後價"). Example measured: 五月花 `price` NT$995 (110抽 variant) ~ NT$1,175,
  card/券後價 NT$915 with voucher YFYSN1080 (NT$80 off, min spend NT$800). It is shown, never sorted on.
- Search sorts by `price`. Shopee's own price sort orders by the card number, so `--sort price` /
  `price-desc` pool Shopee's first 2 × 60 price-sorted results, drop ads, re-sort by `price`, and page
  through that pool. Results are monotonic, but only cover that pool, not Shopee's whole catalog.
- Dropped from results: ads (`adsid` / `is_ads`; 32 of 60 on a signed-in 衛生紙 page), duplicate
  (shopid,itemid), hidden prices (in price sorts), and listings at exactly NT$99,999. Every one of those
  seen was a gift-only placeholder ("<贈品專用，請勿自行下單！>").
- NT$1-3 listings are real: one checked on its product page was NT$1 for a single pack (`【活動用】`), plus NT$45-165 shipping.
  They are kept, and they lead `--sort price`.
- Relevance results reorder between calls on Shopee's side (page 1/2 overlapped 1/10 without caching).
  The page cache plus a 30-minute memory of ids already shown on earlier pages keeps page N from
  repeating page < N.

## Login (no password, ever)

Shopee signs in with a password (+ OTP), so the CLI never takes credentials. It logs in with the `Cookie`
request header of a browser session you already have:

1. In your browser: DevTools > Network > click any request to `shopee.tw` > Request Headers.
2. Copy the value of `Cookie:` (with or without the leading `Cookie: `). It must be from a signed-in
   session (`SPC_U` is a number, not `-`).
3. Run `shopee login`. It reads the clipboard, or stdin when piped.

```bash
shopee login                    # Cookie header from stdin (piped) or the clipboard;
                                #   nothing usable there = exit 2 with these steps
shopee login --from-clipboard   # clipboard only
shopee login --cookie-stdin     # stdin only
shopee login --keyring          # store cookies in the OS keyring (secret-tool / macOS security)
shopee login --from-browser     # opt-in: copy the shopee.tw session from Edge on CDP 9222; if Edge is
                                #   not signed in, opens the sign-in page in a background tab (--wait 300)
shopee whoami                   # verifies the SAVED session over plain HTTPS
shopee logout                   # deletes only the CLI's saved session; Edge stays signed in
```

Session: `~/.config/shopee/state.json` (0600, moved by `SHOPEE_HOME`) or keyring; `SHOPEE_COOKIE` overrides.
Before saving, the header is checked with `get_account_info` over plain HTTPS. Exit 3 means there is no valid
session (or it expired): log in again. Only `shopee.tw` cookies are used (seller.shopee.tw is ignored). Because
the signed transport runs in Edge, `cart list` refuses to run when Edge is signed in as a different user than
the saved session.

**Edge on CDP 9222 is needed for search, product (and price) and cart**, which run inside a background tab; Setup: [Browser on CDP 9222](#browser-on-cdp-9222-setup).
`login` (without `--from-browser`) and `whoami` use plain HTTPS only.

## Browser on CDP 9222 (setup)

The commands listed above as needing Edge drive a background tab in your own browser through the Chrome DevTools
Protocol, so their requests carry a real browser's fingerprint and sign-in. The CLI never launches a browser;
start one yourself with remote debugging on:

```bash
# Linux (Edge; google-chrome / chromium take the same flags)
microsoft-edge-stable --remote-debugging-port=9222 --user-data-dir="$HOME/.config/microsoft-edge-debug"

# macOS
"/Applications/Microsoft Edge.app/Contents/MacOS/Microsoft Edge" --remote-debugging-port=9222 \
  --user-data-dir="$HOME/Library/Application Support/Microsoft Edge Debug"
```

- `--user-data-dir` must point at a separate profile: Chromium-based browsers ignore `--remote-debugging-port` on
  the default profile (Chrome 136+). Sign in to shopee.tw once inside this profile; it stays signed in across
  restarts. If that profile is already open without the flag, quit the browser first: launching again only opens
  a window in the running instance.
- Check it: `curl -s http://127.0.0.1:9222/json/version` prints the browser version, and `shopee doctor` shows
  `cdp ok`. Another port or host: `--cdp http://127.0.0.1:9333` or `SHOPEE_CDP=http://127.0.0.1:9333`.
- `--remote-allow-origins` is not needed: the CLI connects without an `Origin` header (verified 2026-10-30), and
  leaving it out keeps web pages from connecting. The port listens on 127.0.0.1 only; keep it that way, because
  anything that can reach it controls the browser and every session in it.
- The CLI only creates background tabs (`Target.createTarget(background=true)`), never focuses or touches yours,
  closes its own when done, and blanks rather than closes the last tab so the browser does not quit. Run one
  command at a time.

## Cart (購物車): read-only

```bash
shopee cart list            # every line: line#, id, modelid, quantity (+ name/spec/price/checked for the first 20)
```

`cart add / qty / rm` exist but **refuse up front with exit 2** and send nothing. Measured 2026-10-30:
`cart/add_to_cart` through the signed in-page XHR was refused 3/3 (`90309999`), and the re-read cart
was unchanged each time. Clicking the real buttons by automation was declined to protect the account,
which is also the user's seller shop. Change the cart in Edge yourself.

`cart list` shows every line (47 measured), but Shopee only sends details (name, variant, price,
checked state) for the first 20. The page loads the rest on scroll, which does not happen in a hidden
background tab, so lines 21+ show id, variant id and quantity only.

Every command takes `--json` → `{ok, data|error}`. Exit codes: 0 ok, 1 api, 2 usage, 3 auth, 4 net/blocked.

No checkout, payment or order placement by design. Unofficial private API, and it can break at any time.
Use at human rates: Shopee bans automation aggressively.
