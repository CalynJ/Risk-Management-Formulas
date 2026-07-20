-- ============================================================================
-- Risk Management Formulas — Applied SQL
-- ============================================================================
-- Demonstrates portfolio risk formulas (see README.md) implemented against
-- real invoice-level accounts receivable data (invoice_data.csv, 50,000 rows).
--
-- Source table: invoices (loaded from invoice_data.csv)
-- Key columns and their ACTUAL stored formats (verified against real data):
--   invoice_id             -- unique invoice identifier
--   cust_number            -- customer identifier
--   name_customer          -- customer name (see data quality note below)
--   posting_date           -- text, format M/D/YYYY (used as invoice date)
--   due_in_date            -- integer, format YYYYMMDD (not a native date type)
--   clear_date             -- text, format M/D/YYYY H:MM, NULL if still open
--   total_open_amount      -- invoice amount
--   isOpen                 -- 1 = still open/unpaid, 0 = closed/paid
--
-- IMPORTANT — dialect note: due_in_date, posting_date, and clear_date are
-- NOT native date columns in the raw CSV. The CAST/PARSE expressions below
-- assume the data has been loaded with posting_date and clear_date parsed
-- using format 'M/D/YYYY' (or 'M/D/YYYY H:MM' for clear_date), and
-- due_in_date parsed from an 8-digit integer using format 'YYYYMMDD'.
-- Adjust the parsing functions to match your specific SQL engine
-- (e.g., TO_DATE in Snowflake/Postgres, PARSE_DATE in BigQuery,
-- STR_TO_DATE in MySQL).
--
-- IMPORTANT — reference date note: this dataset covers Dec 2018–May 2020.
-- Queries below use MAX(clear_date) from the dataset itself as the
-- "as of" reference date for open invoices, rather than the real current
-- date, since comparing 2019/2020 data against today's date would make
-- every open invoice look thousands of days overdue.
--
-- DATA QUALITY NOTE: the same real-world customer (e.g., Walmart) appears
-- under multiple name_customer variants ("WAL-MAR", "WAL-MAR llc",
-- "WAL-MAR co", "WAL-MAR corp", "WAL-MAR corporation"). This is a classic
-- entity-resolution issue — grouping by cust_number rather than name alone
-- avoids fragmenting true customer exposure, but a real risk process would
-- flag and reconcile these variants rather than treat them as five
-- different customers.
--
-- Note: this dataset does not include fee-level detail (discount/schedule/
-- misc fees), so ROI and Yield formulas from the README are not calculable
-- here and are intentionally omitted. See README for details.
-- ============================================================================


-- ----------------------------------------------------------------------------
-- 1. Days to Pay (DTP) and Aging Buckets
-- ----------------------------------------------------------------------------
-- Core collections risk metric: how long each invoice took (or is taking)
-- to clear, bucketed into standard AR risk tiers.

WITH parsed AS (
    SELECT
        invoice_id,
        cust_number,
        name_customer,
        CAST(posting_date AS DATE) AS posting_date,          -- parsed from M/D/YYYY
        CAST(due_in_date AS DATE) AS due_in_date,             -- parsed from YYYYMMDD
        CAST(clear_date AS DATE) AS clear_date,               -- parsed from M/D/YYYY H:MM, NULL if open
        total_open_amount,
        isOpen
    FROM invoices
),
reference_date AS (
    SELECT MAX(clear_date) AS as_of_date FROM parsed
),
invoice_aging AS (
    SELECT
        p.*,
        DATEDIFF(day, p.posting_date, COALESCE(p.clear_date, r.as_of_date)) AS days_to_pay,
        CASE
            WHEN p.isOpen = 1 AND r.as_of_date > p.due_in_date THEN 'Past Due - Open'
            WHEN p.isOpen = 1 THEN 'Open - Current'
            WHEN DATEDIFF(day, p.posting_date, p.clear_date) <= 30 THEN '0-30 Days'
            WHEN DATEDIFF(day, p.posting_date, p.clear_date) <= 60 THEN '31-60 Days'
            WHEN DATEDIFF(day, p.posting_date, p.clear_date) <= 90 THEN '61-90 Days'
            ELSE '90+ Days'
        END AS aging_bucket
    FROM parsed p
    CROSS JOIN reference_date r
)
SELECT
    aging_bucket,
    COUNT(*) AS invoice_count,
    SUM(total_open_amount) AS total_exposure,
    ROUND(AVG(days_to_pay), 1) AS avg_days_to_pay
FROM invoice_aging
GROUP BY aging_bucket
ORDER BY
    CASE aging_bucket
        WHEN '0-30 Days' THEN 1
        WHEN '31-60 Days' THEN 2
        WHEN '61-90 Days' THEN 3
        WHEN '90+ Days' THEN 4
        WHEN 'Open - Current' THEN 5
        WHEN 'Past Due - Open' THEN 6
    END;

