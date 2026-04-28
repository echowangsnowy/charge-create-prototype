# Research — Create Charge System

> A snapshot of the Create Charge experience in Mainstay's Ops Hub: what's there today, what's already been improved, what's still missing, and what would need to be built. Intended as ramp-up material for anyone (designer, engineer, PM) coming to this project fresh.

This doc complements the prototype and the technical brief:
- **Prototype:** https://echowangsnowy.github.io/charge-create-prototype/
- **TECH_BRIEF.md:** action-oriented integration plan
- **This file:** descriptive system survey + gap analysis

---

## 1. Why this project exists

Charges are the unit through which Mainstay bills HOAs on behalf of single-family rental customers (FirstKey Homes, Evergreen Residential, Invitation Homes, etc.). Operators create charges manually through the Ops Hub at `/charge/create`.

**The problem (from the Ops perspective):** The current create form is permissive — it accepts almost anything an operator types. Subtle errors (wrong category, wrong amount, missing context in notes, duplicate charges) are easy to make and only surface later as downstream remediation work. The Charges Accuracy & Funding Request Improvement project aims to add **guardrails at the point of creation** so fewer errors enter the system in the first place.

This research and prototype focus on the *creation* experience. Editing, voiding, and review live in adjacent surfaces and are referenced where they intersect.

---

## 2. Current state of Create Charge

### 2.1 The form today

Location: `/charge/create` — implemented at `apps/hoa/hoa-web/apps/admin/src/routes/specific/charge/create.tsx` (route entry) and `.../charge/form/createEdit.tsx` (the actual form fields).

**Fields, in order:**
1. Property (typeahead, required)
2. HOA (combobox, required) — populated from `property.property_hoas`
3. Amount (cents-aware text input, required)
4. Customer Approved Amount (cents-aware text input, optional)
5. Category + Subcategory (cascading comboboxes, required)
6. Notice Date + Due Date (date pickers, required; due date defaults to notice + 30 days)
7. Linked Violation (combobox, optional)
8. Evidence (single file upload)
9. Notes (free-text, required)
10. Customer Visible (boolean toggle)

Edit-only fields appear when editing an existing charge: Funding Request, Is Void, Void Reason.

### 2.2 The submit flow

Form data is sent as `multipart/form-data` to `POST /charge/` (Django, `apps/hoa/api/admin_api/viewsets/charges.py`). The backend creates a `Charge` record, attaches any uploaded evidence file, sets `scheduled_mailing_date` to today, and returns `{ status: "success", pk: <uuid> }`.

A duplicate-detection check (`useCheckDuplicateCharges`) runs before submit and, when it finds likely duplicates, surfaces a confirmation modal (`DuplicateChargeModal`).

A separate "violation fine without linked violation" check runs the same way (`checkViolationFine` in `create.tsx`).

### 2.3 Key data models (Django)

| Model | Purpose | Notable fields |
|---|---|---|
| `Charge` | The created charge | `property`, `hoa`, `amount_cents`, `charge_category`, `charge_subcategory`, `notice_date`, `due_date`, `notes`, `is_void`, `void_reason`, `customer_approved_amount_cents`, `is_customer_visible` |
| `Portfolio` | The "customer" (FirstKey, Evergreen, etc.) | `name`, `charge_default_customer_visible`, `allowed_charge_categories` (M2M, recently added) |
| `Property` | A single SFR property | `portfolio` (FK), `property_hoas` (reverse FK) |
| `PropertyHOA` | Join between property and HOA | `property`, `hoa`, `relationship_type` (primary / master / other) |
| `Organization` | An HOA | `name`, `state`, `is_build_to_rent`, `is_master_hoa`, `management_company` |
| `ChargeCategory` | Top-level category | `id` (integer), `name` |
| `ChargeSubCategory` | Nested under a category | `id`, `name`, `category` (FK), `deleted_at` (soft-delete) |
| `PaymentSchedule` | Recurring expected charges per property+HOA | `schedule_type`, `frequency`, `payment_config.payment_months` (cents), `is_active` |

### 2.4 Endpoints used by the form

