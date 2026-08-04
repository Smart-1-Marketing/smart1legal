# GHL / Smart1Suite Workflow — Smart 1 Legal Conquesting

How to wire the `smart1legal` app's webhook into GoHighLevel so every lead is captured
instantly, the proposal is emailed automatically, and your rep is notified while the
lead is still hot.

## How the app posts (what you're wiring against)

The app posts to **`GHL_WEBHOOK_URL`** (Inbound Webhook trigger) up to **twice per lead**,
distinguished by the **`report_status`** field:

| `report_status` | When | What it means |
|---|---|---|
| `captured` | The instant the form is submitted | Contact info + attribution only. **Lead is safe** even if AI fails. |
| `completed` | ~30–60s later | Full report fields + `report_pdf_url`. Fire the proposal email now. |
| `generation_failed` | Instead of `completed`, on AI failure | Lead needs a manual follow-up — the prospect was told the plan will be emailed. |

**Dedupe:** both posts carry the same `contact_email` — GHL merges them into one contact
when "Allow duplicate contacts" is OFF (default). The `completed` post simply enriches
the record created by `captured`.

## Field mapping (webhook payload → GHL)

**Contact fields** (present on every post):
`firm_name` (Company), `contact_name`, `contact_email`, `contact_phone`,
`proposal_recipient_email`, `website`, `firm_zip`, `practice_area`,
`secondary_practice_areas`, `target_radius`, `primary_goal`, `notes`.

**Attribution** (map to custom fields for ROI reporting):
`utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content`,
`gclid`, `fbclid`, `referrer_url`, `landing_page_url`.

**Opportunity fields** (on `completed`):
`opportunity_name`, `recommended_package`, `recommended_investment`,
`opportunity_value_monthly` (use as Opportunity Value), `opportunity_value_annual`.

**Report fields** (on `completed`):
`report_name` (always `legal-conquesting-report`), `report_pdf_url`,
`report_pdf_download_url` (forces download w/ nice filename — use this in emails),
`report_pdf_public_id`, `market_type`, `market_summary`,
`estimated_annual_cases_base`, `case_volume_label`, `weather_triggers`, `report_json`.

## Workflow 1 — "Legal Lead Captured" (speed-to-lead)

Trigger: Inbound Webhook → filter `report_status` **is** `captured`
1. Create/Update Contact (tag: `legal-conquesting-lead`, source: webhook `utm_source` or "Legal Gameplan Page")
2. Create Opportunity in your Legal pipeline → stage **"New Lead"**
3. **Internal notification** (SMS/push to rep): `New legal lead: {{contact.firm_name}} — {{contact.practice_area}} in {{contact.firm_zip}}. Plan generating now.`
4. Wait 0 min — done. (No customer email yet; the plan isn't ready.)

## Workflow 2 — "Proposal Ready" (the close sequence)

Trigger: Inbound Webhook → filter `report_status` **is** `completed`
1. Update Opportunity → stage **"Proposal Sent"**, value = `opportunity_value_monthly`
2. **Email the prospect** (to `proposal_recipient_email`, fallback `contact_email`):
   subject `Your {{practice_area}} market plan for {{firm_name}} is ready`,
   body links the **`report_pdf_download_url`**, CTA button → your booking calendar.
3. SMS (if consented): short "your plan is in your inbox" + booking link.
4. Internal notification to rep with `recommended_package` + `recommended_investment`
   ("$7,500/mo Market Domination — call now").
5. Nurture: +1 day email quoting `market_summary`; +3 days email quoting
   `estimated_annual_cases_base` `case_volume_label`; +7 days "still interested?" — all
   with the booking CTA. Exit on appointment booked or opportunity stage change.

## Workflow 3 — "Generation Failed" (rescue)

Trigger: Inbound Webhook → filter `report_status` **is** `generation_failed`
1. Tag `plan-generation-failed`; Opportunity stage **"Needs Manual Plan"**
2. **Urgent internal task/notification**: "Prospect was told their plan will be emailed —
   build & send manually within 1 hour." (Run the plan yourself in the tool and forward it.)

## Setup checklist

- [ ] Create the Inbound Webhook trigger once; use its URL as `GHL_WEBHOOK_URL` on Render
- [ ] Turn OFF "Allow duplicate contacts" so captured/completed merge
- [ ] Map the custom fields above (esp. `report_pdf_download_url`, UTM set, `recommended_investment`)
- [ ] Build the three workflows with `report_status` filters
- [ ] Send one test through the live form and confirm: contact created instantly,
      proposal email arrives with a working PDF link, rep notified twice
