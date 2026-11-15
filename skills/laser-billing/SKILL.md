---
name: laser-billing
description: >-
  Organization billing with `org billing`. Reads and changes the billing recipient, shows the payment method and removes the stored card, lists and redeems promo credits, lists invoices and saves their PDFs, reads usage reports per deployment, and lists cloud marketplace agreements.
---

# laser-billing

Organization-wide billing. Adding a card and linking a marketplace purchase run in the browser, so they need the console.

## Verbs

```sh
laser org billing show -o json
laser org billing payment-method -o json
laser org billing credits -o json
laser org billing invoice list -o json
laser org billing invoice list --status paid -o json
laser org billing invoice show 615380456123456841 -o json
laser org billing invoice show 615380456123456841 --pdf invoice.pdf
laser org billing report list -o json
laser org billing report show 615380456123456851 -o json
laser org billing marketplace --cloud gcp -o json
laser org billing update-info --company "Example Inc." --country US --email billing@example.com
laser org billing redeem WELCOME100
laser org billing remove-payment-method --yes
```

- `show` returns the billing recipient and payment method.
- `invoice list` filters with `--status` (`draft`, `pending`, `approved`, `paid`, `failed`, `cancelled`, `refunded` or `partially_refunded`). `invoice show` returns the line items, and `--pdf` also saves the PDF of an approved or paid invoice.
- `report` lists usage reports per deployment with cloud, tier, period and total, and `report show` the compute, storage and network costs.
- `update-info` changes the recipient printed on invoices: `--name`, `--company`, `--address`, `--city`, `--state`, `--postal-code`, `--country` (two letters), `--tax-id`, `--email` and `--phone`. It changes only the flags you pass and needs a name or a company.
- `redeem` turns a promo code into a credit. Invoices consume credits oldest first.
- `marketplace` lists AWS agreements (the default) or GCP entitlements linked to the organization.
- `remove-payment-method` removes the stored card. Paid deployments need a new one before they are created.

Prices of a deployment before it exists come from `deployment create preview` and `pricing` (see `/laser-deploy`).

## Hard rules

1. Billing data is financial and personal. Show totals, periods and statuses. Don't paste line items, addresses or tax ids into the chat unless the user asks for them.
2. `--pdf` writes a file. Confirm the path with the user first.
3. `update-info` changes what appears on every future invoice. Print the command and wait for approval.
4. `remove-payment-method` is destructive: without a terminal the CLI refuses it unless `--yes` is passed. Say that paid deployments cannot be created until a new card is added, ask the user to confirm, and only then add `--yes`.
5. Reading billing needs `billing:read`, changing it `billing:manage`. On a 403, say which grant is missing and don't retry.

## Common asks

- "what plan are we on": `laser org show -o json` for the plan.
- "what did orders cost last month": `report list -o json`, pick the deployment and period, then `report show`.
- "download the latest invoice": `invoice list -o json`, take the newest approved or paid one, confirm the path, then `invoice show <id> --pdf <path>`.
- "do we have credits left": `credits -o json`.

## Don't

- Don't offer commands to add a card, change the organization's subscription or pay an invoice. They don't exist in the CLI. A deployment's own plan changes with `deployment settings plan upgrade` (see `/laser-deploy`).
- Don't save PDFs to a path the user did not choose.
