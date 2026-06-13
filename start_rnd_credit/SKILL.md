---
name: start-rnd-credit
description: >
  This skill is used to calculate the R&D Taxes Credit for a company. Provides step by step guidance to the client, collects necessary data, and calls the credit calculation tool. Finally, it generates pre-filled Form 6765 and 3523 for the client to review and download.
user-invocable: true
---


# R&D Tax Credit Calculator

Calculate the client's estimated R&D Tax Credit by collecting all required data first, then passing it to the calculation tool.

**Follow every step in order. Never skip a step.**

Never call the calculation tool until ALL data is collected. Before starting ask the client to confirm for which year they want to calculate the R&D credit. Store this year as `tax_year` and use it in all subsequent steps when asking for data. For example, when asking for Gross Receipts, ask for Gross Receipts for `tax_year` and 3 previous years (if available).

## Session Resume Protocol

At the start of any session (including resumed ones), silently call `fetch_wages`, `fetch_contractors`, `fetch_bank_statements`, and `fetch_prior_qre` to check what data already exists. If data is found for a `tax_year`, inform the client: "It looks like we have some data already on file for [year]. Would you like to resume from where you left off, or start fresh?" 

If resuming, skip completed steps and pick up at the first incomplete one. This avoids asking the
client to re-enter data they already provided.

## Step 0. Introduction and Prerequisites
Start by introducing the process to the client. Explain that calculating R&D Tax Credit is a multi-step process that requires collecting various data points about the company's expenses, wages, prior credits, and gross receipts. Inform the client that you will guide them through each step and help them collect the necessary information.

Also explaing, since AI agent doesn't support File uploads directly from the chat, client will need to upload necessary documents (Bank Statements, Payroll reports, Tax forms etc.) using the Dashboard: https://app.numix.app/dashboard?action=upload. So, it would be good to have this page open while going through the process. 

Also some documents take time to process based on the amount of data in them, so after uploading documents, client will need to wait for 15-20 seconds and then ask for a status in the chat. This is a normal part of the process, so please ask client to be patient after uploading documents.

Show the following memo to the client what data will be needed to calculate the R&D credit. This will help client to prepare all necessary documents in advance and make the process smoother and faster: 
| Step | Description | Required data |
| --- | --- | --- |
| 1 | Checking if 2FA enabled | 2FA status on client's account |
| 2 | Verifying Company information | Company name, EIN, address, EIN confirmation letter |
| 3 | Collecting Bank Statements | Bank statements for the tax year |
| 4 | Collecting information about Wages | Payroll report for the tax year, wages information for each employee (name, gross pay, R&D percentage) |
| 5 | Collecting Prior QRE data | Prior QRE data for 3 years before the tax year. If you filed R&D Tax Credits before, client can upload Form 6765 for previous years or provide the necessary data manually. Partial QRE currently not supportd, therefore we need data for 3 years or this can be skipped |
| 6 | Collecting Gross Receipts data | Gross Receipts data for 4 years excluding Tax year. If you have, please prepare the following Tax forms depending on your company type: 1120, 1120s or 1065 |

Preparing documents before starting the process will make it smoother and faster. Once the introduction is done, ask client to confirm that he's ready to proceed. If client is not ready, ask him if he has any questions or if he needs any help with preparing documents. Answer client's questions and provide necessary assistance before proceeding to the next step.

## Step 1. Checking if 2FA enabled

In this step you need to check if the client has 2FA enabled on their account. This is important for write operations (updating wages, update contractor payments, update Prior QRE and Gross Receipts data). For that you can call `get_user_info` tool and check if TOTP is enabled. If it's not enabled, warn client that to perform any Write operation, they will have to enable 2FA and provide an OTP code for authentication. Also provide instructions on how to enable 2FA and obtain OTP codes. Client should go to https://app.numix.app/dashboard/me?active=security

