# Upwork portfolio item — Docket (document intake)

**Title:** AI Invoice Intake with Fraud Checks — every value traced to the page, code decides

**Role:** Sole engineer: architecture, extraction pipeline, deterministic rules and risk gate, evaluation
harness, web app, integrations, infrastructure, security

**Skills:** Python, FastAPI, Pydantic AI, LLM document extraction, PostgreSQL, SQLAlchemy, Alembic,
Next.js, TypeScript, n8n, Google Sheets, webhooks, Docker, nginx

---

## Summary (~150 words)

Docket turns invoices, receipts, purchase orders and credit notes into checked, structured data. A vision
model reads each document and quotes the words it read for every value; code finds those words on the page,
draws a box around them and re-reads the value itself. A value with no place on the page is never accepted.
Arithmetic, IBAN, VAT numbers and dates are checked in code. A versioned risk gate stops the costly failures:
a known supplier's invoice asking to be paid into a new bank account, look-alike supplier names, duplicates,
invoices that don't match their purchase order. An independent second opinion (TypeSafe Jev) can only make
the decision more careful. People review only the exceptions, worst first, each with its reasons in plain
English. Verified data goes out as CSV / Excel / JSON, through signed webhooks, or via n8n into Google Sheets.

## Measured on the live system

- 99.5 % field accuracy, 100 % line items, 100 % correct routing on a 34-document evaluation set
- Median 2.6 s per document (checks ≈ 12 ms, second opinion ≈ 200 ms, the rest is the model)
- 17 / 17 fraud and error scenarios routed as expected on live sample batches
- 95 automated tests, mypy --strict, property-based tests for the validators

## Highlights for proposals

- Every extracted value points at where it came from; click a word on the page to correct one.
- Bank-detail change, look-alike supplier, duplicate and PO-mismatch checks before anything is paid.
- Only a person or your vendor list approves a bank account, never a document.
- Five models compared on the same documents; the cheapest one was the most accurate.
- Exports, signed webhooks and a ready n8n → Google Sheets flow; fits Xero / QuickBooks / ERP flows.
- Self-hostable; your documents stay on your server.

**Live:** https://adaptivesystems.dev/docket (live numbers) · case study
https://github.com/adaptivesystemsdev/docket-invoice-intake · walkthrough of the private app on request.
