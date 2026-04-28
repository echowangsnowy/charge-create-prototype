# Technical Brief — Create Charge Redesign

> Companion to the [Create Charge prototype](https://echowangsnowy.github.io/charge-create-prototype/). Maps each guardrail in the prototype to existing patterns in the monorepo, identifies backend gaps, and surfaces open questions for EPD.

This brief is **not a spec**. It's an attempt by Ops to make integration thinking concrete based on a recon of the monorepo. Treat assumptions as hypotheses to pressure-test, not facts.

---

## TL;DR

- **Most of the data already exists** in the monorepo: payment schedule, charge categories, portfolio config, HOA, violations, duplicate detection. Few new endpoints needed.
- **The prototype's mock data structures match real API shapes** — categories use integer IDs with nested subcategories, payment schedules use `payment_config.payment_months` in cents, etc.
- **The real lift is UI orchestration** — 3-step wizard, payment schedule integration into the form, customer-conditional rendering — plus a few small backend additions (most notably: customer-specific subcategory filtering and customer-level annualization view).
- **Two backend additions are needed for FirstKey-specific behavior**: subcategory-level allowlist on Portfolio, and an annualization-view field on Portfolio.

---

## Stack confirmed

- **Frontend:** React 18 + TypeScript on Vite
- **UI library:** \`@mainstay-io/ui\` (xstyled + Radix)
- **Server state:** React Query (\`@tanstack/react-query\`)
- **Routing:** React Router v6
- **Backend:** Django REST Framework, FormData mutations
- **Existing form lives at:** \`apps/hoa/hoa-web/apps/admin/src/routes/specific/charge/create.tsx\` (entry) + \`.../charge/form/createEdit.tsx\` (fields)

---

## Guardrail-by-guardrail integration map

### 1. Customer-specific category filtering

**Prototype mock:** \`PORTFOLIOS[customer].allowed_charge_categories: number[]\`

**What exists:** \`Portfolio.allowed_charge_categories\` is a real M2M field. \`GET /charge/categories/?portfolio_id=X\` already filters by it server-side. Hook: \`useChargeCategories(selectedCategoryId, portfolioId)\`.

**What works today:** Category-level filtering ✅

**Backend gap:** **Subcategory-level filtering does not exist.** FirstKey's curated list (e.g., excluding Closing Fee, Internet Fee, etc. from Fees & Fines) requires either:
- A new \`Portfolio.allowed_charge_subcategories\` M2M field, or
- A new \`ChargeCategoryConfig\` join table with \`(portfolio_id, charge_subcategory_id)\` rows

**Frontend lift:** Trivial once backend supports it — \`useChargeCategories\` already returns nested subcategories; just filter the array.

---

### 2. Payment Schedule dropdown with auto-fill

**Prototype mock:** Each HOA has a \`paymentSchedule\` array of items matching the real \`PaymentScheduleItem\` shape, including \`frequency\`, \`payment_config.payment_months\` (cents), and \`schedule_type\`.

**What exists:**
- Endpoint: \`GET /charge/payment_schedule/?property_id=X&hoa_id=Y\` — returns \`{ current: [...], future: [...] }\`
- Serializer: \`apps/hoa/api/admin_api/serializers/payment_schedule.py\`
- Model: \`PaymentSchedule\` with \`payment_config\`, \`frequency\`, \`total_annual_amount\` (computed)

**What works today:** All the data needed is there. ✅

**Frontend lift:**
- New \`<PaymentSchedulePicker>\` component (custom — no existing equivalent)
- Wire it to call the endpoint when Property + HOA + Category=Dues is selected
- On select: prefill \`charge-amount\` field with \`getDisplayAmountCents(item) / 100\`
- Manual edit clears the auto-fill state (already handled in prototype JS)

**Backend gap:** None — endpoint already exists and shape is correct.

---

### 3. Annualized vs. Quarterlized display

**Prototype mock:** \`PORTFOLIOS[customer].annualization_view: 'annualized' | 'quarterlized'\`

**What exists:** Nothing. This is a pure-display business rule today implicit in how operators think about each customer.

**Backend gap:** Add \`annualization_view\` field to \`Portfolio\` model (CharField with choices). Default to \`'quarterlized'\` (the more common case); set FirstKey to \`'annualized'\` in a data migration.

**Frontend lift:** Tiny — read the field, multiply/divide accordingly. Math is simple (×4 for quarterly→annual, ×12 for monthly→annual, ÷4 / ÷2 for quarterlize).

**Open product question:** Does the *saved* \`amount_cents\` change based on this view, or only the *displayed* amount? Prototype currently treats it as display-only (the saved amount IS what's displayed). This needs a Finance/Product call.

---

### 4. Auto-populated note templates

**Prototype mock:** \`noteTemplates[category][subcategory]\` strings with \`{notice}\` and \`{due}\` placeholders, e.g. \`'Annualized Dues: {notice} - {due}'\`.

**What exists:** Nothing. The current form has a free-text notes field.

**Backend gap:** Where templates live is the question:
- **Option A — In code:** Single \`noteTemplates.ts\` config file. Cheap. Every spec change = deploy.
- **Option B — In DB:** New \`ChargeNoteTemplate(charge_subcategory_id, template, customer_id?)\` table. Editable by Ops. Needs a small admin UI.
- **Option C — Hybrid:** Config file but loaded into a feature flag so it can be hot-swapped without deploy.

**Recommendation:** Start with Option A (in code), migrate to Option B when Ops wants self-serve editing. Templates in this prototype are encoded directly from the existing Ops template table.

**Frontend lift:** String interpolation on \`subcategory\` + \`notice_date\` + \`due_date\` change.

---

### 5. 3-step wizard

**Prototype:** \`Property & Charge\` → \`Details & Evidence\` → \`Review & Submit\`.

**What exists:** Current form is single-page (\`createEdit.tsx\`). No \`Stepper\` component in \`@mainstay-io/ui\` based on a search.

**Frontend lift:**
- New \`<Stepper>\` component (or import from a third-party — Radix doesn't have one)
- Refactor \`createEdit.tsx\` into 3 step components, share state via parent hook
- Sub-route option: \`/charge/create/property\` → \`/charge/create/details\` → \`/charge/create/review\`. Cleaner state recovery on refresh.

**Backend gap:** None.

**Open product question:** Where do step boundaries make most sense from an EPD perspective? Prototype's split is one option; not a hill to die on.

---

### 6. Insufficient-funds warning

**Prototype:** Inline + review banner when \`customer_approved_amount < charge_amount\`.

**What exists:** \`Charge.customer_approved_amount_cents\` field exists in the model and serializer, but it's not on the *create* form today — only on edit/review flows.

**Frontend lift:**
- Add \`customer_approved_amount_cents\` to the create form
- Add real-time comparison logic
- Surface inline warning + on review step

**Backend gap:** None — field already exists.

**Worth flagging:** Echo isn't sure this is a real problem today. Worth confirming with Ops before investing UI in it.

---

### 7. Void toggle + void notes

**Prototype:** Toggle on the create form that flags the charge as void at creation, with a notes field.

**What exists:** \`Charge.is_void\` (boolean) and \`Charge.void_reason\` (text) fields exist. The current creation flow sets \`is_void=false\` by default; voiding happens later via a separate edit action.

**Open product question:** Does it make sense to void *at creation*, or should this stay a post-creation action? The prototype's case is for charges Ops knows are bad on entry (e.g., duplicates) but want to log for audit. EPD should weigh in on whether this is the right model.

**Frontend lift:** Trivial if the answer is yes — just expose the existing fields on create.

---

### 8. Drag-and-drop evidence upload

**Prototype:** Drag-drop zone with file previews. Multiple files supported.

**What exists:** Current form has a basic file input. Backend accepts \`evidence\` as multipart \`FormData\`.

**Frontend lift:** New component (Radix has primitives; or use a small lib like \`react-dropzone\`). No backend change needed — \`POST /charge/\` already handles multipart upload.

**Open question:** Does the backend support multiple evidence files per charge today? The list serializer returns \`evidence: Array<...>\` so the model can hold many, but the create endpoint might only take one. Worth verifying.

---

### 9. Review-before-submit summary

**Prototype:** Editable summary view as step 3 of the wizard.

**What exists:** Nothing — current form submits straight from the single page.

**Frontend lift:** Pure UI. Render the form state in a read-mostly format with "Edit" affordances that jump back to earlier steps.

**Backend gap:** None.

---

## Backend changes summary

| # | Change | Why |
|---|---|---|
| 1 | Add \`Portfolio.allowed_charge_subcategories\` M2M (or join table) | Subcategory-level filtering for FirstKey |
| 2 | Add \`Portfolio.annualization_view\` (CharField with choices) | Drive annualized vs. quarterlized display per customer |
| 3 | (Optional) \`ChargeNoteTemplate\` table | Self-serve note template editing for Ops |
| 4 | (Verify) Multi-file evidence upload on \`POST /charge/\` | Multi-file support in the create endpoint |
| 5 | (Optional) Validation rule for customer-approved amount | Required-when-charge-exceeds-threshold |

Items 1 and 2 are the only ones strictly needed for the FirstKey-specific UX. Everything else either exists or is optional polish.

---

## Frontend changes summary

| # | Change | Reuses existing? |
|---|---|---|
| 1 | New \`<Stepper>\` component | ❌ Build new |
| 2 | Sub-route the form: \`/charge/create/{step}\` | ✅ React Router |
| 3 | New \`<PaymentSchedulePicker>\` | ❌ Build new |
| 4 | New \`<DragDropFileUpload>\` | ❌ Build new (or import lib) |
| 5 | Add \`customer_approved_amount_cents\` field | ✅ \`FieldAmountCents\` |
| 6 | Add \`is_void\` + \`void_reason\` fields on create | ✅ \`Toggle\` + \`TextField\` |
| 7 | Note template engine (string interp) | ❌ ~30 lines of code |
| 8 | Insufficient-funds banner | ✅ existing banner patterns |
| 9 | Review summary step | ❌ Build new (~simple) |
| 10 | Wire customer config from \`Portfolio\` data | ✅ existing portfolio queries |

---

## Open questions for EPD

1. **Subcategory filtering** — Are you OK adding a new M2M (or join table) for \`Portfolio → ChargeSubCategory\`? Or is there a different config pattern you'd prefer?
2. **Annualization** — Add \`annualization_view\` to \`Portfolio\`, or model it differently? Does the *stored* charge amount change based on view, or just display?
3. **Note templates** — Code config, DB table, or feature-flagged JSON? What's least painful for Ops to maintain?
4. **Wizard structure** — Is a 3-step wizard worth the refactor cost, or would a single-page form with collapsible sections accomplish most of the goal?
5. **Void at creation** — Does it make sense as a creation-time concept, or should it stay post-creation?
6. **Customer-approved amount** — Is the under-funded charge case actually a real problem today? (Tag Ops if not sure.)
7. **Evidence multi-upload** — Does \`POST /charge/\` currently accept multiple files in \`request.FILES['evidence']\`?

---

## File map (for engineers)

| Prototype concept | Real source |
|---|---|
| Categories mock | \`apps/hoa/api/admin_api/serializers/serializers.py\` (ChargeCategorySerializer) |
| Payment schedule mock | \`apps/hoa/api/admin_api/serializers/payment_schedule.py\` |
| Submit payload | \`apps/hoa/api/actions/charge.py\` |
| Existing form entry | \`apps/hoa/hoa-web/apps/admin/src/routes/specific/charge/create.tsx\` |
| Existing form fields | \`apps/hoa/hoa-web/apps/admin/src/routes/specific/charge/form/createEdit.tsx\` |
| Categories hook | \`apps/hoa/hoa-web/apps/admin/src/routes/specific/charge/hooks/useChargeCategories.ts\` |
| Mutation helper | \`useFetchMutationFormData()\` in \`apps/hoa/hoa-web/apps/admin/src/data/\` |
| Portfolio model | \`apps/hoa/api/models.py\` — Portfolio |

---

*Last updated: April 2026. Prototype source: [github.com/echowangsnowy/charge-create-prototype](https://github.com/echowangsnowy/charge-create-prototype)*