| Endpoint | Purpose |
|---|---|
| `GET /property/search/` | Typeahead for property |
| `GET /property/<id>/` | Property detail incl. portfolio + HOAs |
| `GET /charge/categories/?portfolio_id=X` | Categories with nested subcategories, optionally filtered by portfolio |
| `GET /charge/payment_schedule/?property_id=X&hoa_id=Y` | Recurring schedules on file |
| `GET /violation/?property_id=X` | Linkable violations |
| `POST /charge/check_duplicates/` | Duplicate detection |
| `POST /charge/` | Create charge (multipart with evidence) |

### 2.5 Customer (Portfolio) identity

Each `Property` belongs to exactly one `Portfolio`. The Portfolio is what we colloquially call "the customer" — FirstKey Homes, Evergreen Residential, etc. There's no separate "customer" model — Portfolio *is* the customer.

Real Portfolio UUIDs (for engineers reproducing this locally):
- FirstKey Homes: `943295f6-9543-4035-9e2f-38f8a1b5be7e`
- Evergreen Residential: `1c505537-e8a4-495a-bc14-329f10349d74`
- Roots: `e6e49ead-d2f6-45cb-bc3f-5a88262f0c12`
- Flock Homes: `a5ec0146-c15d-419e-b68d-5b1fb2e80fd1`

---

## 3. What's already been built (and is in flight on `echo/charge-create-redesign`)

### 3.1 Foundation (committed earlier, prior session)

| Change | Where | What it does |
|---|---|---|
| `Portfolio.allowed_charge_categories` M2M field | `apps/hoa/api/models.py` | Stores per-portfolio list of permitted top-level categories |
| Migration 0292 | `apps/hoa/api/migrations/0292_portfolio_allowed_charge_categories.py` | Applies the schema change |
| Categories endpoint filter | `apps/hoa/api/admin_api/viewsets/charges.py` | When `portfolio_id` is passed, returns only the allowed categories. Empty allow-list = no restriction. |
| New `GET /charge/payment_schedule/` endpoint | Same viewset | Returns `{ current: [...], future: [...] }` payment schedules for a property + HOA pair |
| `useChargeCategories(selectedCategoryId, portfolioId)` | Frontend hook | Now passes `portfolio_id` through to the API |
| `Categories` component accepts `portfolioId` | Frontend component | Wired through from the form |
| `createEdit.tsx` passes `property.portfolio.id` | Frontend form | The form auto-scopes categories to the property's portfolio |

### 3.2 Payment schedule integration (committed in this session)

| Change | Where | What it does |
|---|---|---|
| `usePaymentSchedule(propertyId, hoaId)` | `.../charge/hooks/usePaymentSchedule.ts` (new) | Fetches the schedule list, formats items as ComboBox options, exposes a per-period amount helper |
| `PaymentSchedule` component | `.../charge/components/PaymentSchedule.tsx` (new) | Dropdown UI that emits `{ item, amountCents }` on selection |
| `Categories.onCategoryChange` callback | `.../charge/components/Categories.tsx` | Lets parent forms react when category changes (used to show/hide the schedule dropdown) |
| Form wiring | `.../charge/form/createEdit.tsx` | Renders the `PaymentSchedule` only when category = "Dues"; selecting an item prefills the Charge Amount field |

### 3.3 Net effect today on the branch

When an operator on `echo/charge-create-redesign`:
1. Picks a property → form auto-scopes categories to that property's portfolio
2. Picks an HOA + Dues category → a Payment Schedule dropdown appears with the property's recurring expected charges
3. Picks a schedule item → Charge Amount field auto-fills with the per-period price
4. Operator can override the amount manually

For non-Dues categories or properties without payment schedules, the form behaves exactly as it does today — no regressions.

**No changes have been pushed.** The branch lives only on Echo's local checkout of the monorepo.

---

## 4. What's still missing

The prototype demonstrates several additional guardrails that are **not yet built**. Each is marked with backend / frontend / both depending on what the lift is.

### 4.1 Subcategory-level filtering for portfolios `[backend + tiny frontend]`

**Problem:** `Portfolio.allowed_charge_categories` filters at the *category* level only. FirstKey's needs go finer — e.g., within "Fees & Fines," they only want 6 of the 12 subcategories visible (not Closing Fee, Internet Fee, etc.).

