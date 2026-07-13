---
spec_format_version: "0.1"
title: "Studio-Contract"
artifact_type: "prd"
spec_revision: 1
author: "ProductSpec.io"
created_at: "2026-07-09T00:00:00Z"
updated_at: "2026-07-09T00:00:00Z"
---

## Problem

Sales admins currently configure Studio pricing for reseller contracts through the manual flow. This causes a no traces of contract in our database which in further difficult to manage Reseller Console including reporting and auto bill generation.

## Hypothesis

If admins configure Studio pricing (products, VIN-tier slabs, commitment, add-ons) through a structured console form instead of Word, pricing errors will drop and contracts will accurately reflect billing math, because slab lookups, bi-directional discount calculation, and fee totals are enforced by the system instead of manual arithmetic.

## Scope

```productspec-scope
in:
  - Usage vs. Monthly pricing model selection, mutually exclusive
  - 5 independent products (Images, 360° Spin, Video tour, Studio Instant, Studio Promote), each with its own VIN-tier rate slab — no shared "Lite/Pro" plan bundling
  - Bi-directional Offered Price ↔ Disc % (editing either recalculates the other from Cost Price), for both Studio and Vini products
  - Optional Commitment: None or Minimum (floor amount + billing frequency)
  - Add-ons (White label, Quality check, Custom backgrounds) and Add-on products (Studio Instant, Studio Promote) with independent enable/disable
  - Studio Integration Fee note + waiver checkbox (informational, non-blocking)
  - Reseller Type (Partnership/SaaS) and Clause review, surfaced in the Terms & Conditions section, always visible (not progressively gated)
  - Unified Billing Summary combining Studio (reseller-wide) and Vini (per-rooftop) with an adaptive Basis column, plus one combined Total
  - Global currency selector propagates to every price symbol across Studio and Vini
out:
  - Overage billing calculation for VIN-capacity commitments (flagging only, no charge logic)
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
  criterion: Given any Studio or Vini product row, when the admin edits either Offered Price or Disc %, then the other field recalculates automatically from Cost Price, in both directions.
- id: AC-3
  criterion: Given Commitment is set to Minimum with a floor amount, when the Billing Summary computes Studio's Monthly Fee, then it equals the greater of (sum of selected product fees) and the floor amount.
- id: AC-4
  criterion: Given Reseller Type is set to Partnership, when the clause list renders, then Non-solicitation, Dealer account protection, Margin freedom, and GTM support show as auto-included and locked (not removable).
- id: AC-5
  criterion: Given the global currency selector is changed, when any Studio or Vini price is displayed anywhere on the page, then all currency symbols update immediately with none hardcoded.
- id: AC-6
  criterion: Given a Vini product has Rooftops and Offered Price set, when the Billing Summary Total is computed, then Vini's contribution equals Rooftops × Offered Price, not the raw per-rooftop rate alone.
```

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
