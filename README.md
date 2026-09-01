# Sales-Operations-Data-Modeling-Power-BI
Built a relational data model in Power BI by transforming multiple raw business tables into a structured analytical model with fact tables, dimensions, bridge tables, and optimized relationships.
# Sales & Operations Data Model

Power BI data modeling, covering customers, orders, sales, inventory, campaigns, and order-to-pay tracking.

## Data Sources

Raw tables imported directly from `dataset.xlsx`: `CUST_MASTER`, `Address`, `customer_contacts`, `user_details`, `products`, `subcategories`, `cities`, `ORDERS_2025`/`ORDERS_2026` (combined into `orders`), `order_line_items`, `shipments`, `INVOICES`, `payments`, `CAMPAIGN_LOG`, `campaign_skus`, `sales_targets`, `security`. A custom `channels` lookup table was also added (not in the original file) to map order channel codes to names.

## Data Model

**Dimension tables:**
- `dim_customer` — customer + contact + address + region, merged from CUST_MASTER, customer_contacts, user_details, Address, and cities
- `dim_product` — product + category, merged from products and subcategories
- `dim_order_flag` — distinct combinations of channel/status/priority, indexed with a surrogate key
- `dim_geo` — city + region, indexed from cities
- `dim_campaign` — one row per campaign, deduplicated from CAMPAIGN_LOG
- `dim_date` — auto-generated calendar table (`CALENDARAUTO()`)

**Fact tables:**
- `fact_sales` — order line items joined to customer, product, order-flag, and ship/bill-to geo keys
- `fact_inventory` — monthly stock levels, unpivoted from the wide `inventory` sheet
- `fact_campaign_spend` — daily campaign performance (impressions, clicks, spend)
- `fact_promotion_coverage` — bridge table linking campaigns to the SKUs they promoted
- `fact_order_prefrence` — order → ship → invoice → payment dates per order, used for cycle-time analysis
- `fact_sales_target` — monthly revenue targets

**Relationships:** all fact tables connect to dimensions as many-to-one on surrogate keys (`customer_id`, `product_key`, `flag_key`, `geo_key`, `campaign_key`), except `fact_sales` which has **two** links to `dim_geo` (ship-to and bill-to city) — only the bill-to one is active, ship-to is inactive to avoid ambiguity. `dim_customer` also links to `security` on `region` for row-level security by region.

## Cleaning Steps (Power Query)

- Combined `ORDERS_2025` and `ORDERS_2026` into one `orders` table
- Filtered out a placeholder customer (`CustomerID = 9999`) and test products (`ZZZ-000` and two dummy `source_id`s) before building dimensions
- Merged customer contacts, user details, and address/city/region into `dim_customer`; kept only the primary contact
- Merged product subcategory → category lookup into `dim_product`
- Built `dim_order_flag` by deduplicating channel/status/priority combinations from orders and joining to a channel-name lookup
- Unpivoted `inventory`'s monthly columns into one row per product per month
- Split `campaign_skus`'s comma-separated SKU list into one row per SKU for the promotion-coverage bridge table
- Built `fact_order_prefrence` by chaining orders → shipments → invoices → payments on their IDs

## Key Measures & Calculated Columns

```
total_sales            = SUM(fact_sales[line_total])
total_active_customer  = DISTINCTCOUNT(fact_sales[customer_id])
total_customer         = COUNT(dim_customer[customer_id])
total_orders           = DISTINCTCOUNT(fact_sales[order_id])
avg_shipping           = AVERAGE(fact_order_prefrence[order_to_pay])

-- calculated column on fact_order_prefrence:
order_to_pay = DATEDIFF(fact_order_prefrence[order_date], fact_order_prefrence[pay_date], DAY)
```


## Repo Structure

```
sales-data-model/
├── README.md
├── data/
│   └── dataset.xlsx
├── powerbi/
│   └── SalesModel.pbip
└── docs/
    └── model-diagram.png
```

## How to Open

1. `git clone https://github.com/YOUR-USERNAME/sales-data-model.git`
2. Open `powerbi/SalesModel.pbip` in Power BI Desktop
3. Refresh data
