# Create Charge — Redesign Mock-up

A clickable mock-up of a redesigned Create Charge flow for Mainstay Ops Hub, produced as part of the **Charges Accuracy & Funding Request Improvement** project.

> This is a mock-up from an Ops lens — not a spec or a design ask. It's a way to make our hypotheses concrete so we can have a better conversation with EPD.

## 🔗 Live prototype

- **Annotated:** https://echowangsnowy.github.io/charge-create-prototype/
- **Clean UI:** https://echowangsnowy.github.io/charge-create-prototype/?clean=1

Toggle between the two views using the button in the bottom-right corner.

## What it is

A single self-contained HTML file (~1600 lines, no build step, no dependencies) that walks through a 3-step charge creation flow:

1. **Property & Charge** — property, HOA, category/subcategory, payment schedule, charge amount
2. **Details & Evidence** — dates, violation link, evidence upload, notes, void toggle, visibility
3. **Review & Submit** — editable summary with warnings before submission

## Guardrails explored

Each of these represents an Ops hypothesis about what could reduce charge creation errors:

- **Customer-specific category/subcategory filtering** — e.g., FirstKey sees only Dues and Fees & Fines; Evergreen adds Adjustments & Miscellaneous
- **Payment Schedule dropdown** — pulls the property's recurring expected charges and prefills the charge amount
- **Annualized vs. quarterlized amounts** — FirstKey sees annualized (×4 for quarterly, ×12 for monthly), other partners see quarterlized — based on how each partner thinks about dues
- **Auto-populated note templates** — match the Ops standard format per category/subcategory (e.g., `Annualized Dues: MM/DD/YYYY - MM/DD/YYYY`)
- **Review-before-submit step** — editable summary to catch errors before they hit the ledger
- **Insufficient-funds warning** — surfaces inline and on review when the customer-approved amount won't cover the charge
- **Void toggle + void notes** — explicit, discoverable, and reflected on the review summary
- **Drag-and-drop evidence upload** with previews

## Running locally

No build step. Either:

```bash
# Option A: open directly in a browser
open index.html

# Option B: serve over localhost (recommended for full functionality)
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Integration points

All data in the mock is fake. The state that would drive real behavior is concentrated in three JS objects near the top of the `<script>` block — those are the swap points for real API calls:

| Object | What it holds |
|---|---|
| `hoaData` | HOA list + each property's payment schedule line items |
| `categoryData` | Customer-scoped categories and subcategories |
| `noteTemplates` | Per-category/subcategory note format strings |

## Context

Built by [Echo Wang](https://github.com/echowangsnowy) (Ops Design & Strategy, Mainstay), April 2026.