**Proposed:**
- Add `Portfolio.allowed_charge_subcategories` M2M field (parallel to existing field)
- Migration to add the column
- Update `GET /charge/categories/` to also pass `portfolio.allowed_charge_subcategories` IDs to the serializer context, and have `ChargeCategoryWithSubSerializer.get_subcategories` filter accordingly
- Frontend: zero changes — the API will simply hand back fewer subcategories

**Status:** Not started. Was started in this session, then reverted to keep the branch clean while we paused for this research doc.

### 4.2 Annualized vs. quarterlized display `[backend + frontend]`

**Problem:** FirstKey thinks about dues annualized; other partners think about them quarterlized. The same `$275/quarter` schedule should display as `$1,100 (annualized)` for FK and `$275 (quarterlized)` for others — and the *saved* amount needs to match what's displayed.

**Proposed:**
- Add `Portfolio.annualization_view` CharField with choices `('annualized', 'quarterlized')` and default `'quarterlized'`
- Data migration to set FirstKey to `'annualized'`
- Frontend: read the field, multiply/divide accordingly when rendering the Payment Schedule dropdown and prefilling the amount

**Open product question:** Does the *saved* `amount_cents` change based on this view, or is it display-only and the saved amount is always the per-period amount? This needs a Finance/Product decision before implementation. The prototype currently treats the displayed amount as the saved amount (so FK saves $1,100 for an annualized quarterly dues charge).

**Status:** Not started.

### 4.3 Auto-populated note templates `[frontend, no backend]`

**Problem:** Notes today are free-text. Different categories/subcategories require different formats (e.g., `Annualized Dues: MM/DD/YYYY - MM/DD/YYYY`, `Violation Fine: MM/DD/YYYY - MM/DD/YYYY - [violation category]`). Operators have to remember and manually type them.

**Proposed:**
- A `noteTemplates` config map keyed by `(category, subcategory)`
- When category/subcategory + dates are selected, prefill the Notes field with the rendered template
- Operator can edit the prefilled text

**Where templates live:** Three options:
- **A** — In a code config file (`noteTemplates.ts`). Cheap. Each spec change requires a deploy.
- **B** — A new `ChargeNoteTemplate` table editable by Ops. Self-serve but needs an admin UI.
- **C** — A feature-flagged JSON loaded at runtime. Hot-swappable without deploy.

Recommendation in the prototype: start with A, migrate to B when Ops wants self-serve editing. The current Ops template table is a known artifact — it can be encoded directly.

**Status:** Not started.

### 4.4 3-step wizard `[frontend, no backend]`

**Problem:** The current form is a single page — operators see everything at once. The prototype proposes splitting into Property & Charge → Details & Evidence → Review & Submit so cognitive load is lower and the review step catches errors before submission.

**Open product question:** Is the wizard cost worth it, or would a single-page form with collapsible sections accomplish most of the goal? The prototype's split is one option, not a hill to die on.

**Status:** Not started.

### 4.5 Insufficient-funds warning `[frontend, no backend]`

**Problem:** When the customer-approved amount is less than the charge amount, today's form silently lets it submit. The prototype adds a real-time inline warning + a banner on the review step explaining that the charge will need additional funding approval.

**Worth flagging:** Echo isn't sure this is actually a problem operators run into today. Worth confirming with Ops before investing UI in it.

**Status:** Not started.

### 4.6 Void toggle on creation `[frontend, no backend]`

**Problem:** `Charge.is_void` and `Charge.void_reason` exist as fields, but voiding currently only happens *after* creation as a separate edit action. The prototype proposes letting an operator mark a charge as void at the moment of creation (e.g., for known duplicates that still need an audit trail).

**Open product question:** Is this the right model, or does it conflict with the audit/workflow assumption that void is a post-creation action?

**Status:** Not started.

### 4.7 Drag-and-drop evidence upload `[frontend, no backend]`

**Problem:** Today's evidence upload is a basic file input. The prototype shows a drag-and-drop zone with file previews and (potentially) multiple-file support.

**Open question:** Does `POST /charge/` currently accept multiple evidence files in one request, or only one? The list serializer returns `evidence: Array<...>` so the model can hold many, but the create endpoint may only take one.

**Status:** Not started.

### 4.8 Review-before-submit summary `[frontend, no backend]`

**Problem:** Today's form submits straight from the single page. The prototype adds a read-mostly summary step that displays everything entered with clear "Edit" affordances jumping back to the relevant step.

