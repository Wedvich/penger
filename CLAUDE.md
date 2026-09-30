# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**penger** — skills and agents for interacting with the YNAB (You Need A Budget) API using Claude.

## YNAB API

- **Docs**: https://api.ynab.com
- **Base URL**: `https://api.ynab.com/v1`
- **Auth**: Bearer token in `Authorization` header
- **Rate limit**: 200 requests per hour (rolling window, 429 on exceeded)
- **Currency**: milliunits (1000 = one unit of currency)
- **Dates**: ISO 8601
- **Delta requests**: most endpoints support fetching only changed data

## Authentication

The YNAB API token is cached in the macOS login keychain (service `ynab-access-token`), never on disk.

Always read it inline inside the command that uses it, so it never gets printed to output:

```sh
curl -s -H "Authorization: Bearer $(security find-generic-password -s ynab-access-token -w)" \
  https://api.ynab.com/v1/budgets
```

Never run `security find-generic-password ... -w` on its own — that prints the token.

If the keychain item is missing, ask the user to add it (interactive prompt, so the token stays out of shell history and process args):

```sh
security add-generic-password -a "$USER" -s ynab-access-token -w
```

The source of truth is 1Password: item "YNAB", field "claude token" (`op item get "YNAB" --account my.1password.com --fields "claude token" --reveal`).

## Safety

Always ask for explicit confirmation before any destructive YNAB API operation — this includes updating, moving, or deleting transactions, and changing budgeted amounts.

## Workflows

### Moving transactions between categories

When asked to move transactions from one category to another (e.g., "move all Kiwi transactions from Felles ekstra to Felles mat og drikke"):

1. **Find transactions**: Fetch all transactions in the source category and filter by the requested criteria (payee, date range, memo, amount, etc.).
2. **Show matches**: Display the matching transactions in a table for review.
3. **Confirm**: Ask for explicit confirmation before proceeding (destructive operation).
4. **Move transactions**: Update each transaction's `category_id` using `PUT /budgets/{budget_id}/transactions/{transaction_id}` with `{"transaction": {"category_id": "<target>"}}`. Use a bash array and loop to ensure every transaction is processed. **Split transactions**: The API does not support updating subtransaction categories. Flag these to the user for manual editing in YNAB.
5. **Adjust budgets**: For each affected month, read the current `budgeted` amounts for both source and target categories, then move the sum of the moved transactions' amounts from source to target using `PATCH /budgets/{budget_id}/months/{month}/categories/{category_id}` with `{"category": {"budgeted": <new_amount>}}`. If the source category has less budget than the amount being moved, reduce it to 0 and move whatever it had to the target. The shortfall means the target will be overdrafted for that month, which is the honest state — the original spend was already an overdraft in the source.
6. **Verify**: Confirm all transactions moved and budget totals are correct.
