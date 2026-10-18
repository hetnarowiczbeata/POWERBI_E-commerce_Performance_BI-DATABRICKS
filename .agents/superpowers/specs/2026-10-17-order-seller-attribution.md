# Order Seller Attribution Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Attribute each order to its case-sensitive ordinal-lowest nonblank seller ID so seller selections filter order-level delivery measures without changing which seller owns each item.

**Architecture:** Aggregate `fct_items` to one attributed seller per order in the existing `fct_orders` Power Query partition, left-join that result to the order-grain fact, and expose it as a text column. Add one active single-direction relationship from `dim_sellers` to `fct_orders`; preserve the actual seller-to-item relationship and leave `fct_orders_full` unchanged.

**Tech Stack:** Power BI PBIP/TMDL, Power Query M, DAX model validation, Git.

---

## File map

- Modify `E-commerce Performance Report.SemanticModel/definition/tables/fct_orders.tmdl`: declare `attributed_seller_id` and update only the `fct_orders` Power Query partition. This file remains the order-grain fact.
- Modify `E-commerce Performance Report.SemanticModel/definition/relationships.tmdl`: add an active single-direction relationship from `dim_sellers[seller_id]` to `fct_orders[attributed_seller_id]`.
- Do not modify `fct_orders_full.tmdl`, existing item relationships, other user-edited report files, or Databricks objects.

## Chunk 1: Attribute sellers to orders

### Task 1: Capture baseline and define acceptance checks

**Files:** No persistent test files; use Power BI model queries against the current model and temporary validation queries.

If the local model cannot be queried or refreshed, mark all model-dependent checks in this task and Tasks 2–3 as blocked; still run the standalone Power Query synthetic check and repository checks.

- [ ] **Step 1: Record the order-grain baseline**
  Run this DAX query against the connected model:
  ```DAX
  EVALUATE
  ROW(
      "Rows", COUNTROWS(fct_orders),
      "DistinctNonblankOrderIds",
          COUNTROWS(
              FILTER(
                  VALUES(fct_orders[order_id]),
                  NOT ISBLANK(fct_orders[order_id])
                      && LEN(TRIM(fct_orders[order_id])) > 0
              )
          ),
      "BlankOrWhitespaceOrderRows",
          COUNTROWS(
              FILTER(
                  fct_orders,
                  ISBLANK(fct_orders[order_id])
                      || LEN(TRIM(fct_orders[order_id])) = 0
              )
          )
  )
  ```
  Expected: One result row with three counts; record them for comparison after refresh.
- [ ] **Step 1a: Check order-key uniqueness**
  Run:
  ```DAX
  EVALUATE
  FILTER(
      SUMMARIZE(
          FILTER(
              fct_orders,
              NOT ISBLANK(fct_orders[order_id])
                  && LEN(TRIM(fct_orders[order_id])) > 0
          ),
          fct_orders[order_id],
          "RowsPerOrder", COUNTROWS(fct_orders)
      ),
      [RowsPerOrder] > 1
  )
  ```
  Expected: No rows. If the source order fact is not unique by nonblank `order_id`, stop; the seller relationship cannot be made from this fact at order grain without resolving that data issue.
- [ ] **Step 2: Record the seller-key integrity baseline**
  Run:
  ```DAX
  EVALUATE
  ROW(
      "SellerDimensionRows", COUNTROWS(dim_sellers),
      "DistinctNonblankSellerIds",
          COUNTROWS(
              FILTER(
                  VALUES(dim_sellers[seller_id]),
                  NOT ISBLANK(dim_sellers[seller_id])
                      && LEN(TRIM(dim_sellers[seller_id])) > 0
              )
          ),
      "BlankOrWhitespaceSellerRows",
          COUNTROWS(
              FILTER(
                  dim_sellers,
                  ISBLANK(dim_sellers[seller_id])
                      || LEN(TRIM(dim_sellers[seller_id])) = 0
              )
          )
  )
  ```
  Also run `EVALUATE FILTER(SUMMARIZE(dim_sellers, dim_sellers[seller_id], "Rows", COUNTROWS(dim_sellers)), [Rows] > 1)`.
  Expected: One summary row; duplicate-key query returns no rows. Record counts. If the model is unavailable, mark runtime baselines as blocked and continue with static implementation checks; do not claim those baselines passed.
- [ ] **Step 3: Confirm the regression is reproducible**
  Attempt a DAX model query referencing `fct_orders[attributed_seller_id]` before it exists.
  Expected: The query fails because the attribution column is absent, confirming the check exercises the new behavior rather than existing model state.

### Task 2: Add seller attribution to the order-grain fact

**Files:**
- Modify `E-commerce Performance Report.SemanticModel/definition/tables/fct_orders.tmdl`

