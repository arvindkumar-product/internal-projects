# PRD — Studio Contracting (Section 3)

**Status:** Ready for engineering
**Scope:** Reseller contract creation console — Section 3 (Studio AI pricing), the Reseller Type/Clauses block that now lives in Section 7, and their interaction with Section 4 (Vini AI) and the Billing Summary.
**Not covered here:** Sections 1, 2, 5, 6, and the rest of Section 7 (Contract Term/Lock-in/Termination/Cooloff — unchanged), plus PDF/DocuSign generation — see the base PRD for those.

This document describes **behavior**, not layout. Any UI can implement it as long as the rules below hold.

---

## 1. Goal

Let a sales admin configure Studio pricing for a reseller — pricing model, per-product rates, commitment, add-ons, integration fee — and have it combine correctly with Vini's existing rooftop pricing into one accurate Billing Summary.

---

## 2. Products

Five independent products. No "Studio Lite / Pro" plan grouping — each is priced and configured on its own:

- **Core:** Images, 360° Spin, Video tour
- **Add-on products:** Studio Instant, Studio Promote

Each product has its **own independent VIN-tier rate table** (slab). Changing one product's rates must never affect another's.

---

## 3. Pricing Model (choose one, gates everything else)

### 3a. Usage
Priced per VIN, per product, individually selectable.

For each selected product, the admin enters a monthly VIN count. The system:

1. Looks up the tier that VIN count falls into on that product's slab → **Cost Price** (a flat per-VIN rate, e.g. `$3.00`, not multiplied by VIN count).
2. Defaults **Offered Price** to Cost Price.
3. Computes **Disc %** = `(Cost Price − Offered Price) / Cost Price × 100`.

**Offered Price and Disc % are bi-directional** — editing either one recalculates the other from Cost Price. This must work in both directions everywhere it appears (Studio products here, and Vini products in Section 4).

**Offered Price has no ceiling.** It may exceed Cost Price — that just produces a negative Disc % (a markup). Do not clamp it back down.

**Discount is sticky across VIN-count changes.** If an admin sets a 10% discount at 150 VINs, and later changes the VIN count to a value that falls into a different slab tier, the system re-applies **the same 10%** to the new tier's Cost Price — it does not silently reset the discount or freeze the old dollar amount. Defined disc % should be applicable for the whole slab of that product price slab

**Monthly line-item fee** for a product = `Offered Price`. VIN count is **not** a multiplier here — it only determines which slab tier sets the Cost Price. Offered Price at that tier is itself the flat monthly fee for the product.

### 3b. Monthly
Flat monthly amount per selected product, entered directly by the admin. VIN count here is a **capacity control only** — it does not drive billing, it exists so the system can flag overage if actual usage later exceeds it.

---

## 4. Commitment — Usage model only, optional

A single control: **None / Minimum.**

| Choice | Behavior |
|---|---|
| None | No floor or cap. Total = sum of product line-item fees. |
| Minimum | Admin sets a floor amount + billing frequency. Total = `max(sum of product line-item fees, floor amount)`. |

Switching back to **None** clears the commitment entirely — it is not mandatory to pick one.

---

## 5. Additional Feature — independent of pricing model

Three toggles, shown for both Usage and Monthly:

| Add-on | Fields | Billing nature |
|---|---|---|
| White label app + console | Setup fee (default €8,000), payment schedule | One-time |
| Quality check (human review) | QC rate/image, manual edit rate/image | Variable, billed as-per-actual — cannot be forced into a fixed monthly figure |
| Custom backgrounds | Free backgrounds included, cost per additional | Variable, billed as-per-actual |

