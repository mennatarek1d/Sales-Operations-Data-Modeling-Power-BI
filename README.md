# Sales-Operations-Data-Modeling-Power-BI
Built a relational data model in Power BI by transforming multiple raw business tables into a structured analytical model with fact tables, dimensions, bridge tables, and optimized relationships.

# Executive Summary

This project transforms 22 disconnected, flat spreadsheet tables into a governed star-schema data model in Power BI, enabling reliable, self-service reporting across sales, customers, marketing, inventory, and order fulfillment.

The model consolidates fragmented customer, product, and order data into six clean dimension tables and six fact tables, replacing brittle text-based joins with surrogate keys and removing duplicate and placeholder records. It supports key business questions out of the box — total and active customers, order volume, revenue, and order-to-payment cycle time — and is structured to extend cleanly to additional metrics (campaign ROI, inventory turnover, delivery performance) without rework.

Row-level security is built in, restricting each user's view to their assigned region. Two documented cleanup items (redundant auto-generated date tables and a couple of naming inconsistencies) are the only outstanding items before this is production-ready.


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


<img width="2360" height="920" alt="model-before-after" src="https://github.com/user-attachments/assets/589ee4f8-b10e-40b5-862a-96291f865c29" />


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
│   └── Sales_Model.xlsx
├── powerbi/
│   └── SalesModel.pbip
```
