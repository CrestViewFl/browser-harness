# Instantly: campaign setup and billing checks

Use the authenticated app at `https://app.instantly.ai`. Prefer the documented
API with an already configured API key for bulk operations. Never copy browser
session tokens into files or documentation.

## Routes and discovery

- Campaign list: `/app/campaigns`.
- Campaign: `/app/campaign/{uuid}/analytics`. Read the campaign link from the
  list before navigating. Tabs include Leads, Sequences, Schedule, and Options.
- Billing: `/app/settings/billing`; the Credits tab link adds
  `?product=instantly-credits`.
- Current public API index: `https://developer.instantly.ai/llms.txt`.
  Each endpoint has a `.md` page with its OpenAPI schema. Full schema:
  `https://api.instantly.ai/openapi/api_v2.json`.

## Public API patterns

API base `https://api.instantly.ai/api/v2`, Bearer authentication, JSON bodies.
Set a descriptive application User-Agent: Python urllib's default User-Agent
can receive HTTP 403 / Cloudflare error 1010 even with a valid API key.

- `GET /workspace-billing/plan-details` gives contact capacity, usage, subscription
  renewal and verification-credit balance. Subscription renewal is not proof of
  an email-allowance reset; do not conflate the two.
- `GET /accounts` and `GET /campaigns` are paginated. Save only necessary account
  fields; an account response can contain unrelated configuration.
- `POST /leads/list` is a read operation with a JSON filter. It accepts `campaign`,
  `limit` (up to 100), and `starting_after`. Follow `next_starting_after` until
  the page is empty or no cursor remains; a final short page may still have one.
- `POST /campaigns` creates a draft. `PATCH /campaigns/{id}` saves its settings.
  Activation is a separate, consequential `POST /campaigns/{id}/activate`.
- `POST /leads/add` accepts `campaign_id`, up to 1,000 `leads`,
  `verify_leads_on_import`, and `skip_if_in_workspace`. The default workspace
  blocklist is applied. `company_name` maps to the `{{companyName}}` variable.
- `DELETE /leads` accepts `campaign_id` and exact `ids`. Deletion frees contact
  storage, not monthly sends. Export and verify a backup before authorized
  deletion; never delete unfinished sequence contacts to rotate a batch early.

## Sequence pitfalls verified through persisted readback

1. **Wrap text in HTML elements.** Bare root text mixed with `<br/>` and anchors
   can be silently stripped by the API sanitizer, leaving just breaks and links.
   `<div>Sentence</div><div><br/></div>` preserves the text. After saving, GET the
   campaign and compare normalized visible text, subjects, links and delays to
   the intended copy. A successful HTTP response alone is insufficient.
2. Use one `sequences` array item containing all `steps`. The step's `delay` is
   the delay **before the next email**, with `delay_unit` defaulting to `days`.
   The service may add default fields such as `pre_delay_unit`; compare relevant
   semantics rather than full raw-object equality.
3. Blank subjects on later steps continue the actual thread. Do not send an
   instructional placeholder or manually invent `Re:`.
4. **Weekday keys use Sunday=0.** Monday-Friday is keys 1-5 true, 0 and 6 false.
   The schema examples can be misleading. Verify the saved Schedule tab's
   labeled checkbox states after reload, not just the JSON that was submitted.
5. False options may read back as null. Confirm their effective UI state instead
   of repeatedly patching semantically equivalent false/null values.
6. Imported lead `status=1` means active within the campaign; a draft campaign
   still does not send. Verification is a different field: 1 verified, 11/12
   pending, negative values non-deliverable/risky categories. Missing status is
   unverified, not valid.

## Verification and capacity

Verification credits are separate from the outreach sending subscription.
Check current balance and cost before enabling verification, and obtain user
authorization for any new purchase. On the Leads tab select the intended batch
and choose Verify leads. Read back the completed results before launch.

Contact capacity counts campaign contacts; CRM Lists have different storage
semantics. Allow 5-10 minutes for billing usage to update after deletion. Export
outcomes, preserve opt-outs and Unibox history, and keep a durable processed-email
ledger so deleted contacts are not accidentally enrolled again.

For UI exploration prefer screenshots, then DOM inspection when required. Locate
tabs by visible text and schedule controls by their enclosing labels. Native
checkbox `checked` properties provide a useful verification of the actual day
mapping after a saved campaign reload.
