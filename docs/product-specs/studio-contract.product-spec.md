---
spec_format_version: "0.1"
title: "Studio-Contract"
artifact_type: "prd"
spec_revision: 4
author: "ProductSpec.io"
created_at: "2026-07-09T00:00:00Z"
updated_at: "2026-07-17T00:00:00Z"
---

## Problem

Sales admins currently configure Studio pricing for reseller contracts through the manual flow. This causes a no traces of contract in our database which in further difficult to manage Reseller Console including reporting and auto bill generation.

## Hypothesis

If admins configure Studio pricing (products, VIN-tier slabs, commitment, add-ons) through a structured console form instead of Word, pricing errors will drop and contracts will accurately reflect billing math, because slab lookups, bi-directional discount calculation, and fee totals are enforced by the system instead of manual arithmetic.

## Product Summary

A console form lets a sales admin configure Studio pricing for a reseller — choose Usage or Monthly pricing, set VIN-tier rates and discounts per product, pick a commitment and add-ons — and see it combine with Vini's pricing into one accurate Billing Summary, all without touching a Word document. This covers the full contract lifecycle for reseller customers: create, save as draft, generate the contract PDF, and amend an existing contract. Beyond the contract itself, each Studio product carries a monthly VIN wallet at the Reseller level that drives actual billing (minimum-commitment floor, rollover, overage), while Resellers separately get self-serve control in Partner Console to cap how much of that pool an individual Dealer or Rooftop can draw on.

## Scope

```productspec-scope
in:
  - Usage vs. Monthly pricing model selection, mutually exclusive
  - 5 independent products (Images, 360° Spin, Video tour, Studio Instant, Studio Promote), each with its own VIN-tier rate slab — no shared "Lite/Pro" plan bundling
  - Bi-directional Offered Price ↔ Disc % (editing either recalculates the other from Cost Price), for both Studio and Vini products, with NO ceiling on Offered Price for either — it may exceed Cost Price (markup), producing a negative Disc %, and must not be clamped
  - Optional Commitment: None or Minimum (floor amount)
  - Add-ons (White label, Quality check, Custom backgrounds) and Add-on products (Studio Instant, Studio Promote) with independent enable/disable
  - Studio Integration Fee note + waiver checkbox (informational, non-blocking)
  - Reseller Type (Partnership/SaaS) and Clause review, surfaced in the Terms & Conditions section, always visible (not progressively gated)
  - Unified "Advance Breakdown" table (title with no "per live Rooftop" qualifier) combining Studio (reseller-wide) and Vini (per-rooftop) with an adaptive Basis column; Monthly Fee shows the per-unit rate only (e.g. `$3/VIN`, `$500/Rooftop`) for both products, never a computed total, and there is no Total row — this table is a rate card at contract-creation time, not the actual invoice
  - Global currency selector propagates to every price symbol across Studio and Vini
  - Draft save/resume for a Studio-priced reseller contract, using the existing draft/prefill mechanism unchanged
  - Amendment of an existing reseller contract's Studio pricing, using the existing amendment mechanism unchanged (new linked contract version per amendment; no new approval/re-signature step). A Studio product not previously selected may be newly enabled on amendment; a previously-enabled product cannot be disabled but its VIN count/Offered Price/Disc % remain editable
  - Contract PDF: Commercial Summary renders Studio and Vini as two separate tables (not merged), followed by an Additional Terms section listing selected clauses. Amendment PDFs are a full regenerated document (not a delta/addendum), consistent with the existing PDF pipeline
  - A monthly VIN credit wallet per Studio product, per Reseller (never per Dealer/Rooftop): topped up each month with the contracted VIN count, debited per request processed, balance allowed to go negative to permit overusage
  - Usage-model billing: charge the greater of (actual usage × contracted rate) and the minimum commitment floor; if actual usage is below the floor, the unused VIN balance rolls over into next month's wallet
  - Monthly-model billing: fixed monthly fee regardless of actual usage; wallet resets every cycle with no rollover in either direction
  - Usage recorded at Reseller, Dealer, and Rooftop granularity (three-level rollup) for reporting and for splitting bills when Spyne invoices Dealers directly, independent of the wallet computation itself
  - Dealer-direct billing: when Spyne bills each Dealer instead of the Reseller, each Dealer is billed their own actual usage, and any shortfall needed to meet the Reseller's pooled minimum commitment is split across Dealers in proportion to their share of that month's usage
  - Reseller-facing, self-serve VIN target + configurable overage allowance % (default 20%) per Dealer/Rooftop, set in Partner Console, hard-blocking further processing for that Dealer/Rooftop once usage reaches target × (1 + overage%) — entirely independent of and without effect on the Reseller-level wallet/billing computation
  - Combined multi-product requests (e.g. Images + Video tour + 360° Spin submitted together): if only one product has reached its Dealer/Rooftop hard cap, only that product is blocked (with an explanatory message) — the rest of the request proceeds normally
out:
  - Overage billing calculation for VIN-capacity commitments (flagging only, no charge logic)
  - Any new approval/re-signature gate on amendment (none exists today, none is being added)
  - Client-side effective-dating of price changes (see Dependencies — enforced downstream, not in this form)
  - Any minimum commitment held at the Dealer or Rooftop level — the minimum commitment always stays pooled at the Reseller level only, individual Dealers/Rooftops are never held to their own minimum
  - Any validation, reference, or display of the contract's committed VIN count inside the Partner Console's Dealer/Rooftop VIN target UI — that allocation is a fully independent number the Reseller chooses, with no tie to what's on the Spyne contract
cut:
  - Fixed VIN-capacity commitment type (removed after initial build — Commitment is None/Minimum only)
  - Rooftop-based pricing for Studio (explicitly rejected; Studio stays reseller-wide per-VIN)
  - Annual Fee column in the Billing Summary (removed — Monthly Fee is the only recurring figure shown)
  - VIN count as a multiplier on the Advance Breakdown display table for Studio's Usage model (removed — Offered Price shown there is a rate, e.g. `$3/VIN`, not a pre-multiplied sum; VIN count only selects which slab tier applies). This does NOT apply to the actual monthly bill: the wallet/billing engine (AC-9, AC-10) does multiply actual usage by this same rate — the two must not be conflated
  - Total row on the Advance Breakdown table (removed — Studio's rate is per-VIN and Vini's is per-rooftop, so summing them is not meaningful; the table is a rate card, not an invoice)
  - The "(per live Rooftop)" qualifier on the Advance Breakdown table's title (removed — it only ever applied to Vini and would mislabel Studio's reseller-wide rows)
```