- [ ] **Step 0: Snapshot the target file's current state**
  Run `git diff -- "E-commerce Performance Report.SemanticModel/definition/tables/fct_orders.tmdl"` and read the full current `fct_orders` table and partition definition.
  Expected: Record any pre-existing edits and preserve them; edit only the `fct_orders` column list and the relevant Power Query steps.
- [ ] **Step 1: Implement the grouped order-to-seller query**
  In the existing Power Query `let` expression, replace the unused
  `fct_items_OrderIds` step with these steps. Keep the existing Databricks
  navigation and date type conversions:
  ```powerquery
  fct_items_OrderSellers =
      Table.Group(
          Table.SelectRows(
              fct_items_Table,
              each [order_id] <> null and Text.Trim([order_id]) <> ""
          ),
          {"order_id"},
          {
              {
                  "attributed_seller_id",
                  each
                      List.First(
                          List.Sort(
                              List.Select(
                                  List.RemoveNulls([seller_id]),
                                  each Text.Trim(_) <> ""
                              ),
                              Comparer.Ordinal
                          ),
                          null
                      ),
                  type nullable text
              }
          }
      ),
  fct_orders_JoinedSellers =
      Table.NestedJoin(
          fct_orders_Table,
          {"order_id"},
          fct_items_OrderSellers,
          {"order_id"},
          "order_seller",
          JoinKind.LeftOuter
      ),
  fct_orders_WithSeller =
      Table.ExpandTableColumn(
          fct_orders_JoinedSellers,
          "order_seller",
          {"attributed_seller_id"},
          {"attributed_seller_id"}
      ),
  #"Zmieniono typ kolumny" =
      Table.TransformColumnTypes(
          fct_orders_WithSeller,
          {
              {"order_purchase_timestamp", type date},
              {"order_approved_at", type date},
              {"order_delivered_carrier_date", type date},
              {"order_delivered_customer_date", type date},
              {"order_estimated_delivery_date", type date}
          }
      )
  ```
  Expected: The grouped lookup has one row per valid order ID; the left join keeps every `fct_orders_Table` row.
- [ ] **Step 2: Declare the new TMDL column**
  Add `column attributed_seller_id` with `dataType: string`, `summarizeBy: none`, `sourceColumn: attributed_seller_id`, and a new unique lineage tag. Do not add a calculated DAX column.
- [ ] **Step 3: Refresh and run the attribution regression query**
  Refresh the local semantic model and query `fct_orders[order_id]` with `fct_orders[attributed_seller_id]`.
  Expected: The query succeeds; each order has at most one attribution; multi-seller orders show the case-sensitive ordinal-lowest nonblank seller ID; orders with no valid matching item seller show blank.
- [ ] **Step 4: Verify order grain and key coverage**
  Re-run the baseline counts and duplicate-key query. In Power Query, run a temporary diagnostic using the exact `fct_orders_Table` and `fct_orders_WithSeller` steps:
  ```powerquery
  let
      Missing = Table.NestedJoin(
          fct_orders_Table, {"order_id"},
          fct_orders_WithSeller, {"order_id"},
          "match", JoinKind.LeftAnti
      ),
      Extra = Table.NestedJoin(
          fct_orders_WithSeller, {"order_id"},
          fct_orders_Table, {"order_id"},
          "match", JoinKind.LeftAnti
      )
  in
      #table(
          {"MissingOrderRows", "ExtraOrderRows", "OriginalRows", "JoinedRows"},
          {{
              Table.RowCount(Missing),
              Table.RowCount(Extra),
              Table.RowCount(fct_orders_Table),
              Table.RowCount(fct_orders_WithSeller)
          }}
      )
  ```
  Expected: Both anti-join counts are zero and original/joined row counts match. Also confirm `order_id` is text on both sides of the join.
- [ ] **Step 5: Verify data edge cases**
  In Power Query, evaluate this temporary synthetic test. It runs the grouping against both source row orders and left-joins the result to orders with unmatched, null, and blank keys:
  ```powerquery
  let
      Sample = #table(
          type table [order_id = nullable text, seller_id = nullable text],
          {
              {"O1", "a"},
              {"O1", "A"},
              {"O1", "b"},
              {"O2", null},
              {"O2", " "},
              {null, "C"},
              {"O3", " "}
          }
      ),
      AttributeSellers = (Items as table) as table =>
          Table.Group(
              Table.SelectRows(
                  Items,
                  each [order_id] <> null and Text.Trim([order_id]) <> ""
              ),
              {"order_id"},
              {
                  {
                      "attributed_seller_id",
                      each List.First(
                          List.Sort(
                              List.Select(
                                  List.RemoveNulls([seller_id]),
                                  each Text.Trim(_) <> ""
                              ),
                              Comparer.Ordinal
                          ),
                          null
                      ),
                      type nullable text
                  }
              }
          ),
      Forward = Table.Sort(AttributeSellers(Sample), {{"order_id", Order.Ascending}}),
      Reversed = Table.Sort(
          AttributeSellers(Table.ReverseRows(Sample)),
          {{"order_id", Order.Ascending}}
      ),
      Orders = #table(
          type table [order_id = nullable text],
          {{"O1"}, {"O2"}, {"O3"}, {"O4"}, {null}, {" "}}
      ),
      Joined = Table.NestedJoin(
          Orders,
          {"order_id"},
          Forward,
          {"order_id"},
          "order_seller",
          JoinKind.LeftOuter
      ),
      Expanded = Table.ExpandTableColumn(
          Joined,
          "order_seller",
          {"attributed_seller_id"},
          {"attributed_seller_id"}
      ),
      Result = [
          ForwardOrderSeller = Table.ToRows(Forward),
          ReversedOrderSeller = Table.ToRows(Reversed),
          OrdersAfterLeftJoin = Table.ToRows(Expanded)
      ]
  in
      Result
  ```
  Expected: `ForwardOrderSeller` equals `ReversedOrderSeller`; `O1` maps to `"A"`; `O2` and `O3` have blank attribution; and the left join keeps all six rows, with `O4`, null order ID, and whitespace order ID also blank. Delete the temporary query after the check.
