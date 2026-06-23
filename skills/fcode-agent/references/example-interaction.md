# Example interaction

A worked example of the iterative, confirmation-driven workflow — building a
process that fetches new Shopify orders and emails customers.

**User:** "Create a process to fetch new orders from Shopify API and send email
notifications with order details to customers"

**Agent:** "I'll help you create this integration. First, for sending emails,
would you prefer:

1. **Email service** (SendGrid, AWS SES) — easier setup, better deliverability
2. **Direct SMTP** — more control, no third-party needed

Which approach?"

**User:** "SendGrid"

**Agent:** "Perfect! Here's the plan:

- Input parameters: `dateFrom` (date), `dateTo` (date), `orderStatus` (select)
- Variables needed:
  - `SHOPIFY_API_KEY`, `SHOPIFY_SHOP_URL` (you'll create)
  - `SENDGRID_API_KEY` (you'll create)
  - `SENDGRID_FROM_EMAIL` (I'll create with a default)
- Dependencies: @shopify/shopify-api, @sendgrid/mail
- Modules: I'll create `shopify-client` and `sendgrid-client` for reusability

Steps:

1. Test Shopify & SendGrid connectivity with a discovery script
2. Implement order fetching logic
3. Implement email templating and sending
4. Test end-to-end with sample data

Shall I proceed?"

**User:** "Yes"

**Agent:** "I've created `SENDGRID_FROM_EMAIL`. Please create these sensitive
variables: `SHOPIFY_API_KEY`, `SHOPIFY_SHOP_URL`, `SENDGRID_API_KEY`. Let me know
when ready."

**User:** "Done"

**Agent:** "Thanks! Now I'll run a discovery script to test connectivity with
both Shopify and SendGrid, and fetch sample order data to understand the
structure."

_[Agent runs discovery script via `run_code`]_

**Agent:** "Great! Both connections are working:

- Shopify: connected to your shop, found 3 order statuses (pending, processing,
  completed)
- SendGrid: API key validated, ready to send emails

I can see the order structure includes: order_id, customer_email, order_total,
items. Now I'll create the main process with the order-fetching and
email-sending logic. Should I proceed?"

**User:** "Yes"

**Agent:** "Process created. I've implemented:

- Order fetching from Shopify with date-range and status filtering
- Email template with order details
- Error handling and logging

The process is ready. Would you like me to run a test execution with sample
data?"

_[… continues iteratively, validating each step with the user.]_
