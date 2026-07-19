# Risk-Management-Formulas
Financial risk and portfolio formulas used in B2B factoring and account management

Risk Management Formulas
Core financial and risk formulas used in B2B invoice factoring and account
management, applied to real portfolio decisions around collections risk,
yield, and exposure.
Purpose
In factoring, decisions about which accounts to escalate, which invoices
carry outsized risk, and how a portfolio is performing can't rely on raw
averages alone — a single large outlier invoice can distort the picture.
These formulas were used to weight, monitor, and forecast portfolio health
on a $15M+ revolving portfolio.
Formulas
Portfolio Turnover
Measures how efficiently invoices move through the portfolio in a given month.
Monthly Portfolio Turnover =
    (Invoices Purchased - Invoices Paid) /
    ((Total Starting Balance + Ending Balance) / 2)
Return on Investment (ROI)
ROI = (Discount + Schedule Fees + Misc Fees) / Client Advance
Yield
Yield = ROI / (Days to Pay / 365)
Weight %
Used to prevent small, slow-paying invoices from skewing decision-making
relative to large, fast-paying ones.
Weight % = Purchased Amount / Total Purchased Amount
Example: a $25 invoice taking 100 days to pay shouldn't carry equal weight
in decision-making against a $50,000 invoice paying in 30 days. Weighting
corrects for this.
Weighted Average Yield
Weighted Average Yield = SUMPRODUCT(Yield, Weight %)
Actual Average Yield
Includes the weighting effect directly in the yield calculation.
Actual Average Yield = (Discount + Schedule Fees / Client Average) /
                        (Average Days to Pay / 365)
Loan Ratio
Used to monitor exposure relative to total accounts receivable.
Loan Ratio = (Loan Amount / Total AR) * 100
Growth Metrics
Percentage Growth = ((New Value - Old Value) / Old Value) * 100

Compound Annual Growth Rate (CAGR) =
    ((End Value / Start Value) ^ (1 / Number of Years)) - 1

Cumulative Growth = ((End Value / Start Value) - 1) * 100