## Acceptance Criteria

```productspec-acceptance-criteria
- id: AC-1
  criterion: Given a Studio product has a VIN count entered and a Disc % previously set, when the VIN count changes to a value that falls into a different slab tier, then Cost Price recalculates from the new tier and the same Disc % is reapplied to derive the new Offered Price — the discount is not reset or frozen at the old dollar amount.
- id: AC-2
  criterion: Given any Studio or Vini product row, when the admin edits either Offered Price or Disc %, then the other field recalculates automatically from Cost Price, in both directions, with no ceiling on Offered Price for either product — it may exceed Cost Price and produce a negative Disc %.
- id: AC-3
  criterion: Given Commitment is set to Minimum with a floor amount, when the wallet/billing engine computes the Reseller's actual monthly bill for Studio (not the Advance Breakdown display table, which never totals), then it equals the greater of (sum of actual per-product usage fees) and the floor amount.
- id: AC-4
  criterion: Given Reseller Type is set to Partnership, when the clause list renders, then Non-solicitation, Dealer account protection, Margin freedom, and GTM support show as auto-included and locked (not removable).
- id: AC-5
  criterion: Given the global currency selector is changed, when any Studio or Vini price is displayed anywhere on the page, then all currency symbols update immediately with none hardcoded.
- id: AC-6
  criterion: Given a customer type of Reseller, when Vini's actual monthly bill is computed (a separate calculation from the Advance Breakdown display table, which shows rate only and never totals), then it follows the existing reseller Vini billing computation already implemented in the console, unchanged — this spec introduces no new formula for Vini.
- id: AC-7
  criterion: Given an existing reseller contract with some Studio products enabled, when a sales admin opens it in amendment mode, then previously-unselected products can be newly enabled, previously-enabled products cannot be disabled but their VIN count/Offered Price/Disc % remain editable, and saving creates a new linked contract version rather than mutating the original.
- id: AC-8
  criterion: Given a reseller contract's Commercial Summary is rendered to PDF (new or amended), then Studio and Vini appear as two separate tables, followed by an Additional Terms section listing only the selected clauses, and an amended contract regenerates the full PDF rather than producing a delta document.
- id: AC-9
  criterion: Given a Studio product on the Usage model with a monthly VIN wallet topped up to the contracted amount, when actual usage for the month is less than the committed minimum, then the Reseller is billed the minimum commitment amount and the unused VIN balance rolls over into next month's wallet.
- id: AC-10
  criterion: Given a Studio product on the Usage model, when actual usage for the month exceeds the committed minimum, then the Reseller is billed for actual usage at the contracted rate and no VIN balance rolls over (rollover only ever applies to unused credit, never to overage).
- id: AC-11
  criterion: Given a Studio product on the Monthly model, when the month ends, then the Reseller is billed the fixed monthly fee regardless of actual usage, and the wallet resets with no rollover in either direction.
- id: AC-12
  criterion: Given Spyne bills Dealers directly instead of the Reseller, when the Reseller's total actual usage across all Dealers falls short of the pooled minimum commitment, then each Dealer is billed their own actual usage plus a share of the shortfall proportional to their share of total usage, and the sum of all Dealer invoices equals exactly the Reseller's minimum commitment.
- id: AC-13
  criterion: Given a Reseller has set a VIN target and overage allowance % for a Dealer/Rooftop in Partner Console, when that Dealer/Rooftop's usage reaches target × (1 + overage%), then further processing for that Dealer/Rooftop is blocked until the Reseller raises the limit or the next cycle resets it, and the Reseller's own wallet/billing computation is unaffected either way.
- id: AC-14
  criterion: Given a Rooftop submits Images, Video tour, and 360° Spin together in one combined request and has already reached its Dealer/Rooftop hard cap for Images only, when the request is processed, then only the Image option is blocked (with an explanatory message shown) and Video tour and 360° Spin proceed normally.
- id: AC-15
  criterion: Given the Reseller customer type's Advance Breakdown table (renamed from "Advance Breakdown (per live Rooftop)"), when Studio and Vini rows are rendered, then the Monthly Fee column shows the per-unit rate only for both (e.g. `$3/VIN` for Studio, `$500/Rooftop` for Vini) — never a computed total — and there is no Total row anywhere in the table.
```

