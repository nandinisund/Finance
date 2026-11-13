---
name: monthly-variance-review
description: Use when I ask for a variance review, budget-vs-actual, or month-end variance summary of the coffee chain P&L in this workbook.
---

# Monthly variance review

## Purpose
Produce the same budget-to-actual review every month, for one month I name, so the close commentary is consistent no matter who runs it.

## Rules
1. Read the "P&L Data" sheet. Rows with Version = "Actuals" are actuals; rows with Version = "Budget" are budget. Month is stored as text in YYYY-MM form.
2. Compare Actuals to Budget for each Location and Account in the month I name.
3. Variance $ = Actual minus Budget. Variance % = Variance $ divided by Budget. Always show dollars before percent.
4. Revenue above budget is Favorable. Any cost line (Cost of Sales, Labor, Rent, Utilities) above budget is Unfavorable.
5. Flag a line when the absolute Variance % is above 10 percent OR the absolute Variance $ is above $25,000, whichever is hit first.
6. Sort from the worst unfavorable variance to the best favorable one. Never sort alphabetically.
7. If a location has no Actuals rows in the month, it was not open yet. Leave it out of the table and list it under "Not yet open". It is not a miss.
8. If references/company-overview.md shows a location opened partway through the month, label its lines "Partial month" in the Flag column.
9. Explain variances only from the "Manager Commentary" sheet for the same month and location. If no comment covers a flagged line, write "cause not stated". Never guess a cause.

## Output
1. A table with columns: Location, Account, Actual, Budget, Variance $, Variance %, Fav/Unfav, Flag, Cause.
2. A short "Not yet open" list, only if rule 7 applied.
3. Three sentences of commentary that name the single largest driver first.
