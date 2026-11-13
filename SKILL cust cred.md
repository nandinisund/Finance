---
name: d365-customer-credit-snapshot
description: Use when someone asks for a customer's balance, credit limit, or credit-hold status in Dynamics 365 Finance & Operations (D365 F&O) for a given legal entity, through the Dynamics 365 ERP MCP server. Read-only. Not for releasing credit holds, changing customer master data, or posting transactions.
---

# D365 customer credit snapshot

Answers one question: "What does this customer owe us, and are they on credit hold?"
Works through the Dynamics 365 ERP MCP server (dynamic data tools). Read-only.

## Prerequisites
- The Dynamics 365 ERP MCP server is connected for this user (Cowork plugin or Copilot Studio tool).
- The user's D365 security role grants read access to customers and customer transactions.
  The MCP server only returns entities, fields and records that role can see.

## Inputs
- Customer name or customer account number (required).
- Legal entity, for example 101 or 323. If it is not given, ask once. Never guess.

## Steps
1. Find the customer entity: call `data_find_entity_type` with the search term "customer".
   From the hits, choose the customer master entity.
2. Call `data_get_entity_metadata` on that entity. Confirm the field names for customer
   account, name, legal entity (data area), credit limit, and the blocked/credit-hold field.
   Use only fields the metadata returns.
3. Read the customer with `data_find_entities_sql` (on versions earlier than 10.0.48 use
   `data_find_entities`), filtered on legal entity and on customer account or a name match.
   Select only the fields confirmed in step 2.
   - If more than one customer matches, list them (account, name, legal entity) and ask which one.
   - If none match, say so and stop.
4. Repeat steps 1–3 for open customer transactions (search "customer transaction" or
   "open transactions"), filtered on the same customer and legal entity. Sum the open amounts.
   For large results, set `returnAsResource` to true.
5. Calculate headroom = credit limit − open balance. Present the output below.

## Output
| Field | Value |
|---|---|
| Customer / account | … |
| Legal entity | … |
| Currency | … |
| Credit limit | … |
| Open balance | … |
| Headroom (limit − balance) | … |
| Credit hold / blocked status | … |
| Flag | "Over limit" if headroom < 0; "Below $70K threshold" if open balance < 70,000 |

Always end with a source line:
`Source: <entity names> · Filter: <filter used> · Rows: <count> · As of <date and time>`

## Variations
- "All customers on credit hold in <legal entity>": run step 3 filtered on the hold/blocked
  field instead of one customer and return one row per customer, sorted by open balance.
- "Make this weekly": suggest a scheduled task that runs this skill every Monday and
  emails the table to the requester (draft first unless the user says to send).

## Guardrails
- READ-ONLY. Never call `data_create_entities`, `data_update_entities`, `data_delete_entities`,
  `form_set_control_values`, `form_save_form`, `form_click_control` or `api_invoke_action`.
- If asked to release a hold or change a credit limit, explain that an authorized user must do
  it in D365 F&O, and offer to draft the request instead.
- Never invent a field, entity or value. If a field is not in the metadata, say it is not available.
- Show amounts in the legal entity's currency and state the currency. Dates use ISO format.
- If the MCP server is unavailable (for example, during a servicing window), say so and stop.
  Do not answer from memory or from older files.

## Example prompts
- "Use d365-customer-credit-snapshot for Contoso Health in legal entity 101."
- "Which customers in 323 are on credit hold? Show balance and headroom."
- "Is account C-10045 over its credit limit in 101?"