Before continuing and ask for client confirmation if they enabled 2FA and ready to proceed. Once client confirms, call `get_user_info` again to verify that 2FA is enabled. If it's still not enabled, inform client that you won't be able to proceed until the 2FA is enabled.


## Step 2. Verifying Company information
In this step call the `get_company_info` tool to get the company information. Ask client to verify if everything is correct. If not correct, ask client to provide correct information and update it using `update_company_info` tool. You can update Company Legal Name, EIN, Address, Company type and DBA (Optional) using this tool. Make sure to ask client for all these details if the information is not correct.

If client has EIN Confirmation letter, he can upload it using the same upload link as for Bank Statements: https://app.numix.app/dashboard?action=upload. Once client confirms the upload, call `get_company_info` and ask client if company information has been successfully updated with the uploaded EIN confirmation letter. If not, ask client to wait a bit and try again in 15-20 seconds. Processing the uploaded document can take some time, so it's normal if the information is not updated immediately after upload. If after several attempts the information is still not updated, inform client that he can still do it manually by typing in the chat what he wants to change

If client decides to update company information, call `get_company_info` again after the update to verify that the information was updated successfully.

Do not proceed until the company information is correct and verified by the client.


## Step 3. Collecting Bank Statements
Next step is to collect Bank Statements for year client wants to get R&D Tax Credits for. Explain to the client that we need Bank Statements to extract transactions and identify if they're eligible for R&D credit. 

Inform client that we support the following banks: 
* Mercury
* Chase
* Brex
* Silicon Valley Bank
* Capital One
* Bank of America
* US Bank
* Ramp
* American Express

If he has other bank, ask him to contact us at office@numix.co

Client can upload Bank Statements using the following options: 
1. Upload via Dashboard: https://app.numix.app/dashboard?action=upload
2. Send all files to Receipts URL. Take this URL from `get_user_info` response. Inform client that if he has a lot of files, he can ZIP them and send the ZIP file. 

Once the client confirms the upload, call `fetch_bank_statements` tool to fetch the uploaded Bank Statements. If Bank Statements are not found, ask client to wait a bit and try again in 15-20 seconds. Processing Bank Statements can take some time, so it's normal if they're not available immediately after upload. If after several attempts Bank Statements are still not found, ask client to contact us at office@numix.co or write to us in Slack channel. 

Do not proceed further until client confirms that all files are succesfully uploaded and shown in the output. 

Once the client confirms, call `fetch_expenses` method for the whole year and show client the list of transactions, total number of transactions, number of R&D eligible transactions. 

**IMPORTANT! R&D Eligibility Rule:** A transaction is R&D eligible ONLY if user_defined_r_d === "R&D Qualified Expense" (exact match). Any other value ("Unknown", "Not Eligible", or anything else) means the transaction is NOT eligible. Never use the r_d_eligible boolean field — it is an AI estimate only and must be ignored.

Inform the client they can review and correct transaction categorizations at:
https://app.numix.app/dashboard/tax/credits

Do not proceed to the next step until client confirms that transactions are correct and ready to proceed.


## Step 4. Collecting information about Wages

In this step we need to collect information about wages for the tax year. This information is an important part of the R&D credit calculation, as wages paid to employees who are directly involved in R&D activities can be eligible for the credit. 

**Before collecting new data**, call `fetch_wages` and `fetch_contractors` to check if wage data already exists. If found, show it to the client and ask if it is correct or needs changes. If correct, skip to the confirmation at the end of this step.

**IMPORTANT!**
* We need a payroll for provided Tax Year (tax_year). If client provides payroll for another year, ask him to provide payroll for the correct year.
* Once the employees and/or contractors are added to the system, from this session client will be able only to update R&D percentage for each employee/contractor. If client wants to add new employee/contractor or remove existing one, he will need to contact us at office@numix.co or write to us in Slack channel.

Payroll data can be provided using the following options: 
* Report from a Payroll provider (only PDFs are supported)
* Manually provide information for each employee

### Uploading data from Payroll provider

We support the following Payroll providers: 
* Gusto
* ADP
* Rippling

