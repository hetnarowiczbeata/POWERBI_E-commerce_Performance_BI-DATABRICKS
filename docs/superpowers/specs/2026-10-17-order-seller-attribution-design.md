# Order Seller Attribution Design

## Goal

Allow a seller selection to filter order-level delivery metrics by assigning
each order to one seller represented in its item rows.

## Current context

- `fct_orders` is an order-grain fact loaded from Databricks through Power Query.
- `fct_items` is an item-grain fact with `order_id` and text `seller_id`.
- `dim_sellers` already filters `fct_items`.
- `fct_orders_full` is a calculated one-row-per-order hub connected to the
  facts. `dim_sellers` must not be connected to this hub for attribution:
  filtering through it would also restrict item rows by the chosen order
  attribution, not only by the actual seller on each item.
- The `fct_orders` Power Query partition already navigates to `fct_items` and
  creates a distinct order-ID table that is currently unused.
- Model and report files already have user changes. Implementation must preserve
  all unrelated edits and integrate only the targeted query and relationship
  changes.

## Approved behavior

Assign each order to exactly one seller: the lowest `seller_id` by
case-sensitive ordinal text comparison among its item rows. Do not trim or
otherwise rewrite nonblank seller IDs. This is a deliberate attribution rule,
not a claim that the selected seller is the only seller on that order. A seller
selection will include order-level metrics only for orders attributed to that
seller; other sellers on a multi-seller order will not receive those order
metrics.

## Design

In the `fct_orders` Power Query partition:

1. Exclude item rows with null or blank/whitespace `order_id` from the lookup
   grouping; keep those rows in `fct_items`.
2. Group the remaining `fct_items` by `order_id`.
3. Select the lowest non-null, nonblank `seller_id` per group using an explicit
   case-sensitive ordinal comparer, independent of input row order.
4. Left-outer join this one-row-per-order result to `fct_orders` by `order_id`.
5. Expand the seller value as `attributed_seller_id`.

Add a single-direction relationship from the unique `dim_sellers[seller_id]`
to `fct_orders[attributed_seller_id]`, active and filtering from
`dim_sellers` to `fct_orders`. This directly filters order-grain delivery
measures. Keep the existing single-direction `dim_sellers`-to-`fct_items`
relationship so item-level seller metrics continue to reflect the actual
seller on each item.

`fct_orders_full` remains a shared order hub that filters its connected facts
in the existing direction. Seller filters must not flow backward from either
fact into the hub, and must not reach `fct_items` through an attributed-order
path. Leave all `fct_orders_full` relationships unchanged. Do not add a seller
relationship to `fct_orders_full`, change customer relationship direction, or
alter Databricks source tables.

## Edge cases and data behavior

- Orders with no matching item rows remain in `fct_orders`; their attributed
  seller is blank and they are not selected by a seller filter.
- Null, empty, and whitespace-only seller IDs are ignored. If every item seller
  ID for an order is ignored, the attributed seller is blank.
- An attributed seller ID absent from `dim_sellers` cannot be selected through
  that dimension; validation must identify such orphan seller IDs and fail
  rather than treating them as successfully attributed.
- `fct_orders` rows with null or blank/whitespace order IDs remain in the table
  with a blank attributed seller.
- The grouped table must have at most one row per order before joining, so the
  join cannot duplicate order-grain rows.
- The existing order key and seller key types must remain compatible.

## Validation

- Check the Power Query expression for a one-row-per-order grouped result,
  left-outer join, and expected expanded field.
- Check that the relationship uses `dim_sellers` as the one side and
  `fct_orders` as the many side, with active single-direction filtering.
- Check that the existing seller-to-items relationship and all
  `fct_orders_full` relationships remain unchanged; there must be no seller
  relationship to `fct_orders_full`.
- Verify all relevant filter paths are single-direction from dimensions to
  facts and there is no active ambiguous route from `dim_sellers` to
  `fct_items`.
- Validate that the TMDL parses, `fct_orders` has no duplicate nonblank
  `order_id` values, row count is unchanged, and order-key coverage is unchanged
  after refresh.
- Verify attributed, unmatched, all-null/blank-seller, and null/blank-order-key
  cases with expected blank/nonblank attribution results.
- Verify seller IDs that differ only by case are ordered by the explicit
  ordinal comparer consistently regardless of source-row order, and fail
  validation if an attributed seller is absent from `dim_sellers`.
- In Power BI, verify a seller slicer filters order-level delivery measures
  only for orders attributed to the selected seller, while item-level measures
  still reflect the actual seller on each item. For a multi-seller order,
  assert that exactly its case-sensitive ordinal-lowest seller receives the
  order-level metric.
- Report clearly if the local model cannot be refreshed or queried for runtime
  validation.
