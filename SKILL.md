---
name: coupang
description: Use when searching Coupang Taiwan (www.tw.coupang.com) products, checking TWD prices/discounts/stock, or viewing/changing the user's Coupang cart (no checkout).
---

# coupang — Coupang Taiwan from a shell (no checkout)

`~/.local/bin/coupang` → `~/Projects/coupang-cli/coupang`. Stdlib Python 3.8+ with zero dependencies. Runs over plain HTTPS with automatic background-tab Edge CDP fallback when Akamai bot challenges occur. Start with `coupang doctor`.

- `coupang search <keywords> --json [--sort price|relevance|sales|newest] [--limit N] [--min/--max TWD] [--page P]`
  returns items[] with id, item_id, vendor_item_id, title, price, origin_price, discount, rating, reviews, rocket, out_of_stock, url
  CAVEAT (measured 2026-10-30): sponsored/ad slots (pinned regardless of sort/page) and
  listings priced exactly NT$99,999 (a seller placeholder, confirmed on the product page)
  are dropped. Coupang's own price/price-desc sort is only reliable for the first ~7-8
  items per request, so the CLI fetches extra remote pages as needed, dedupes by base
  product id, sorts client-side, then applies --page/--limit as a window over that pool
  (not a literal pass-through of Coupang's own page param).
  CAVEAT (added 2026-10-30): human-mode (non --json) search table shows a short
  https://www.tw.coupang.com/products/<id> URL instead of Coupang's ~400-char tracked
  link (itemId/vendorItemId/sourceType/clickEventId...) to avoid flooding the terminal;
  --json still returns the full tracked URL unchanged.
- `coupang product <id|url> --json` (alias: `item`)
  returns title, brand, price, origin_price, discount, rating, reviews, in_stock, shipping (火箭速配), category, variants[] (selectable specs)
- `coupang price <id> <id>... --json`
  quick price check for 1 or more products for price monitoring (price, origin_price, discount, in_stock, checked_at)
- Login: never ask for or type passwords/OTP, and never ask the user to paste cookies into chat. Tell
  the user: DevTools > Network > click any www.tw.coupang.com request > copy the `Cookie:` request header
  to the clipboard, then run `coupang login` themselves (or pipe it in on stdin). No usable Cookie
  header = exit 2 with those steps. `login --from-browser` is opt-in only (Edge on CDP 9222).
  `whoami` checks the saved session over the cart API (name, member_srl, cart items). Exit 3 = run `coupang login` again.
- Cart (needs login):
  - `coupang cart list`
  - `coupang cart add <id|url> [--qty N] [--spec TEXT] --yes`
  - `coupang cart qty <line#|id> <N> --yes`
  - `coupang cart rm <line#|id> --yes`
  Always run first WITHOUT `--yes` to get the dry run, show it to the user, and pass `--yes` only after user confirmation.
  The CLI automatically re-reads the cart from the API and verifies that the change landed.
- Safety: Never call checkout, payment, or order submission endpoints. The CLI has no checkout commands by design.
