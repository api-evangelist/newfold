---
name: newfold-woocommerce-store-operations
description: Manage a WooCommerce catalog and read store reports on a Newfold-hosted
  WordPress site through the BLU MCP gateway, across both the Bluehost wrapper
  abilities and the WooCommerce-native ones.
generated: '2026-08-13'
method: generated
source: https://github.com/newfold-labs/wp-module-mcp/blob/main/README.md
api: newfold:blu-mcp
endpoint: https://{site}/wp-json/blu/mcp
operations:
  - blu-call-ability
  - blu-list-abilities
  - blu-wc-products-search
  - blu-wc-get-product
  - blu-wc-add-product
  - blu-wc-update-product
  - blu-wc-delete-product
  - blu-wc-list-product-categories
  - blu-wc-add-product-category
  - blu-wc-list-product-tags
  - blu-wc-list-product-brands
  - blu-wc-orders-search
  - blu-wc-reports-sales
  - blu-wc-reports-orders-totals
  - blu-wc-reports-products-totals
  - blu-wc-reports-customers-totals
  - blu-wc-reports-coupons-totals
  - blu-wc-reports-reviews-totals
---

# Run a WooCommerce store through BLU MCP

WooCommerce abilities appear only when WooCommerce is active on the site.

## Two surfaces, not one

This is the part that surprises integrators. The gateway whitelists **both**:

- **Bluehost wrappers** — `blu/wc-*`, MCP form `blu-wc-*`. Isolate with
  `name_prefix: "blu-wc-"`.
- **WooCommerce-native abilities** — `woocommerce/<resource>-<op>`, MCP form
  `woocommerce-<resource>-<op>`, registered by WooCommerce 10.3+ through its own
  Abilities API integration. Isolate with `name_prefix: "woocommerce-"`.

They overlap on products and orders. Decide once per task which surface you are
on and stay there; do not mix wrapper and native abilities inside one flow.
List both prefixes before starting so you know what this particular site has.

## Catalog

- Products: `blu-wc-products-search` → `blu-wc-get-product` →
  `blu-wc-add-product` / `blu-wc-update-product` / `blu-wc-delete-product`.
- Categories: `blu-wc-list-product-categories`, `blu-wc-add-product-category`,
  `blu-wc-update-product-category`, `blu-wc-delete-product-category`.
- Tags: the matching `blu-wc-*-product-tag` set.
- Brands: the matching `blu-wc-*-product-brand` set.

Create taxonomy terms before referencing them on a product.

## Orders and reporting

`blu-wc-orders-search` lists orders. The report abilities are read-only rollups:
`blu-wc-reports-sales`, `-orders-totals`, `-products-totals`,
`-customers-totals`, `-coupons-totals`, `-reviews-totals`. Prefer a report over
paginating orders when the question is "how much" rather than "which one".

## Credentials for reports

If you are reaching the site through `@newfold/wp-mcp-connector`, the WooCommerce
report tools automatically switch to `WOO_CUSTOMER_KEY` / `WOO_CUSTOMER_SECRET`
instead of the primary basic-auth credentials. Those two variables must be set,
or report calls fail while every other ability keeps working — which reads like a
permissions bug and is not one.

## Rules that bite

- **No idempotency.** A retried `blu-wc-add-product` creates a duplicate SKU.
  Search first.
- **Deletes are permanent.** Confirm before `blu-wc-delete-product`.
- **Read `statusCode` in the ability wrapper**, not the HTTP status, and treat
  `result.isError === true` as a failure even though the JSON-RPC call succeeded.
- Prefer `name_prefix` + `search` on every discovery call; the catalog is large
  enough that an unfiltered list is a real token cost.
