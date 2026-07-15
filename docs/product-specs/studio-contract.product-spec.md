---
spec_format_version: "0.1"
title: "Studio-Contract"
artifact_type: "prd"
spec_revision: 2
author: "ProductSpec.io"
created_at: "2026-07-09T00:00:00Z"
updated_at: "2026-07-15T00:00:00Z"
---

## Problem

Sales admins currently configure Studio pricing for reseller contracts through the manual flow. This causes a no traces of contract in our database which in further difficult to manage Reseller Console including reporting and auto bill generation.

## Hypothesis

If admins configure Studio pricing (products, VIN-tier slabs, commitment, add-ons) through a structured console form instead of Word, pricing errors will drop and contracts will accurately reflect billing math, because slab lookups, bi-directional discount calculation, and fee totals are enforced by the system instead of manual arithmetic.

## Product Summary

A console form lets a sales admin configure Studio pricing for a reseller — choose Usage or Monthly pricing, set VIN-tier rates and discounts per product, pick a commitment and add-ons — and see it combine with Vini's pricing into one accurate Billing Summary, all without touching a Word document. This covers the full contract lifecycle for reseller customers: create, save as draft, generate the contract PDF, and amend an existing contract.

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
  - Unified Billing Summary combining Studio (reseller-wide) and Vini (per-rooftop) with an adaptive Basis column, plus one combined Total
  - Global currency selector propagates to every price symbol across Studio and Vini
  - Draft save/resume for a Studio-priced reseller contract, using the existing draft/prefill mechanism unchanged
  - Amendment of an existing reseller contract's Studio pricing, using the existing amendment mechanism unchanged (new linked contract version per amendment; no new approval/re-signature step). A Studio product not previously selected may be newly enabled on amendment; a previously-enabled product cannot be disabled but its VIN count/Offered Price/Disc % remain editable
  - Contract PDF: Commercial Summary renders Studio and Vini as two separate tables (not merged), followed by an Additional Terms section listing selected clauses. Amendment PDFs are a full regenerated document (not a delta/addendum), consistent with the existing PDF pipeline
out:
  - Overage billing calculation for VIN-capacity commitments (flagging only, no charge logic)
  - Any new approval/re-signature gate on amendment (none exists today, none is being added)
  - Client-side effective-dating of price changes (see Dependencies — enforced downstream, not in this form)
cut:
  - Fixed VIN-capacity commitment type (removed after initial build — Commitment is None/Minimum only)
  - Rooftop-based pricing for Studio (explicitly rejected; Studio stays reseller-wide per-VIN)
  - Annual Fee column in the Billing Summary (removed — Monthly Fee is the only recurring figure shown)
  - VIN count as a billing multiplier for Studio's Usage model (removed — Offered Price is the flat per-tier fee; VIN count only selects which slab tier applies)
```

## Acceptance Criteria

```productspec-acceptance-criteria
- id: AC-1
  criterion: Given a Studio product has a VIN count entered and a Disc % previously set, when the VIN count changes to a value that falls into a different slab tier, then Cost Price recalculates from the new tier and the same Disc % is reapplied to derive the new Offered Price — the discount is not reset or frozen at the old dollar amount.
- id: AC-2
  criterion: Given any Studio or Vini product row, when the admin edits either Offered Price or Disc %, then the other field recalculates automatically from Cost Price, in both directions, with no ceiling on Offered Price for either product — it may exceed Cost Price and produce a negative Disc %.
- id: AC-3
  criterion: Given Commitment is set to Minimum with a floor amount, when the Billing Summary computes Studio's Monthly Fee, then it equals the greater of (sum of selected product fees) and the floor amount.
- id: AC-4
  criterion: Given Reseller Type is set to Partnership, when the clause list renders, then Non-solicitation, Dealer account protection, Margin freedom, and GTM support show as auto-included and locked (not removable).
- id: AC-5
  criterion: Given the global currency selector is changed, when any Studio or Vini price is displayed anywhere on the page, then all currency symbols update immediately with none hardcoded.
- id: AC-6
  criterion: Given a customer type of Reseller, when the Billing Summary Total is computed, then Vini's contribution follows the existing reseller Vini billing computation already implemented in the console, unchanged — this spec introduces no new formula for Vini.
- id: AC-7
  criterion: Given an existing reseller contract with some Studio products enabled, when a sales admin opens it in amendment mode, then previously-unselected products can be newly enabled, previously-enabled products cannot be disabled but their VIN count/Offered Price/Disc % remain editable, and saving creates a new linked contract version rather than mutating the original.
- id: AC-8
  criterion: Given a reseller contract's Commercial Summary is rendered to PDF (new or amended), then Studio and Vini appear as two separate tables, followed by an Additional Terms section listing only the selected clauses, and an amended contract regenerates the full PDF rather than producing a delta document.
```

## Dependencies

- **Downstream invoicing/billing system** (not part of this console, not yet located by the product owner as of 2026-07-15): a Studio price change made via amendment must apply only to service requests from the 1st of the next calendar month onward — requests before that keep the prior price. This console/API only needs to record the amendment as it already does (new versioned contract, dated); the effective-dating enforcement itself is that system's responsibility and must be confirmed with its owning team before this ships. This is the one open item not resolved by this spec.

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
