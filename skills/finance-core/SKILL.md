---
name: finance-core
description: Activate when working with /finance:setup, /finance:analyze, or /finance:chat commands. Provides Monarch CSV parsing rules, tax reserve math, savings projection formulas, and the schema for finance.config.md and budget-snapshot.md.
version: 1.0.0
---

# Finance Core

Shared logic for the finance-planner plugin.

## Monarch CSV Format

Monarch Money exports CSVs with these columns (always read the header row first to confirm):
```
Date,Merchant,Category,Account,Original Statement,Amount,Notes,Tags,Type
```

- **Date**: YYYY-MM-DD
- **Amount**: negative = expense (money out), positive = income (money in)
- **Type**: `income`, `expense`, or `transfer`
- **Category**: Monarch's built-in categories (e.g. "Groceries", "Dining & Drinks", "Paycheck", "Income")

**Detecting income:** Look for rows where Amount > 0 and Type = "income", or Category contains "Paycheck" / "Income" / "Direct Deposit". Sum these per month.

**Detecting wife's income:** Look for deposits from tattoo-related merchants or irregular deposit amounts in the $800–$2,000 range that don't match a regular payroll pattern. Flag ambiguous deposits for user confirmation.

**Expense categories to track:**
- Housing (rent, utilities, insurance)
- Groceries
- Dining (restaurants, coffee, delivery)
- Transport (gas, transit, parking, rideshare)
- Subscriptions (streaming, software, memberships)
- Healthcare
- Shopping (clothing, household, Amazon)
- Other (everything else)

**Deduplication:** If multiple CSVs cover overlapping date ranges, deduplicate by (Date + Merchant + Amount). Keep one instance per unique transaction.

## Tax Reserve Calculation

Wife is self-employed (1099 tattoo artist, Massachusetts). Calculate quarterly:

```
wife_net_profit = wife_monthly_income - wife_monthly_business_expenses
tax_reserve_monthly = wife_net_profit × tax_reserve_rate   # default: 0.28

# Why 28%: SE tax ~15.3% + federal marginal ~7% (estimated) + MA state 5%
```

**Quarterly tax deadlines** (estimated tax payment dates):
- April 15 (Q1: Jan–Mar)
- June 15 (Q2: Apr–May)
- September 15 (Q3: Jun–Aug)
- January 15 (Q4: Sep–Dec)

**YTD tax reserve needed** = sum of (wife_net_profit × tax_reserve_rate) for all months so far in the calendar year.

## Savings Projection

```
wife_spendable = wife_monthly_income - tax_reserve_monthly
total_usable_income = your_takehome + wife_spendable
monthly_savings = total_usable_income - total_monthly_expenses
months_to_goal = down_payment_target / monthly_savings
target_date = current_month + months_to_goal months
```

**Always show two projections:**
- Conservative: `wife_monthly_income = wife_income_min` (from config)
- Average: `wife_monthly_income = (wife_income_min + wife_income_max) / 2`

**Pre-calculated what-if scenarios** (generate from actual top spending categories):
For each of the top 3 expense categories above Housing, calculate:
- `new_savings = monthly_savings + reduction_amount`
- `new_months = down_payment_target / new_savings`
- `months_saved = months_to_goal - new_months`

Example output line: "Cut Dining -$200/mo → saves 3 months (reach goal Oct 2029 vs Jan 2030)"

## finance.config.md Schema

The config file lives in the user's working directory. Format:

```markdown
# Finance Config
# Last updated: YYYY-MM-DD

your_takehome: 5200
wife_income_min: 3000
wife_income_max: 4000
wife_business_expenses: 380
tax_reserve_rate: 0.28
down_payment_target: 120000
condo_price_min: 500000
condo_price_max: 700000
last_updated: 2026-09-21
```

**Reading config:** Parse each `key: value` line. Treat numeric values as numbers.
**Writing config:** Preserve the comment header. Update `last_updated` on every write.

## budget-snapshot.md Schema

Written to user's working directory by `/finance:analyze`. Format:

```markdown
# Budget Snapshot — [Month YYYY]

_Generated: YYYY-MM-DD | Statements: [list of CSV filenames used]_

## Household Income

| Source | Amount/mo |
|--------|-----------|
| Your take-home (W-2) | $X,XXX |
| Wife's income (tattoo) | $X,XXX |
| Wife's business expenses | -$XXX |
| **Wife's net profit** | **$X,XXX** |
| Tax reserve (28%) | -$XXX |
| **Total usable income** | **$X,XXX** |

## Spending by Category

| Category | Amount | % of Usable Income |
|----------|--------|--------------------|
| Housing | $X,XXX | XX% |
| Groceries | $XXX | X% |
| Dining | $XXX | X% |
| Transport | $XXX | X% |
| Subscriptions | $XXX | X% |
| Healthcare | $XXX | X% |
| Shopping | $XXX | X% |
| Other | $XXX | X% |
| **Total Expenses** | **$X,XXX** | **XX%** |

## Savings & Projection

| | Conservative ($X,XXX wife/mo) | Average ($X,XXX wife/mo) |
|-|-------------------------------|--------------------------|
| Monthly savings | $X,XXX | $X,XXX |
| Months to $XXX,XXX goal | XX | XX |
| Estimated target date | Mon YYYY | Mon YYYY |

## Tax Reserve Status

| | Amount |
|-|--------|
| Wife YTD income | $XX,XXX |
| Wife YTD business expenses | $X,XXX |
| YTD net profit | $XX,XXX |
| Tax reserve owed YTD (28%) | $X,XXX |
| Next quarterly deadline | [Month DD, YYYY] |
| Recommended reserve balance | $X,XXX |

## What-If Scenarios

| Change | New monthly savings | Months saved | New target date |
|--------|---------------------|--------------|-----------------|
| Cut [Category] -$XXX/mo | $X,XXX | X | Mon YYYY |
| Cut [Category] -$XXX/mo | $X,XXX | X | Mon YYYY |
| Cut [Category] -$XXX/mo | $X,XXX | X | Mon YYYY |

---
_Previous snapshot: [Month YYYY] | Savings rate change: [+X%/-X%] | Monthly savings change: [+$XXX/-$XXX]_
```
(Omit the "Previous snapshot" line if no prior snapshot exists.)
