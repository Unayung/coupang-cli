# coupang-cli

Agent-native CLI for Coupang Taiwan (`www.tw.coupang.com`, TWD, 火箭速配). Zero dependencies (Python 3.8+ stdlib).
Search, product, price checks, login, and the cart. **No checkout.**

Portable: plain HTTPS with browser headers and automatic fallback to a background tab in Edge on CDP (`COUPANG_CDP`, default `127.0.0.1:9222`) when challenged by Akamai Bot Manager.

Measured endpoints (2026-10-29):

| what | endpoint | transport | needs |
|---|---|---|---|
| search | `GET www.tw.coupang.com/np/search?q=&sorter=&page=&minPrice=&maxPrice=` | HTTP / CDP fallback | nothing |
| product | `GET www.tw.coupang.com/products/<id>` (LD+JSON, specs/options table) | CDP background tab | nothing |
| price | Product & search price endpoints (TWD, discount rate, stock) | HTTP / CDP fallback | nothing |
| whoami | `GET cart.tw.coupang.com/api/node/cart/content?isLogin=true` | HTTP with session | login |
| cart list | `GET cart.tw.coupang.com/api/node/cart/content?isLogin=true` | HTTP / CDP fallback | login |
| cart add | `POST www.tw.coupang.com/next-api/order/add-cart` | CDP background tab | login |
| cart qty | `POST cart.tw.coupang.com/api/node/cart/quantity` | HTTP / CDP fallback | login |
| cart rm | `POST cart.tw.coupang.com/api/node/cart/deleteItems` | HTTP / CDP fallback | login |

Tabs in Edge are never focused or disrupted. Background tabs are created with `Target.createTarget(background=True)` and closed immediately after the request finishes.

## Login (no password, ever)

Coupang signs in with a password (+ OTP), so the CLI never takes credentials. It logs in with the
`Cookie` request header of a browser session you already have:

1. In your browser: DevTools > Network > click any request to `www.tw.coupang.com` > Request Headers.
2. Copy the value of `Cookie:` (with or without the leading `Cookie: `).
3. Run `coupang login`. It reads the clipboard, or stdin when piped.

```bash
coupang login                   # Cookie header from stdin (piped) or the clipboard;
                                #   nothing usable there = exit 2 with these steps
coupang login --from-clipboard  # clipboard only
coupang login --cookie-stdin    # paste or pipe Cookie header via stdin
coupang login --keyring         # store in OS keyring (secret-tool / macOS security)
coupang login --from-browser    # opt-in: copy the session from Edge on CDP 9222; if not signed in,
                                #   opens Coupang login in a background tab (--wait 300)
coupang whoami
coupang logout
```

Sessions are verified against the real cart API before saving to `~/.config/coupang/state.json`
(mode 0600, `COUPANG_HOME` moves it) or the OS keyring. The `COUPANG_COOKIE` environment variable
overrides both. Exit 3 means there is no valid session (or it expired): log in again.

Edge on CDP 9222 is needed only for `login --from-browser`, `product`, `cart add`, and as the
fallback when Akamai challenges a plain HTTPS read.

## Commands

### Doctor
```bash
coupang doctor                  # diagnose CDP browser, HTTP reachability, session, cart
```

### Search
```bash
coupang search 衛生紙 [--limit 20] [--page 1] [--sort relevance|price|price-desc|sales|newest]
                     [--min 100 --max 500] [--in-stock]
```

**Caveat (measured 2026-10-30):** search results are cleaned before they reach you:
- Sponsored/ad slots are dropped (they're pinned into the result list regardless of
  sort/page and would otherwise repeat across pages and break price ordering).
- Listings priced exactly `NT$99,999` are dropped (a seller-side placeholder for
  listings that aren't really purchasable - confirmed on the product's own page).
- Coupang's own `price`/`price-desc` sort is only reliable for ~7-8 items per
  request, so the CLI fetches extra pages as needed, dedupes by base product id,
  sorts client-side, and then applies `--page`/`--limit` to that clean pool.
  `--page`/`--limit` are therefore a pure client-side window over this pool, not a
  literal pass-through of Coupang's own `page` query parameter.

**Caveat (added 2026-10-30):** the human-mode (non `--json`) search table shows a short
canonical URL (`https://www.tw.coupang.com/products/<id>`) instead of Coupang's actual
product link, which carries ~400 chars of tracking params (`itemId`, `vendorItemId`,
`sourceType`, `clickEventId`, ...) and used to flood the terminal. `--json` output is
unchanged and still returns the full tracked URL.

### Product
```bash
coupang product <id|url>        # (alias: item) title, brand, price, original price,
                                # discount, rating, reviews, stock, delivery, specs/variants
```

### Price Check
```bash
coupang price 257300782612480 723840334151718    # check prices across multiple products
```

### Raw (Debug)
```bash
coupang raw "https://cart.tw.coupang.com/api/node/cart/content?isLogin=true" --json
```

### Cart (Login Required)
```bash
coupang cart list                               # view current cart items and totals
coupang cart add <id|url> [--qty N] [--spec TEXT] --yes
coupang cart qty <line#|cartItemId> <N> --yes   # update line item quantity
coupang cart rm <line#|cartItemId|id> --yes     # remove item from cart
```

Without `--yes`, write commands run in **dry-run** mode, printing the exact action without modifying the cart. After every write (`add`, `qty`, `rm`), the CLI automatically re-reads the cart from the API and verifies that the change landed.

Every command supports `--json` and returns `{ok, data|error}`.
Exit codes: `0` ok, `1` api error, `2` usage error, `3` auth error, `4` network/blocked error.

## Safety & Disclaimers

**NO checkout / payment / order placement.** The CLI does not implement, and will never call, order completion or payment APIs. Unofficial client for private API; Coupang is a registered trademark of Coupang Corp.
