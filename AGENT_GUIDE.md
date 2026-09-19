# Chase Bank Data via Stripe Financial Connections — Agent Guide

Purpose: query Kiril's Chase account balances and transactions through the Stripe API.
The bank is already connected. No user interaction is needed for reads.

## Auth

All calls use the Stripe live secret key as HTTP Basic auth username (note trailing colon):

```
curl -s "https://api.stripe.com/v1/..." -u "$STRIPE_KEY:"
```

The key is available as the `STRIPE_KEY` env var (same value as `STRIPE_SECRET_KEY` in
the Vercel project `plaid-test`). Never print, log, or echo the key.

## Connected accounts (permanent IDs)

| Account | fca ID | Type |
|---|---|---|
| PREMIER PLUS CKG (...2197) | `fca_1UHWsiFpZ7LrZ3fJxwNjkJ1e` | checking |
| PREMIER SAVINGS (...9368) | `fca_1UHWsiFpZ7LrZ3fJGnLS7MwD` | savings |
| Chase Freedom Unlimited (...7390) | `fca_1UHWsiFpZ7LrZ3fJWfmkF26c` | credit card |
| CHASE AUTO ACCOUNT (...4208) | `fca_1UHWsiFpZ7LrZ3fJ13KAinFc` | auto loan |

Customer wrapper: `cus_VI6uV5wo7slCD7`. To re-list accounts (e.g. if new ones are added):

```
GET /v1/financial_connections/accounts?account_holder[customer]=cus_VI6uV5wo7slCD7
```

## Balances (two-step: refresh, then read)

Balances are NOT live; you must request a refresh first, wait ~5-10s, then read.

```bash
# 1. trigger
curl -s "https://api.stripe.com/v1/financial_connections/accounts/{fca_id}/refresh" \
  -u "$STRIPE_KEY:" -d "features[]=balance"

# 2. read (poll until balance_refresh.status == "succeeded")
curl -s "https://api.stripe.com/v1/financial_connections/accounts/{fca_id}" -u "$STRIPE_KEY:"
```

- Balance lives at `balance.current.usd`, in **cents**. Cash accounts also have
  `balance.cash.available.usd`.
- Refreshes are rate-limited; if `next_refresh_available_at` is in the future, read the
  cached balance instead of re-triggering.
- Credit accounts (Freedom Unlimited, auto loan): `balance.current` = amount owed.

## Transactions

Accounts are subscribed to the transactions feed; history updates automatically
(a few times/day). No refresh call needed for normal use.

```bash
curl -s "https://api.stripe.com/v1/financial_connections/transactions?account={fca_id}&limit=100" \
  -u "$STRIPE_KEY:"
```

- Amounts in **cents**; sign convention: negative = money out, positive = money in.
- Key fields: `description`, `amount`, `transacted_at` (unix), `status`
  (`posted` | `pending` | `void`).
- Paginate with `starting_after={last_transaction_id}`; `has_more` tells you when to stop.
- There is no category field. Categorize from `description` yourself.
- To force-refresh transactions (rarely needed): same `/refresh` endpoint with
  `features[]=transactions`, subject to the same rate limit.

## Gotchas

- ID prefixes: `fca_` = bank account (what you query), `cus_` = Stripe customer,
  `fctxn_` = transaction. Balance/transaction endpoints only accept `fca_` IDs.
- All amounts are integer cents. Divide by 100.
- If an account shows `status: "inactive"` or API returns permission errors, the bank
  connection needs re-auth: Kiril must reconnect via the site (Vercel project
  `plaid-test`, repo github.com/kirilQC/plaid-test) using "Connect with Stripe".
- The site also has an unused Plaid path; ignore it. Stripe is the live provider.

## Quick recipes

Total cash position (checking + savings), after refreshing both:

```
(balance.current.usd of fca_...xwNjkJ1e + fca_...GnLS7MwD) / 100
```

Net worth snapshot: cash accounts minus credit balances (Freedom + auto loan).
