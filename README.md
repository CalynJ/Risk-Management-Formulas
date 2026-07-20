# Risk Management Formulas

Financial risk and portfolio formulas used in B2B invoice factoring and account management, demonstrated in SQL against a real 50,000-row invoice/accounts receivable dataset.

## Purpose

In factoring, decisions about which accounts to escalate, which invoices carry outsized risk, and how a portfolio is performing can't rely on raw averages alone — a single large outlier invoice can distort the picture. These formulas were used to weight, monitor, and forecast portfolio health on a $15M+ revolving portfolio.

## Files

- `invoice_data.csv` — 50,000-row real invoice-level accounts receivable dataset (invoice dates, due dates, clearing/payment dates, amounts, customer IDs, open/closed status)
- `risk_formulas_queries.sql` — SQL implementing the formulas below against this dataset, using CTEs and window functions

## Formulas Implemented in SQL

### Days to Pay (DTP) and Aging Buckets
Core collections risk metric: how long each invoice took, or is taking, to clear, bucketed into standard risk tiers (0-30, 31-60, 61-90, 90+ days, plus open/past-due status).

### Loan Ratio (Customer Concentration)
```
Loan Ratio = Customer Total AR / Portfolio Total AR * 100
```
Used to monitor exposure concentration — flags when a small number of customers carry a disproportionate share of total receivables.

### Monthly Portfolio Turnover
```
Monthly Portfolio Turnover =
    (Invoices Purchased - Invoices Paid) /
    ((Total Starting Balance + Ending Balance) / 2)
```

### Percentage Growth / Cumulative Growth
```
Percentage Growth = ((New Value - Old Value) / Old Value) * 100
Cumulative Growth = ((Current Value / Start Value) - 1) * 100
```
Applied month-over-month to portfolio volume using window functions (`LAG`, `FIRST_VALUE`) rather than self-joins.

### Weight %
Used more generally to prevent small, slow-paying invoices from skewing decision-making relative to large, fast-paying ones.
```
Weight % = Purchased Amount / Total Purchased Amount
```
Example: a $25 invoice taking 100 days to pay shouldn't carry equal weight in decision-making against a $50,000 invoice paying in 30 days. Weighting corrects for this.

## Formulas Not Calculable From This Dataset

The dataset used here does not include fee-level detail (discount fees, schedule fees, misc fees), so the following formulas — while used in real account management work — aren't demonstrated in the SQL file, since there's no underlying data to calculate them from:

```
ROI = (Discount + Schedule Fees + Misc Fees) / Client Advance
Yield = ROI / (Days to Pay / 365)
Weighted Average Yield = SUMPRODUCT(Yield, Weight %)
Actual Average Yield = (Discount + Schedule Fees / Client Average) / (Average Days to Pay / 365)
```

## Data Notes

- **Reference date:** this dataset covers Dec 2018–May 2020. Aging calculations use the dataset's own most recent date as the "as of" point, rather than today's date, so open invoices are evaluated in the context of when the data was collected.
- **Entity resolution:** the same real-world customer (e.g., a large retail chain) appears under multiple inconsistent name variants in the raw data. Queries group by customer ID rather than name to avoid fragmenting true exposure — a data quality issue worth flagging in any real risk process rather than working around silently.
- **First-month growth:** the first month in the dataset is a partial period with unusually low volume, which produces a misleading growth percentage for the following month. Worth excluding or footnoting in any summary of results.

## Notes

Traditional financial management formulas (standard CAGR, generic growth rate calculations) don't always translate cleanly to factoring-specific risk decisions, where short payment cycles and invoice-level variance matter more than they would in traditional lending or investment contexts. These formulas were adapted with that in mind.