If client is using another payroll provider, ask him to contact us at office@numix.co or write to us in Slack channel. If client is not using any payroll provider and managing payroll manually, ask him to provide the necessary information manually in the next step.

If client selects provider from the list, ask him to generate a report in PDF format for the Tax Year. Here are the instructions for each provider:

**Gusto**:
1. Login to Gusto
2. From the left panel select **Reports**
3. Select the following: 
   * Payroll Summary
   * Payroll Journal
   * Contractor Payments
   * Tax Reports
4. Set the Date filter - Yearly
5. Click **Generate report** 
6. Once it's ready, click **Download** and select PDF format to download the report


**ADP**:
1. Login to ADP. Open ADP Workforce Now and sign in with your credentials.
2. From the top menu select **Reports** -> **Payroll reports**
3. Select the required report
   * Payroll Summary
   * Payroll Register
   * Employee Earnings Report
4. Select the date range for the report to cover the entire Tax Year (e.g. January 1, `tax_year` - December 31, `tax_year`)
5. Click **Run Report**
6. Once it's ready, click **Export** and select PDF format to download the report

**Rippling**:
1. Login to Rippling
2. Go to **Reports** -> **Payroll**
3. Select the required report - **Payroll Register**
4. Apply filters - set the Full Year for the Tax Year (e.g. January 1, `tax_year` - December 31, `tax_year`)
5. Click **Run Report**
6. Once it's ready, click **Export** and select PDF format to download the report

Once client has it, ask him to upload the report using the same upload link as for Bank Statements: https://app.numix.app/dashboard?action=upload.

Once the client confirms the upload, call `fetch_tax_forms` tool with type=payroll_report to fetch the uploaded payroll report. If the report is not found, ask client to wait a bit and try again in 15-20 seconds. Processing the report can take some time, so it's normal if it's not available immediately after upload. If after several attempts the report is still not found, ask client to contact us at office@numix.co or write to us in Slack channel.

Once the data is available, for each employee from the payroll report, determine their type:
- **W2 employees** → use `update_wages`
- **Contractors** → use `update_contractors`

**If the employee type is ambiguous from the report**, ask the client to confirm before calling
either tool.

For each employee, present the following fields (pre-filled from the report where available):
- `employee_name`
- `employee_type` (w2 or contractor)
- `period_type` (monthly or yearly)
- `period_start` / `period_end`
- `gross_wages`
- `r_d_percentage` — the estimated percentage of time spent on R&D activities

We're going to create Wages one by one for each employee using `update_wages`or `update_contractors` tool. Please display an interactive form for each employee with the following fields: employee_name, employee_type (contractor or w2), period_type (monthly or yearly), period_start, period_end, gross_wages, r_d_percentage. **If interactive forms are not available**, present the data as a structured table in chat and ask the client to confirm or correct each row before calling the update tool.

For `r_d_percentage`, you may estimate based on role if the client is unsure:
- Software engineer / engineering dept → ~80%
- Product manager → ~60%
- Sales / marketing → ~10%
Always confirm estimates with the client before saving.

Once the wages are updated, call `fetch_wages` and `fetch_contractors` tool to fetch the updated wages information and show it to the client. Ask client to confirm that everything is correct and if he wants to make any changes. If client wants to make some changes, allow him to do it and then call `update_wages` or `update_contractors` tool again to update wages information for each employee.


### Providing information manually
If client decides to provide wages information manually, ask him to provide the following information:

**Regular Employee (w2)**
- Employee name
- Period type (monthly or yearly)
- Period start and end dates
- Gross wages for the period
- Estimated R&D percentage (if client is not sure, you can provide estimates based on the role, but always confirm with the client before saving)

**Contractor**
- Contractor name
- Amount paid to the contractor for the period
- Estimated R&D percentage (if client is not sure, you can provide estimates based on the role, but always confirm with the client before saving)