## Dependencies

- **Downstream invoicing/billing system**: a Studio price change made via amendment must apply only to service requests from the 1st of the next calendar month onward — requests before that keep the prior price. This console/API only needs to record the amendment as it already does (new versioned contract, dated); the effective-dating enforcement itself is that system's responsibility. Likely owner: the backend service behind the existing `GET /v1/reseller/summary` (User Management) and `GET /api/v1/credit-history` endpoints already used by Partner Console's Credit History feature — that service already pre-computes invoice amounts today, so it is the most likely place the wallet/rollover/minimum-commitment/shortfall-split math described above needs to live or be validated against.
- **Existing "Product Credit Partner" ledger** (Partner Console, `apps/partners`): there is already a live per-product credit system (separate pools for `vins`/`images`/`videos`/`threesixtys`, each with current/allocated/purchased units and a debit/credit transaction history) that is structurally very close to the Studio VIN wallet described here. Worth confirming with that system's owners whether Studio's wallet should extend this existing ledger rather than building a parallel one — and specifically whether its transactions' `expiryDate` field means unused credit currently **expires**, which would directly conflict with the **rollover** behavior required here (AC-9). This conflict is not yet resolved.
- **Rooftop-facing submission UI**: AC-14 assumes a screen where a Rooftop can submit a combined Images/Video/360° request and see the Image option specifically disabled — this UI has not yet been identified/confirmed to exist as described.

## Success Metrics

```productspec-success-metrics
- id: SM-1
  metric: avg_studio_contract_turnaround_time
  target: "reduced by 50% vs. Word baseline"
  window: 30 days post-launch
- id: SM-2
  metric: studio_deals_created_via_console (adoption)
  target: "100% of new Studio deals"
  window: 90 days post-launch
```
