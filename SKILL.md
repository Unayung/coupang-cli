---
name: coupang
description: Use when searching Coupang Taiwan (www.tw.coupang.com) products, checking TWD prices/discounts/stock, or viewing/changing the user's Coupang cart (no checkout).
---

# coupang — Coupang Taiwan from a shell (no checkout)

`~/.local/bin/coupang` → `~/Projects/coupang-cli/coupang`. Stdlib Python 3.8+ with zero dependencies. Runs over plain HTTPS with automatic background-tab Edge CDP fallback when Akamai bot challenges occur. Start with `coupang doctor`.

- `coupang search <keywords> --json [--sort price|relevance|sales|newest] [--limit N] [--min/--max TWD] [--page P]`
  returns items[] with id, item_id, vendor_item_id, title, price, origin_price, discount, rating, reviews, rocket, out_of_stock, url
- `coupang product <id|url> --json` (alias: `item`)
  returns title, brand, price, origin_price, discount, rating, reviews, in_stock, shipping (火箭速配), category, variants[] (selectable specs)
- `coupang price <id> <id>... --json`
  quick price check for 1 or more products for price monitoring (price, origin_price, discount, in_stock, checked_at)
- Login: `coupang login` (default `--from-browser`) copies the active session from Edge on CDP 9222.
  `whoami` checks the saved session over the cart API (name, member_srl, cart items). Exit 3 = run `coupang login` again.
- Cart (needs login):
  - `coupang cart list`
  - `coupang cart add <id|url> [--qty N] [--spec TEXT] --yes`
  - `coupang cart qty <line#|id> <N> --yes`
  - `coupang cart rm <line#|id> --yes`
  Always run first WITHOUT `--yes` to get the dry run, show it to the user, and pass `--yes` only after user confirmation.
  The CLI automatically re-reads the cart from the API and verifies that the change landed.
- Safety: Never call checkout, payment, or order submission endpoints. The CLI has no checkout commands by design.