Display an interactive simple form where client can provide employee data specified above. Once client clicks Submit button for each employee, call `update_wages` or `update_contractors` tool based on the employee type to save this information. If data has been successfully updated, call `fetch_wages` and `fetch_contractors` tool to fetch the updated wages information and show it to the client. Ask client to confirm that everything is correct and ask him whether he wants to add more employees or contractors. If client wants to add more employees/contractors, show the form again to provide information for the next employee. If client doesn't want to add more employees/contractors, proceed to the confirmation step.

Once all wages information is collected, show a summary table with all employees and their wages information and ask client to confirm that everything is correct and if he wants to make any changes. If client wants to make some changes, allow him to do it and then call `update_wages` or `update_contractors` tool again to update wages information for each employee.

## Step 5. Collecting Prior QRE data
In this step we need to collect Prior QRE data for 3 years before the tax year. This information is important for the R&D credit calculation, as it can affect the amount of credit that the client is eligible for.

**IMPORTANT**
Partial QRE currently not supported, therefore we need data for 3 years (tax_year - 1, tax_year - 2, tax_year - 3) or this step should be skipped entirely

Before proceeding ask client if he filed R&D Tax Credits before. If not, inform the client that without prior QRE data, the calculation will use the simplified credit method, which may result in a lower estimated credit than the regular method. Confirm they want to skip, then proceed to Step 6. If the client answers "Yes", then client needs to provide this data. It can be provided in two ways: 
1. Upload Form 6765 for previous years (tax_year - 1, tax_year - 2, tax_year - 3)
2. Provide the data manually (tax_year - 1 QRE amount, tax_year - 2 QRE amount, tax_year - 3 QRE amount)

### 5.1 Uploading Form 6765
If the client chooses to upload Form 6765, ask him to upload the 6765 form for previous years using the same upload link as for Bank Statements: https://app.numix.app/dashboard?action=upload. Once the client confirms the upload, call `fetch_tax_forms` tool with type=prior_qre to fetch the uploaded form. If the form is not found, ask client to wait a bit and try again in 15-20 seconds. Processing the form can take some time, so it's normal if it's not available immediately after upload. If after several attempts the form is still not found, ask client to contact us at office@numix.co or write to us in Slack channel.

Since Form 6765 doesn't have any indication of tax year, please ask client to specify which year corresponds to each uploaded form. This is important for us to correctly use this data in the R&D credit calculation. Once client provides this information, call `update_prior_qre` tool to update Prior QRE data for each year based on the uploaded forms. Once the request is successful, show a message that Prior QRE data was updated successfully and call `fetch_prior_qre`. Then ask client if this information is correct and if he wants to make any changes. If client wants to make some changes, allow him to do it and then call `update_prior_qre` tool again to update Prior QRE data.

If the request fails, show an error message and allow client to try again.

Do not proceed to the next step until client confirms that Prior QRE data is correct and ready to proceed.

### 5.2 Providing Prior QRE data manually
If the client chooses to provide Prior QRE data manually, ask him to provide the following information for each previous year (tax_year - 1, tax_year - 2, tax_year - 3):
* Tax Year
* Qualified Research Expenses (QRE) amount for that year

Once the client provides this information, call `update_prior_qre` tool to update Prior QRE data for each year based on the provided information. Once the request is successful, show a message that Prior QRE data was updated successfully and call `fetch_prior_qre`. Then ask client if this information is correct and if he wants to make any changes. If client wants to make some changes, allow him to do it and then call `update_prior_qre` tool again to update Prior QRE data.

Do not proceed to the next step until client confirms that Prior QRE data is correct and ready to proceed.

## Step 6. Collecting Gross Receipts data
In this step we need to collect Gross Receipts data for 4 years (tax_year - 1, tax_year - 2, tax_year - 3, tax_year - 4). This information is important for the R&D credit calculation, as it can affect the amount of credit that the client is eligible for.

This information can be found in 3 forms, based on what entity type client is (can be checked using `get_company_info` tool): 
* Form 1120 - C-Corp
* Form 1120S - S-Corp
* Form 1065 - Partnership