-- Verified output against real data:
-- 0-30 Days:        36,472 invoices | $1.20B exposure | 14.7 avg days
-- 31-60 Days:         2,375 invoices | $64.8M exposure | 40.8 avg days
-- 61-90 Days:           916 invoices | $18.6M exposure | 70.3 avg days
-- 90+ Days:             231 invoices | $3.3M exposure  | 112.4 avg days
-- Past Due - Open:   10,000 invoices | $334.2M exposure (as of dataset's last date)


-- ----------------------------------------------------------------------------
-- 2. Customer Concentration / Loan Ratio (window function: RANK)
-- ----------------------------------------------------------------------------
-- Loan Ratio = Customer Total AR / Portfolio Total AR * 100
-- Flags concentration risk: a small number of customers carrying a
-- disproportionate share of total receivables is an early warning sign.
-- Grouped by cust_number (not name alone) to avoid fragmenting exposure
-- across inconsistent name variants of the same real-world customer.

WITH customer_exposure AS (
    SELECT
        cust_number,
        MAX(name_customer) AS name_customer,   -- representative name for display only
        SUM(total_open_amount) AS total_ar,
        SUM(CASE WHEN isOpen = 1 THEN total_open_amount ELSE 0 END) AS open_ar,
        COUNT(*) AS invoice_count
    FROM invoices
    GROUP BY cust_number
),
portfolio_total AS (
    SELECT SUM(total_ar) AS portfolio_total_ar FROM customer_exposure
)
SELECT
    c.cust_number,
    c.name_customer,
    c.total_ar,
    c.open_ar,
    c.invoice_count,
    ROUND((c.total_ar / p.portfolio_total_ar) * 100, 2) AS loan_ratio_pct,
    RANK() OVER (ORDER BY c.total_ar DESC) AS exposure_rank
FROM customer_exposure c
CROSS JOIN portfolio_total p
ORDER BY exposure_rank
LIMIT 20;


-- ----------------------------------------------------------------------------
-- 3. Monthly Portfolio Turnover (CTE chain + running balance)
-- ----------------------------------------------------------------------------
-- Monthly Portfolio Turnover =
--     (Invoices Purchased - Invoices Paid) / ((Starting Balance + Ending Balance) / 2)
-- "Purchased" = invoiced this month, "Paid" = cleared this month.

WITH monthly_invoiced AS (
    SELECT
        DATE_TRUNC('month', CAST(posting_date AS DATE)) AS month,
        SUM(total_open_amount) AS invoices_purchased
    FROM invoices
    GROUP BY DATE_TRUNC('month', CAST(posting_date AS DATE))
),
monthly_paid AS (
    SELECT
        DATE_TRUNC('month', CAST(clear_date AS DATE)) AS month,
        SUM(total_open_amount) AS invoices_paid
    FROM invoices
    WHERE clear_date IS NOT NULL
    GROUP BY DATE_TRUNC('month', CAST(clear_date AS DATE))
),
monthly_activity AS (
    SELECT
        COALESCE(i.month, p.month) AS month,
        COALESCE(i.invoices_purchased, 0) AS invoices_purchased,
        COALESCE(p.invoices_paid, 0) AS invoices_paid
    FROM monthly_invoiced i
    FULL OUTER JOIN monthly_paid p ON i.month = p.month
),
monthly_balance AS (
    SELECT
        month,
        invoices_purchased,
        invoices_paid,
        SUM(invoices_purchased - invoices_paid)
            OVER (ORDER BY month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
            AS ending_balance,
        LAG(SUM(invoices_purchased - invoices_paid)
            OVER (ORDER BY month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW))
            OVER (ORDER BY month) AS starting_balance
    FROM monthly_activity
)
SELECT
    month,
    invoices_purchased,
    invoices_paid,
    starting_balance,
    ending_balance,
    ROUND(
        (invoices_purchased - invoices_paid)
        / NULLIF((COALESCE(starting_balance, 0) + ending_balance) / 2.0, 0),
        4
    ) AS monthly_portfolio_turnover
FROM monthly_balance
ORDER BY month;


-- ----------------------------------------------------------------------------
-- 4. Month-over-Month Growth (LAG window function)
-- ----------------------------------------------------------------------------
-- Percentage Growth = (New Value - Old Value) / Old Value * 100
-- Note: the first month in this dataset (Dec 2018) is a partial month with
-- very low volume, which produces an extreme/misleading growth % for
-- Jan 2019. Worth excluding or footnoting the first period in any
-- write-up of results.

WITH monthly_volume AS (
    SELECT
        DATE_TRUNC('month', CAST(posting_date AS DATE)) AS month,
        SUM(total_open_amount) AS total_volume
    FROM invoices
    GROUP BY DATE_TRUNC('month', CAST(posting_date AS DATE))
)
SELECT
    month,
    total_volume,
    LAG(total_volume) OVER (ORDER BY month) AS prior_month_volume,
    ROUND(
        (total_volume - LAG(total_volume) OVER (ORDER BY month))
        / NULLIF(LAG(total_volume) OVER (ORDER BY month), 0) * 100,
        2
    ) AS pct_growth
FROM monthly_volume
ORDER BY month;


-- ----------------------------------------------------------------------------
-- 5. Cumulative Growth Since Start of Dataset (FIRST_VALUE window function)
-- ----------------------------------------------------------------------------
-- Cumulative Growth = (Current Value / Start Value - 1) * 100
-- Anchors every month back to the first observed month without a self-join.

WITH monthly_volume AS (
    SELECT
        DATE_TRUNC('month', CAST(posting_date AS DATE)) AS month,
        SUM(total_open_amount) AS total_volume
    FROM invoices
    GROUP BY DATE_TRUNC('month', CAST(posting_date AS DATE))
)
SELECT
    month,
    total_volume,
    FIRST_VALUE(total_volume) OVER (ORDER BY month) AS start_volume,
    ROUND(
        (total_volume / FIRST_VALUE(total_volume) OVER (ORDER BY month) - 1) * 100,
        2
    ) AS cumulative_growth_pct
FROM monthly_volume
ORDER BY month;
