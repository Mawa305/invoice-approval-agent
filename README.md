# Invoice Approval Agent

An n8n workflow that checks incoming invoices with an AI policy agent. Small invoices are approved automatically. Larger ones go to a human for approval before anything is recorded as approved.

## Flow

1. **Webhook** receives the invoice (vendor, invoice number, amount, currency, due date, PO number, description).
2. **Valid Invoice?** checks that all required fields are present and the amount is greater than 0. Invalid requests get a 400 response.
3. **Respond Accepted** returns 202 right away, so the sender does not wait for approval.
4. **Policy Agent** decides `auto_approve` or `needs_human` against the policy rules and returns JSON with a short reason.
5. **Parse Policy** turns the agent output into structured data. If the output cannot be parsed, the invoice goes to a human.
6. **Auto Approve?** routes the invoice:
   - **Auto-approve:** the owner gets a notification email, and the invoice is logged as `auto_approved`.
   - **Needs human:** the owner gets an approval email with Approve and Reject buttons.
7. **Human Approved?** routes the result:
   - **Approved:** logged as `approved`.
   - **Rejected:** logged as `rejected`.

All outcomes are logged in a Google Sheet with the timestamp, vendor, invoice number, amount, currency, PO number, decision, outcome, and reason.

## Policy Rules

- Auto-approve only if the amount is 50000 or less, the currency is PKR, and a PO number is present.
- Everything else needs human approval.

Change these in the **Policy Agent** system prompt and in the **Parse Policy** logic if your limits differ.

## Required Credentials

- OpenAI API
- Gmail (OAuth2)
- Google Sheets (OAuth2)

## Setup

1. In n8n, go to **Import from file** and select `invoice-approval-agent.json`.
2. Re-select the credentials for OpenAI, Gmail, and Google Sheets.
3. Replace `OWNER_EMAIL` in the **Human Approval**, **Notify Owner Auto Approved** nodes with your email.
4. Replace `YOUR_GOOGLE_SHEET_ID` in the three Google Sheets nodes with your sheet's ID.
5. Create a Google Sheet with a tab named `Invoices` and these headers in row 1:
   `timestamp | vendor | invoice_number | amount | currency | po_number | decision | outcome | reason`
6. Publish the workflow, or run it with **Listen for test event** while testing.

## Test

Send a POST request to `/webhook/invoice-intake`:

```json
{
  "vendor": "Cool Breeze Parts Co.",
  "invoice_number": "INV-2041",
  "amount": 185000,
  "currency": "PKR",
  "due_date": "2026-11-05",
  "description": "Compressor and capacitor replacement, 3 units",
  "po_number": "PO-778"
}
```

Use `amount: 20000` to test the auto-approval path. Amounts above 50000 go to human approval.

## Notes

- Invoices are never marked approved without a human decision unless they fall under the auto-approval limit.
- Built as part of an n8n automation portfolio.
