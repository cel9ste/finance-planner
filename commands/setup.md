---
name: finance:setup
description: First-run wizard that collects household financial details and writes finance.config.md. Run this before /finance:analyze.
allowed-tools: ["Read", "Write"]
---

# Finance Setup Wizard

Guide the user through creating `finance.config.md` in their current working directory. Ask each question below **one at a time**, wait for the answer, then ask the next. Do not ask multiple questions at once.

If `finance.config.md` already exists in the working directory, read it first and pre-fill answers as defaults (show the current value in brackets so they can just press enter to keep it).

## Questions (ask in this order)

**Q1 — Your take-home:**
"What is your monthly net take-home pay? This is your after-tax paycheck amount — what actually hits your bank account each month."

**Q2 — Wife's income range:**
"What is your wife's monthly income range from tattooing? This is after the shop takes their 25% cut. Enter as a range like '3000-4000' or a single number if it's consistent."

Parse the answer: if range like "3000-4000", set `wife_income_min: 3000` and `wife_income_max: 4000`. If single number X, set both min and max to X.

**Q3 — Wife's business expenses:**
"What are her average monthly deductible business expenses? (supplies, equipment, continuing education, etc. — check her spreadsheet for a recent monthly average). Enter 0 if unsure."

**Q4 — Tax reserve rate:**
"What percentage of her net profit should we reserve for taxes? Press enter to use the default of 28% (covers ~15.3% self-employment tax + ~7% federal + 5% Massachusetts state income tax). Enter a different number only if you have a specific reason."

If user presses enter or says "default", use 0.28. Otherwise divide the entered percentage by 100 (e.g. user enters "30" → store 0.30).

**Q5 — Down payment target:**
"What's your down payment savings goal? Press enter for the default of $120,000 (20% of a $600k Cambridge condo), or enter your own target."

If user presses enter or says "default", use 120000.

**Q6 — Condo price range:**
"What's the condo price range you're targeting? Press enter for the default of $500,000–$700,000, or enter your own range."

If user presses enter or says "default", use 500000 and 700000. Parse a range like "500000-700000" into min/max.

## After collecting all answers

Write `finance.config.md` in the current working directory using this exact format:

```
# Finance Config
# Last updated: [today's date YYYY-MM-DD]

your_takehome: [answer]
wife_income_min: [min]
wife_income_max: [max]
wife_business_expenses: [answer]
tax_reserve_rate: [answer as decimal, e.g. 0.28]
down_payment_target: [answer]
condo_price_min: [min]
condo_price_max: [max]
last_updated: [today's date YYYY-MM-DD]
```

Then tell the user:
"Setup complete. Config saved to `finance.config.md`.

Next step: drop your Monarch CSV export into a `statements/` folder in this directory, then run `/finance:analyze`."