- [ ] **Step 6: Verify seller referential integrity**
  In a temporary Power Query diagnostic, load the seller IDs from `dim_sellers` into `SellerIds`, and evaluate:
  ```powerquery
  Table.SelectRows(
      fct_orders_WithSeller,
      each
          [attributed_seller_id] <> null
              and Text.Trim([attributed_seller_id]) <> ""
              and not List.Contains(
                  SellerIds,
                  [attributed_seller_id],
                  Comparer.Ordinal
              )
  )
  ```
  Expected: Zero rows. Use exact ordinal comparison to detect case-sensitive orphans that a model-collation DAX comparison may not reveal. If any are present, stop and report them instead of claiming seller slicer coverage is complete.

### Task 3: Connect seller selections to order-level measures

**Files:**
- Modify `E-commerce Performance Report.SemanticModel/definition/relationships.tmdl`

- [ ] **Step 1: Add the seller-to-orders relationship**
  Add an active relationship with `fromColumn: dim_sellers.seller_id` and `toColumn: fct_orders.attributed_seller_id`, using single-direction filtering from the seller dimension to the order fact.
- [ ] **Step 2: Validate the model relationship graph**
  Refresh or validate the local model. Confirm `dim_sellers[seller_id]` is unique, the new relationship is one-to-many and active, and no seller relationship was added to `fct_orders_full`.
  Expected: Model validation succeeds without ambiguous active paths; existing seller-to-items and all `fct_orders_full` relationships remain unchanged.
- [ ] **Step 3: Verify seller slicer behavior**
  First identify multi-seller examples:
  ```DAX
  EVALUATE
  TOPN(
      20,
      FILTER(
          ADDCOLUMNS(
              VALUES(fct_items[order_id]),
              "SellerCount",
                  CALCULATE(
                      COUNTROWS(
                          FILTER(
                              VALUES(fct_items[seller_id]),
                              NOT ISBLANK(fct_items[seller_id])
                                  && LEN(TRIM(fct_items[seller_id])) > 0
                          )
                      )
                  ),
              "AttributedSeller",
                  LOOKUPVALUE(
                      fct_orders[attributed_seller_id],
                      fct_orders[order_id], fct_items[order_id]
                  )
          ),
          [SellerCount] > 1
      ),
      fct_items[order_id], ASC
  )
  ```
  Choose one result and verify the attributed ID against the sample's ordinal-sorted seller IDs. Then run:
  ```DAX
  EVALUATE
  SUMMARIZECOLUMNS(
      dim_sellers[seller_id],
      "AttributedOrders", DISTINCTCOUNT(fct_orders[order_id]),
      "DeliveryRate", [Delivery Rate],
      "AvgDeliveryDays", [Avg Total Delivery Time],
      "ActualSellerItems", COUNTROWS(fct_items)
  )
  ```
  Expected: For each seller, `AttributedOrders`, `DeliveryRate`, and `AvgDeliveryDays` are evaluated for only orders attributed to that seller; `ActualSellerItems` reflects rows directly owned by that seller. Confirm the example multi-seller order contributes to order-level delivery measures only under its attributed seller.
- [ ] **Step 4: Run repository checks**
  Run `git diff --check` and inspect the targeted TMDL diff.
  Expected: No whitespace errors; only the two planned model files are changed by this implementation, in addition to pre-existing user edits.

## Runtime validation blocker

If the local model cannot be connected or refreshed, proceed with the agreed static implementation, Power Query synthetic checks, and repository diff checks. Mark every model-dependent DAX baseline, pre-change regression query, post-refresh grain/key/orphan query, and Power BI relationship/slicer check as blocked; do not claim the Power Query refresh or slicer behavior passed. Leave the model edits intact for user validation in Power BI and report exactly which runtime checks remain.