None of these feed into the recurring Monthly/Annual Fee totals (they're one-time or usage-variable) — surface them as a separate note/list wherever the Billing Summary is shown.

---

## 6. Studio Integration Fee

Separate from the White label setup fee. A flat note — **"charged as per actual"** — plus a waive checkbox with the confirmation text: *"I confirm that customer agrees to a 6/12 month lock-in period OR does 6/12 months advance payment."*

This is shown in Section 3 once a pricing model is selected. It is **informational, not a gate** — the admin can continue configuring the rest of the contract without interacting with it.

---

## 7. Reseller type → clause set

Lives inside **Section 7 (Terms & Conditions)**, not Section 3 — positioned after Contract Term / Lock-in / Termination Days / Cooloff. It is **always visible there, not progressively gated** — the admin sees Reseller Type and the Clause list immediately, in that fixed order, regardless of whether the other Section 7 fields are filled in yet.

One choice:

| Answer | Template | Clause behavior |
|---|---|---|
| Yes — invoices dealers independently | Partnership Agreement | Non-solicitation, dealer account protection, margin freedom, GTM support: **auto-included, locked** |
| No — processes for own operation | SaaS Agreement | Non-solicitation: **on by default, removable.** Everything else optional. |

All other clauses (territory & exclusivity, MFN pricing, advance deposit, credit/image rollover, quality rebate, price escalation) are optional toggles for either template, off by default.

---

## 8. Currency

One global currency selector (Section 1). Every `$`/currency symbol shown anywhere in Studio **and** Vini pricing — headers, Billing Summary — must update immediately when this changes. Nothing should be hardcoded to a specific symbol.

---

## 9. Billing Summary — combining Studio + Vini

Studio (per-VIN, reseller-wide) and Vini (per-rooftop) are structurally different. **Do not force Studio into a rooftop model** — that was explicitly ruled out for this release. Instead, use one shared table with an adaptive basis:

| Column | Studio row | Vini row |
|---|---|---|
| Product | Product name | Product name |
| Basis | `{VIN count} VINs/mo (reseller-wide)` | `{Rooftops} × {usage/rooftop} @ {rate}/rooftop` |
| Monthly Fee | `Offered Price` (flat — VIN count only picked the tier, see §3a) | `Rooftops × Offered Price per rooftop` |
| Discount Applied | Product's Disc % | Product's Disc % |

There is no Annual Fee column — Monthly Fee is the only recurring figure shown.

**Total row** = sum of Monthly Fee across every checked/selected row (Studio + Vini).

Footnote text: *"One-time Integration Fee will be charged as per actuals, unless waived above. Any taxes will be levied extra, as per applicable laws."*

---

## 10. Vini AI — one behavioral difference

Vini's own Cost Price / Offered Price / Disc % must also be bi-directional (§3a applies here too). Unlike Studio, **Vini's Offered Price stays clamped** — it cannot exceed Cost Price, and Disc % is clamped to 0–100%. (Studio intentionally removed this ceiling; Vini did not.)

Vini's Monthly Fee is genuinely rooftop-scaled (`Rooftops × Offered Price`) — this is different from Studio's flat-per-tier Monthly Fee (§3a) and must not be confused with it.

---

## 11. Data contract (indicative)

```json
{
  "studio": {
    "pricing_model": "usage | monthly",
    "currency": "USD | EUR | GBP | INR",
    "products": [
      {
        "id": "images | 360 | video | instant | promote",
        "selected": true,
        "vins_per_month": 150,
        "cost_price": 3.00,
        "offered_price": 2.70,
        "disc_pct": 10.0,
        "monthly_fee": 2.70
      }
    ],
    "commitment": { "type": "none | min", "amount": null, "billing_frequency": null },
    "addons": {
      "white_label": { "enabled": false, "setup_fee": 8000, "payment_schedule": "" },
      "quality_check": { "enabled": false, "qc_rate": 0.03, "manual_edit_rate": 0.10 },
      "custom_backgrounds": { "enabled": false, "free_included": 10, "cost_per_additional": 29 }
    },
    "integration_fee_waived": false
  },
  "reseller_type": "partnership | saas",
  "clauses": [{ "id", "name", "auto_included", "enabled", "fields": {} }],
  "vini": [{ "product", "rooftops", "usage_per_rooftop", "cost_price", "offered_price", "disc_pct", "monthly_fee" }]
}
```

`studio.products[].monthly_fee` = `offered_price` directly (no VIN multiplier). `vini[].monthly_fee` = `rooftops × offered_price`.

---

## 12. Out of scope — v1

- Overage billing calculation for Fixed VIN-capacity commitment (removed — Studio Commitment is now None/Minimum only, see §4)
- Rooftop-based pricing for Studio (explicitly rejected for this release)
- DocuSign trigger, contract amendments/renewals, exclusive-territory conflict detection — separate specs

## 13. Open questions

| # | Question |
|---|---|
| 1 | Should Quality check / Custom backgrounds ever get a projected monthly estimate, or stay as-per-actual only? |
| 2 | Governing law default for non-US resellers (currently Delaware) — confirm with Legal. |
