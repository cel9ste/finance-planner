---
name: finance:analyze
description: Parse Monarch CSV statement exports from the statements/ folder, confirm detected values, and write budget-snapshot.md with spending breakdown, savings rate, and projection to condo down payment goal.
allowed-tools: ["Read", "Write", "Bash", "Glob"]
---

# Finance Analyze

Generate a monthly budget snapshot from Monarch CSV exports.

## Step 1: Check prerequisites

Check if `finance.config.md` exists in the current working directory.
- If missing: tell the user "No config found — let me set that up first." then run the `/finance:setup` flow before continuing.
- If present: read it and parse all key: value pairs.

## Step 2: Find statement files

Look for CSV files in `statements/` in the current working directory.

```bash
ls statements/*.csv 2>/dev/null
```

If no CSVs found: "No statement files found. Please export your transactions from Monarch Money and drop the CSV into a `statements/` folder in this directory, then run `/finance:analyze` again."

## Step 3: Parse CSVs

For each CSV file found:
1. Read the header row to identify column names and their positions
2. Expected Monarch columns: `Date`, `Merchant`, `Category`, `Account`, `Original Statement`, `Amount`, `Notes`, `Tags`, `Type`
3. If column names differ, adapt accordingly — the key fields are Date, Amount, Category, and Merchant
4. Parse all transaction rows

**Determine the analysis month:** Use the most recent full calendar month represented in the transactions. If transactions span multiple months, focus on the most recent complete month. Tell the user which month you're analyzing.

**Deduplicate** across files: treat (Date + Merchant + Amount) as a unique key; keep one instance.

**Income detection:**
- Sum all rows where Amount > 0 and (Type = "income" OR Category contains "Paycheck"/"Income"/"Direct Deposit"/"Salary")
- Separate your regular payroll (consistent large amount, same merchant each period) from wife's income (irregular amounts, tattoo/cash/Venmo/Zelle patterns)
- Flag any deposit you cannot confidently categorize for user confirmation

**Expense categorization:**
Map Monarch categories to these groups: Housing, Groceries, Dining, Transport, Subscriptions, Healthcare, Shopping, Other. When in doubt, use Other. Exclude transfers between accounts.

## Step 4: Confirm detected values

Present a confirmation block — **do not proceed without user confirmation**:

```
Detected values for [Month YYYY] — confirm or correct each:

  Your take-home:          $[detected or config value]/mo  → [press enter or type new amount]
  Wife's income:           $[detected or "not detected"]/mo  → [press enter or type new amount]
  Wife's business exp:     $[config value]/mo  → [press enter or type new amount]

  Any income not in statements? (cash tips, Venmo, etc.):  → [enter amount or press enter to skip]
```

If the user corrects any value, update `finance.config.md` with the corrected value and today's date in `last_updated`.

If wife's income was not detectable from statements, use the average of `wife_income_min` and `wife_income_max` from config and note this in the snapshot.

## Step 5: Calculate

Using confirmed values and the formulas from finance-core skill:

1. `wife_net_profit = wife_income - wife_business_expenses`
2. `tax_reserve_monthly = wife_net_profit × tax_reserve_rate`
3. `wife_spendable = wife_income - tax_reserve_monthly`
4. `total_usable_income = your_takehome + wife_spendable`
5. `total_expenses = sum of all expense categories`
6. `monthly_savings = total_usable_income - total_expenses`
7. `months_to_goal_conservative = down_payment_target / (monthly_savings calculated with wife_income_min)`
8. `months_to_goal_average = down_payment_target / (monthly_savings calculated with wife average income)`
9. `target_date = add months to today's date`

**YTD tax reserve:** If prior snapshots exist, sum wife's net profit across all months this calendar year and multiply by `tax_reserve_rate`.

**What-if scenarios:** Take the top 3 expense categories (excluding Housing) by amount. For each, suggest a round-number reduction ($50, $100, $150, or $200 — pick the one closest to 20% of that category's spend). Calculate new monthly savings and new months-to-goal using average projection.

**Month-over-month delta:** If `budget-snapshot.md` already exists, read the previous monthly savings figure and calculate the change.

## Step 6: Write budget-snapshot.md

Write `budget-snapshot.md` in the current working directory using this schema exactly. Overwrite any existing snapshot.

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

If wife's income was not detected from statements and you fell back to the config average, add this note below the Household Income table:
`_Wife's income not detected in statements — using config average ($X,XXX/mo). Correct above if needed._`

## Step 7: Confirm to user

Tell the user:
- Which month was analyzed
- Monthly savings rate (conservative and average)
- Timeline to goal (both projections)
- The single highest-leverage what-if scenario
- "Run `/finance:chat` to explore strategies interactively."
