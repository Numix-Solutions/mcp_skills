---
name: calculate-rnd-credit
description: >
  Calculate the client's estimated R&D Tax Credit.
  Use this skill when the client asks to calculate their R&D credit,
  estimated credit amount, research credit, or Section 41 credit.
  Triggers on: "calculate my credit", "how much credit do I have",
  "what is my R&D credit", "estimate my research credit".
user-invocable: true
---

# R&D Tax Credit Calculator

Calculate the client's estimated R&D Tax Credit by collecting all required
data first, then passing it to the calculation tool.

Follow every step in order. Never skip a step.
Never call the calculation tool until ALL data is collected.

---

## Step 1 — Determine the tax year

If the client did not specify a year, ask:
> "Which tax year would you like to calculate the R&D credit for?"

Store as `tax_year`. All fetches in the steps below use this year.

---

## Step 2 — Fetch current year expenses (QREs)

Fetch all categories in a single call using `fetch_total`. Supplies are coming in 'total' key:

- `fetch_total(category: "", year: tax_year)`

Store results as:
- `wages` = total wages for tax_year
- `contractors` = total contractor payments for tax_year
- `supplies` = total supplies/expenses for tax_year

Present a confirmation to the client:

> **Current Year Expenses (tax_year)**
> | Category | Amount |
> |---|---|
> | Wages | $wages |
> | Contractors | $contractors |
> | Supplies | $supplies |

If any value returns empty or zero, flag it to the client and ask them to confirm
the value is correct before proceeding.

---

## Step 3 — Collect Prior QRE Data (3 years)

The credit calculation requires QRE totals for the 3 years prior to tax_year.

Attempt to fetch each prior year from the platform using `fetch_prior_qre` for
years: `tax_year - 1`, `tax_year - 2`, `tax_year - 3`.

For each prior year where data IS available, store as:
- `qre_<year>`

For each prior year where data is NOT available or returns empty, pause and ask
the client. Note that this data can be found on **Form 6765** for each year.
Allow the client to skip a year if they don't have it — they may be a new
business that didn't file for the credit in prior years.

> "I couldn't find QRE data for [year]. Please provide (leave blank if not available):
> - Total QRE for [year]
>
> You can find this on Form 6765 for that year."

Wait for the client's input before continuing.

Present a confirmation once all prior year data is collected:

> **Prior Year QREs**
> | Year | QRE |
> |---|---|
> | tax_year - 1 | $qre_<tax_year-1> |
> | tax_year - 2 | $qre_<tax_year-2> |
> | tax_year - 3 | $qre_<tax_year-3> |

---

## Step 4 — Collect Gross Receipts (4 years)

The credit calculation requires gross receipts for the current year and the
3 prior years: `tax_year - 1`, `tax_year - 2`, `tax_year - 3`, `tax_year - 4`.

Attempt to fetch gross receipts from the platform for each year.

For each year where gross receipts are NOT available, pause and ask the client.
Note that for previous years this can be found on **Form 1120** using the formula:
`gross_receipts = Line 1c + Line 4 + Line 5 + Line 6 + Line 7 + Line 10`.
Allow the client to skip a year if the data is not available.

> "I couldn't find gross receipts for [year]. Please provide the total gross
> receipts for that year (leave blank if not available).
>
> For prior years, you can find this on Form 1120:
> Line 1c + Line 4 + Line 5 + Line 6 + Line 7 + Line 10"

Wait for the client's input.

Present a confirmation:

> **Gross Receipts**
> | Year | Gross Receipts |
> |---|---|
> | tax_year - 1 | $... |
> | tax_year - 2 | $... |
> | tax_year - 3 | $... |
> | tax_year - 4 | $... |

---

## Step 5 — Confirm before calculating

Before calling the calculation tool, present a full data summary to the client
and ask for confirmation:

> "Here's all the data I've collected. Shall I proceed with the credit
> calculation?"
>
> **Current Year: tax_year**
> | Category | Amount |
> |---|---|
> | Wages | $wages |
> | Contractors | $contractors |
> | Supplies | $supplies |
>
> **Prior QREs**
> | Year | QRE |
> |---|---|
> | [tax_year - 1] | $qre_<tax_year-1> |
> | [tax_year - 2] | $qre_<tax_year-2> |
> | [tax_year - 3] | $qre_<tax_year-3> |
>
> **Gross Receipts**
> | Year | Gross Receipts |
> |---|---|
> | [tax_year - 1] | $... |
> | [tax_year - 2] | $... |
> | [tax_year - 3] | $... |
> | [tax_year - 4] | $... |

Wait for explicit confirmation ("yes", "proceed", "looks good") before Step 6.

---

## Step 6 — Call the credit calculation tool

Once the client confirms, call `calculate_credit` with all collected data:
Prior QRE should be passed as a dictionary with year as key, e.g. `prior_qre = {"2022": 100000, "2021": 80000, "2020": 0}`.
Gross receipts should be passed in a similar format, e.g. `gross_receipts = {"2023": 500000, "2022": 450000, "2021": 400000, "2020": 350000}`.

---

## Step 7 — Present the results

Present the result clearly to the client:

> **Estimated R&D Tax Credit — tax_year**
>
> | | Amount |
> |---|---|
> | Estimated Credit | $result |
>
> _This is an estimate. Consult your tax advisor before filing._

If the calculation tool returns additional breakdown fields (e.g. base amount,
fixed-base percentage, ASC vs regular method), display all of them in the table.

---

## Step 8 - Generate Form 6765 and 3523
In this step you will generate Form 6765 and 3523 using the `generate_form_6765` and `generate_form_3523` tools, which takes all the data collected in previous steps and produces pre-filled forms for the client to review and download. These tools return a download link to the generated PDF

To get the company name and EIN, use the tool to get the company information, which will return the company name and EIN. If the tool does not return this information, ask the client to provide it.

## Error handling

- If any tool call fails, inform the client which step failed and ask if they
  want to retry or provide the value manually.
- If the client provides a value that seems unusually high or low (e.g. $0 or
  an outlier vs other years), flag it: "This looks different from other years —
  can you confirm this is correct?"
- Never make assumptions about missing data. Always ask.