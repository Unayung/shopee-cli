---
name: shopee
description: Use when searching Shopee Taiwan (shopee.tw) as a buyer, comparing TWD prices across listings and variants, checking a product's variants/stock/seller, or viewing (read-only) the user's Shopee cart. Shopee rate-limits hard (a few dozen requests per account per day): one command at a time, never loop or retry on exit 4.
---

# shopee: Shopee Taiwan buyer CLI (no checkout)

`~/.local/bin/shopee` → `~/Projects/shopee-cli/shopee`. Stdlib Python 3.8+. Reads go through a background
tab of the user's signed-in Edge on CDP 9222 (`SHOPEE_CDP`), because Shopee signs its API calls in the page
and refuses plain HTTPS (`90309999`). Start with `shopee doctor` (it costs at most one request).

## ⚠ Rate limits: read first (exceeding them blocks the user's account, which is also their seller shop)
- Shopee answered `429 Too Many Requests` after ~12 page loads in ~10 min, and after ~40 requests in ~45
  min it refused even its own search page (`90309999`) (measured 2026-10-30). The account is also the
  user's seller shop, so keep usage to a few dozen requests and batch what you need. Run ONE `shopee` command at a time, never in parallel.
- Built in: ≥ `SHOPEE_MIN_INTERVAL` s (default 9) between requests across all processes, and a 10-minute
  cache per Shopee result page (`--fresh` bypasses it). For long sessions set `SHOPEE_MIN_INTERVAL=25`.
- Exit 4 = blocked (429 / 90309999 / captcha / traffic-verify) or a cool-down is active. Do NOT retry. Tell
  the user the wait from the message; every later call refuses until the cool-down ends (`shopee doctor` shows it).
- Multi-request commands print "needs N Shopee requests, about M min" to stderr first. Pass that on to the user.

## Read (Edge must be signed in on shopee.tw)
- `shopee search <q> --json [--sort relevance|price|price-desc|sales|newest] [--page P] [--limit N] [--min/--max TWD]`
  returns items[] with id (`<shopid>.<itemid>`), title, price, price_max, orig_price, discount_pct, card_price,
  card_voucher, rating, reviews, sold, shop_location, url. meta reports dropped ads/dupes/placeholders.
- `shopee product <id|url> --json` (alias `item`) returns price/price_max (variant range), orig_price,
  final_price (券後價 + voucher codes), stock/in_stock, brand, shop, shipping, tiers[], models[] (modelid,
  name, price, stock).
- `shopee price <id> <id>... --json` checks price/stock for monitoring, one product-page load per id.

## Price basis (state it when comparing)
- `price` = lowest variant price now (price_min, after product promotion, before vouchers). Search and
  product agree on it (measured: 995 = 995). Variants differ, so quote ranges (`price`~`price_max`) or a
  specific `models[]` entry.
- `card_price` / `final_price` subtract a recommended voucher (e.g. 995 → 券後價 915 with an NT$80 voucher,
  min spend NT$800). Never sort on them, and mention the voucher condition when you quote them.
- `--sort price/price-desc` re-sorts a pool of Shopee's first 120 price-sorted results by `price`: monotonic
  within that pool, not across the whole catalogue. Ads, duplicates and NT$99,999 gift placeholders are
  dropped. NT$1-3 single-pack listings are real but bait; check the per-unit value and shipping.

## Login (never ask for or type passwords/OTP; never ask the user to paste cookies into chat)
- Tell the user: DevTools > Network > click any shopee.tw request (signed in) > copy the `Cookie:` request
  header to the clipboard, then run `shopee login` themselves (or pipe it in on stdin). No usable Cookie
  header = exit 2 with those steps. `--from-browser` is opt-in only (Edge on CDP 9222; if Edge is signed out
  it opens the sign-in page in a background tab and waits for the user). `shopee whoami` verifies the SAVED
  session over plain HTTPS (username, userid). `shopee logout` deletes only the CLI's copy. Exit 3 = run
  `shopee login`. Login and whoami do not need Edge; search, product, price and cart do.

## Cart (read-only)
- `shopee cart list --json` lists every line (line, id, modelid, quantity). Name/spec/price/checked come
  back only for the first 20 lines (Shopee lazy-loads the rest; a hidden tab doesn't trigger it).
- `cart add/qty/rm` are DISABLED: they exit 2 without sending anything. Shopee's anti-bot refused API
  writes (`90309999`), and UI automation was declined to protect the user's seller account. Tell the user
  to change the cart in Edge themselves, and do not work around it.
- Never checkout, place an order or pay. The CLI has no such command, by design.
