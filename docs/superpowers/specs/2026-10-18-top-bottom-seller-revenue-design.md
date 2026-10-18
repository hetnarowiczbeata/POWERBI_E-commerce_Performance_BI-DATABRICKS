# Top and Bottom Seller Revenue Measures

## Goal

Add two reusable DAX measures that identify seller revenue for the five highest- and lowest-revenue sellers. The measures are intended for seller-level visuals, not aggregate KPI cards.

## Existing model context

- `[Total Revenue]` is defined as `SUM(fct_items[price])`.
- Seller identity is `dim_sellers[seller_id]`, related to seller item rows.
- Related measures use the `Seller Performance` display folder.

## Design

Add two measures to the existing semantic-model table named `*Miary` (declared as `table '*Miary'` in TMDL):

- `Top 5 Sellers Revenue`
- `Bottom 5 Sellers Revenue`

For each seller in the visual context, each measure returns `[Total Revenue]` if that seller belongs to the corresponding five-seller ranking; otherwise it returns `BLANK()`. This makes the measures suitable for seller-level charts and tables while hiding out-of-rank sellers.

The ranking is recalculated in the current report filter context. `ALLSELECTED(dim_sellers[seller_id])` removes the current visual row's seller ID while respecting external seller selections and other report filters, such as date, state, and product category. Other seller-column filters remain active, so the candidate set is additionally constrained by them.

The exact ranking table for the top measure is:

```DAX
VAR RankedSellers =
    TOPN(
        5,
        FILTER(
            ALLSELECTED(dim_sellers[seller_id]),
            NOT ISBLANK(dim_sellers[seller_id])
                && NOT ISBLANK(CALCULATE([Total Revenue]))
        ),
        CALCULATE([Total Revenue]), DESC,
        dim_sellers[seller_id], ASC
    )
```

For the bottom measure, use the same table expression but order revenue `ASC` and seller ID `ASC`. Each measure captures `SELECTEDVALUE(dim_sellers[seller_id])` and returns `[Total Revenue]` only when `CONTAINS(RankedSellers, dim_sellers[seller_id], CurrentSeller)` is true and a single current seller is in scope; otherwise it returns `BLANK()`. The unique seller ID as secondary ordering resolves revenue ties deterministically. Sellers whose revenue is blank are excluded; sellers whose revenue is zero are included. If fewer than five eligible sellers exist, all eligible sellers are returned. Duplicate IDs are represented once by the dimension; blank IDs are not included in the ranked results.

Both measures will use the existing `Seller Performance` display folder and format values consistently with `[Total Revenue]`.

## Alternatives considered

1. **Per-seller ranked revenue (selected):** useful for seller-level visuals; returns blank outside the top/bottom five.
2. **Aggregate top/bottom five revenue:** useful for cards, but does not show which sellers make up the total.
3. **Include ties:** can return more than five sellers, which conflicts with an exact five-seller request.

## Validation

- Validate both DAX expressions against the model.
- Confirm the measures appear in `*Miary` under `Seller Performance`.
- With at least five sellers having nonblank revenue, a seller-level table must show nonblank values for exactly five sellers per measure; all other sellers must be blank.
- Top results must be ordered by revenue descending, bottom results by revenue ascending; equal revenues are ordered by `seller_id` ascending and still produce no more than five sellers.
- With fewer than five eligible sellers, all eligible sellers are returned; sellers with blank revenue are excluded, and sellers with zero revenue remain eligible.
- Under date, state, product-category, and external seller selections, ranking must be recalculated using revenue in that selected context.
- Without a single seller in the current row context, both measures return blank, avoiding an aggregate being mistaken for a seller-ranked value.

## Scope

This change adds only two measures. It does not change `[Total Revenue]`, relationships, report visuals, or existing measure behavior.