**Status:** Not started.

---

## 5. Open product questions

These need product/finance/Ops alignment, not engineering decisions:

1. **Subcategory filtering ownership** — Is per-portfolio subcategory filtering Ops-managed (admin UI to maintain) or product-managed (deploys when it changes)?
2. **Annualization saved-amount semantics** — Does `amount_cents` store the annualized value or the per-period value when FK has `annualization_view='annualized'`?
3. **Note template ownership** — Same question as #1 but for note templates. Self-serve for Ops, or product-managed?
4. **Wizard vs. progressive form** — Is a 3-step wizard the right pattern, or is a single-page form with sections sufficient?
5. **Void at creation** — Should creation-time void be allowed, or stay a post-creation action?
6. **Insufficient-funds reality check** — Is this an actual operator pain point today, or anticipated?
7. **Evidence multi-upload** — Is this a real ask or nice-to-have?

---

## 6. What changed during the design exploration

The prototype evolved iteratively as Ops thinking sharpened. Notable shifts captured here for context:

- **Initial assumption:** Payment Schedule data didn't exist → it does, and a `/charge/payment_schedule/` endpoint was added in the prior session.
- **Initial assumption:** Schedules show recurring frequencies (Monthly/Quarterly/Annual). After clarification: schedules are *line items* (Dues, Special Assessment, Lawn Maintenance, Pool Fob), each with its own frequency and amount.
- **Initial assumption:** Auto-fill should happen for any category. After clarification: auto-fill is meaningful only for Dues; non-Dues categories should leave the amount blank.
- **Annualization math:** ×4 for quarterly → annual (initial typo "275×5" was confirmed as ×4).
- **Customer Approved Amount:** Defaults to blank, not zero, to match the prevailing convention in the current form.
- **Funding warning wording:** Reframed from "funding request may be needed" to "insufficient approved funds" because the latter is what operators actually need to see.
- **Charge Voided on review:** Added explicit Yes/No row on the review summary so void state is unmistakable.

---

## 7. Where to take this next

**For design refinement:** The prototype currently uses a hand-coded HTML/CSS approximation of the existing Ops Hub style. Visual polish, accessibility, and Mainstay design-system alignment would benefit from a designer's pass — best done in a design tool (Figma, Claude design, etc.) rather than continuing to hand-edit the prototype HTML.

**For engineering:** TECH_BRIEF.md outlines a guardrail-by-guardrail integration plan. The branch `echo/charge-create-redesign` in `mainstay-io/monorepo` (local only, not pushed) has the foundation + payment schedule integration committed and ready for review.

**For product:** The seven open questions in §5 are gating product decisions. None can be fully answered by engineering or design alone — they need Ops + Finance + Product alignment.

---

## 8. File map (quick reference)

### Frontend (existing, modified, or new on the branch)
```
apps/hoa/hoa-web/apps/admin/src/routes/specific/charge/
├── create.tsx                          # entry point, owns submit + duplicate check
├── form/createEdit.tsx                 # all input fields (modified on branch)
├── components/
│   ├── Categories.tsx                  # category/subcategory cascading select (modified)
│   ├── PaymentSchedule.tsx             # NEW — schedule dropdown
│   ├── Violations.tsx
│   └── DuplicateChargeModal.tsx
└── hooks/
    ├── useChargeCategories.ts          # category fetcher (modified for portfolioId)
    └── usePaymentSchedule.ts           # NEW — schedule fetcher
```

### Backend
```
apps/hoa/api/
├── models.py                           # Portfolio, ChargeCategory, ChargeSubCategory, PaymentSchedule
├── admin_api/
│   ├── viewsets/charges.py             # /charge/ + /charge/categories/ + /charge/payment_schedule/
│   └── serializers/
│       ├── serializers.py              # ChargeCategoryWithSubSerializer
│       └── payment_schedule.py         # PaymentScheduleSerializer
├── actions/charge.py                   # ChargeActions.create
└── migrations/
    └── 0292_portfolio_allowed_charge_categories.py
```

---

*Last updated: April 2026. Prototype: [github.com/echowangsnowy/charge-create-prototype](https://github.com/echowangsnowy/charge-create-prototype). Branch: `echo/charge-create-redesign` in `mainstay-io/monorepo` (local only).*
