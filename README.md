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

## Companion docs

- **[RESEARCH.md](./RESEARCH.md)** — descriptive system survey: what Create Charge looks like in Mainstay today, what foundation work has already landed, what's still missing, and where to take the project next. Best starting point for anyone arriving fresh.
- **[TECH_BRIEF.md](./TECH_BRIEF.md)** — action-oriented integration plan: which existing components/endpoints each guardrail would plug into, what backend changes are needed, and the open product questions to resolve before implementation.

## Integration points

All data in the mock is fake, but the **mock data structures match the real API shapes** in the monorepo. Each block in the `<script>` carries a comment pointing to its real source file. The main swap points:

| Object | Mocks | Real source |
|---|---|---|
| `ALL_CATEGORIES` | `GET /charge/categories/` response | `apps/hoa/api/admin_api/serializers/serializers.py` |
| `PORTFOLIOS` | Portfolio model + customer config | `apps/hoa/api/models.py` |
| `hoaData[*].paymentSchedule` | `GET /charge/payment_schedule/` items | `apps/hoa/api/admin_api/serializers/payment_schedule.py` |
| `noteTemplates` | (No backend equivalent — see TECH_BRIEF) | — |

Amounts in the mock are stored as **cents** to match the backend's `amount_cents` convention.

## Context

Built by [Echo Wang](https://github.com/echowangsnowy) (Ops Design & Strategy, Mainstay), April 2026.
