---
name: d365-close-variance-investigator
description: Use when a finance user needs a period-end variance (flux) investigation in Dynamics 365 Finance & Operations (D365 F&O) through the Dynamics 365 ERP MCP server, covering trial-balance movement between two periods for one or more legal entities, a policy-threshold check, drill-down from flagged main accounts to journal and voucher level, anomaly detection (duplicates, unreversed accruals, weekend or manual postings, large journals, intercompany mismatches), and auditor-ready commentary with an evidence pack. Read-only against D365. Not for posting, reversing or approving journals, or for changing master data.
---

# D365 close variance investigator

Turns "explain the October movement in our P&L and balance sheet" into a finished, evidenced
investigation: what moved, why it moved, what looks wrong, and who has to act.
Everything is read from D365 F&O through the Dynamics 365 ERP MCP server. Nothing is written to D365.

## When to use
- Month-end or quarter-end flux review (period vs prior period, or vs same period last year).
- Audit support: "Why did account 6420 increase by $1.3M?" with voucher-level evidence.
- Pre-close sanity check across several legal entities, including a newly acquired one.

Not for: posting or reversing entries, releasing holds, changing accounts or dimensions,
budget planning, or forecasting.

## Inputs (ask once for anything missing; never guess)
| Input | Example | Default |
|---|---|---|
| Legal entities | 101, 323 | none, required |
| Current period | 2026-10 | last closed month |
| Comparison | prior month, or same month prior year | prior month |
| Flux policy | variance > 10% AND > $250,000 | ask the user, or read it from a close checklist the user names |
| Scope | P&L only, balance sheet only, or both | both |
| Account filter (optional) | 6000-6999 | all main accounts |
| Output | workbook + memo + owner emails (draft) | all three |

Read `references/flux-policy-template.md` for the default policy wording and override rules.

## Prerequisites
- The Dynamics 365 ERP MCP server is connected for this user, on F&O 10.0.47 or later
  (`data_find_entities_sql` needs 10.0.48; on earlier versions use `data_find_entities`).
- The user's D365 security role can read the general ledger, main accounts, and vendor and
  customer transactions for the requested legal entities. The MCP server only returns what
  that role can see; if an entity is missing, report it as a permission gap, not as "no data".

## Workflow

### Phase 1 - Discover the data model (once per session)
1. Call `data_find_entity_type` for each concept in `references/entity-discovery-map.md`
   (trial balance or ledger balances, general journal account entries, main accounts,
   vendor transactions, customer transactions, legal entities, intercompany).
2. For each chosen entity, call `data_get_entity_metadata`. Record the exact field names for
   legal entity (data area), main account, account name, posting date, accounting date,
   amount in accounting currency, voucher, journal number, posting type, transaction text,
   created by, created date and time, and any reversal or "posted by" fields.
3. Write the mapping to `working/entity_map.json`. Use only field names that the metadata
   returned. If a needed field does not exist, record it as "not available" and skip any
   rule that depends on it; say so in the memo's limitations section.

### Phase 2 - Pull balances and compute flux
4. For each legal entity, query period balances by main account for both periods with
   `data_find_entities_sql`. Prefer aggregated SELECTs (SUM by main account); if the server
   rejects aggregation, pull lines and aggregate in the script. Set `returnAsResource` to true
   whenever a result may exceed ~160 KB.
5. Save raw results to `working/balances_<entity>.json`.
6. Run `scripts/flux_engine.py` (see `references/script-usage.md`) to compute, per entity and
   main account: prior, current, $ change, % change ("n/m" when prior = 0), the policy flag,
   and a "large but under threshold" watch flag (absolute change > $ threshold but % under limit).
7. Tie-out check: total debits = total credits per entity and period, or the trial balance
   nets to zero. If it does not, stop and report the gap before any commentary.

### Phase 3 - Drill down on flagged accounts
8. For each flagged account (and up to five watch-list accounts), query general journal
   account entries for the current and comparison periods, filtered on entity and main account.
   Return voucher, journal, date, amount, text, posting type, user and created date and time.
9. Group lines into drivers in this order:
   a. Same vendor or customer, or the same transaction text pattern.
   b. Accruals and their reversals (match on amount and opposite sign within 45 days).
   c. Recurring run-rate (lines that also exist in the comparison period within +/-15%).
   d. Remainder ("other, not individually material").
   Build a bridge: comparison balance + each driver = current balance. The bridge must tie to
   the cent; if it does not, show the unexplained difference as its own line, never hide it.