This data can be collected in two ways:
1. Uploading Form 1120, Form 1120s or Form 1065 for specified year
2. Providing Gross Receipts data manually per each year

### Step 6.1. Uploading Tax Forms
If the client chooses to upload tax forms, ask him to upload Form 1120, Form 1120s or Form 1065 for the tax year using the same upload link as for Bank Statements: https://app.numix.app/dashboard?action=upload. Once the client confirms the upload, call `fetch_tax_forms` tool with type=1120, type=1120s or type=1065 to fetch the uploaded form. If the form is not found, ask client to wait a bit and try again in 15-20 seconds. Processing the form can take some time, so it's normal if it's not available immediately after upload. If after several attempts the form is still not found, ask client to contact us at office@numix.co or write to us in Slack channel.

Once the forms are fetched, extract Gross Receipts data from the forms and show client the table with year and value and ask him to confirm that the data is correct. Once client confirms that the data is correct, call `update_gross_receipts` tool to update Gross Receipts data for each year based on the extracted data. Once the request is successful, show a message that Gross Receipts data was updated successfully and call `fetch_gross_receipts`. Then ask client if this information is correct and if he wants to make any changes. If client wants to make some changes, allow him to do it and then call `update_gross_receipts` tool again to update Gross Receipts data.

Don't proceed to the next step until client confirms that Gross Receipts data is correct and ready to proceed.

### Step 6.2. Providing Gross Receipts data manually
If the client chooses to provide Gross Receipts data manually, ask him to provide the Gross Receipts for 4 previous years (tax_year - 1, tax_year - 2, tax_year - 3, tax_year - 4) if available. Once the client provides this information, call `update_gross_receipts` tool to update Gross Receipts data for each year based on the provided information. Once the request is successful, show a message that Gross Receipts data was updated successfully and call `fetch_gross_receipts`. Then ask client if this information is correct and if he wants to make any changes. If client wants to make some changes, allow him to do it and then call `update_gross_receipts` tool again to update Gross Receipts data.

Don't proceed to the next step until client confirms that Gross Receipts data is correct and ready to proceed.


## Step 7. Calculating R&D Credit
In this step we're going to calculate the R&D credit. Before calling the corresponding tool, need to display a summary of all collected data to the client, including:
* **Supplies** — call `fetch_total` to get this information. Note: the backend returns a `total` key, but it represents Supplies only — display it as "Supplies", not "Total".
* **Wages and contractor payments** — call `fetch_wages` and `fetch_contractors` to get this information.
* **Prior QRE data for previous years** — call `fetch_prior_qre` tool to get this information.
* **Gross Receipts data for previous years** — call `fetch_gross_receipts` tool to get this information.

Before proceeding to the calculation, ask client to confirm that all data is correct and if he wants to make any changes. If client wants to make some changes, allow him to do it and then update the corresponding data using the corresponding tools (e.g. `update_wages`, `update_contractors`, `update_prior_qre`, `update_gross_receipts` etc.)

Once client confirms that all data is correct and ready to proceed, call `calculate_credit` tool to calculate the R&D credit. Once the calculation is done, show the estimated credit amount to the client.

## Step 8. Contact Numix
Once the client confirms that everything is OK, display him a button "I am done", text input for comments (if any) and message that from this point we'll take it. Once client clicks the button, call the the `credits_done` tool with client's message. This will notify our team that the client has completed the process and provide us with any comments or feedback they may have. After that, our team will review the collected data and the estimated credit amount, and will contact the client if we need any additional information or clarification.


## Error handling

- If any tool call fails, inform the client which step failed and ask if they
  want to retry or provide the value manually.
- If the client provides a value that seems unusually high or low (e.g. $0 or
  an outlier vs other years), flag it: "This looks different from other years —
  can you confirm this is correct?"
- Never make assumptions about missing data. Always ask the client to provide it or confirm it.