10. Where a driver points to vendor or customer activity, confirm it against vendor or customer
    transactions (invoice number, invoice date, due date).

### Phase 4 - Anomaly checks
Run every rule in `references/anomaly-rules.md` on the drilled lines. Each hit gets:
rule ID, severity (High / Medium / Low), entity, account, voucher, amount, and evidence.
At minimum:
- A1 Possible duplicate: same vendor, same amount, within 10 days, different voucher.
- A2 Accrual not reversed: accrual in the prior period with no reversal in the current period.
- A3 Manual journal above the approval limit (default $1,000,000; take it from the policy).
- A4 Weekend or holiday posting, or posted outside 06:00-22:00 local time.
- A5 Round-amount manual journal (a multiple of 10,000) with generic text.
- A6 Intercompany mismatch: due-to in one entity vs due-from in the other, difference > $1,000.
- A7 Posting to an account that is inactive or blocked for the period.
- A8 Same user created and posted a journal above the approval limit (segregation of duties).

### Phase 5 - Commentary and deliverables
11. Write commentary for each flagged account in this format (keep each one under 80 words):
    "<Account> moved <+/-$X> (<Y%>) from <prior> to <current>. Primary drivers: <driver 1 $>,
    <driver 2 $>. <One sentence on whether the movement is expected, one-off or recurring>.
    <Open item, if any>." Never state a cause the data does not show; write
    "driver not determinable from ledger data - confirm with <owner role>" instead.
12. Produce the deliverables listed in `references/output-spec.md`:
    - Workbook `Flux_Investigation_<entities>_<period>.xlsx` with the sheets Summary, Flux,
      Bridges, Drill_Detail, Anomalies, Intercompany, Open_Items, Source_and_Method.
    - Word memo `Flux_Commentary_<period>.docx` for the Controller and auditors: summary,
      per-account commentary, anomalies needing action, limitations, and an evidence index
      (entity, query and filters, row counts, as-of time).
    - Owner emails, one per accountable owner, as DRAFTS ONLY, listing their open items and the
      evidence requested. Use the owner roles from the user's close checklist if one is given;
      otherwise address them to the requester for routing.
13. Finish in chat with: number of accounts reviewed and flagged per entity, the top three
    drivers, High-severity anomalies, and any open questions.

## Guardrails
- READ-ONLY against D365. Never call `data_create_entities`, `data_update_entities`,
  `data_delete_entities`, `form_set_control_values`, `form_save_form`, `form_click_control`
  or `api_invoke_action`. If the user asks to post, reverse or correct an entry, write the
  proposed journal (accounts, amounts, text) into Open_Items and explain that an authorized
  user must post it in D365 through the normal approval workflow.
- Never invent entities, fields, vouchers, amounts or causes. Every number in the memo must
  trace to a row in Drill_Detail or Flux.
- Emails are drafts. Do not send unless the user explicitly says to send.
- Stop and report if the trial balance does not tie, if an entity returns no rows, or if the
  MCP server is unavailable (for example, during a servicing window). Do not fall back to old files.
- Amounts are in each entity's accounting currency. When combining entities, state the
  reporting currency and the exchange-rate source, or keep entities separate.
- Dates from the MCP server are ISO format; convert to the user's locale only in the outputs.
- Mark every output "Prepared with AI assistance - review before use" and keep the
  Source_and_Method sheet complete so the work can be re-performed (SOX evidence).

## Performance and cost
- Typical run for two entities and about 10 flagged accounts is 25-40 MCP tool calls.
  Reuse `working/entity_map.json` instead of rediscovering entities.
- Prefer one aggregated query per entity and period over many single-account queries.
- Use `returnAsResource` for drill-downs larger than a few hundred lines.

## Example prompts
- "Run the close variance investigation for entities 101 and 323, October 2026 vs September,
  policy 10% and $250K. Draft emails to owners."
- "Why did 6420 Professional Fees jump in October for 101? Give me a bridge and auditor commentary."
- "Pre-close check for 330 (Northwind): flux vs September, intercompany against 101, anomalies only."

## Files in this skill
- `references/entity-discovery-map.md` - search terms and fields to find in D365.
- `references/anomaly-rules.md` - rule definitions, parameters and severity.
- `references/flux-policy-template.md` - default thresholds and override wording.
- `references/output-spec.md` - workbook sheets, columns and memo outline.
- `references/script-usage.md` - how to run the flux engine.
- `scripts/flux_engine.py` - flux, bridge tie-out and anomaly calculations on exported JSON.
