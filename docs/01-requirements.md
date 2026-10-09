# MedPro: Requirements Summary, Assumptions and Open Legal Questions

**Deliverable 1 of 9** · **Signed off by B on 2026-10-09 with all defaults** · 2026-10-09 (BS 2083-06-23, *verify conversion in app*)

Conventions in this document:

- **[VERIFY-n]** marks an assumption or legal value I have not confirmed from a primary source. Every one is listed in section 6 and every legal value will live in configuration, not code.
- **[Q-n]** marks an open question for you (section 7).
- Rules quoted from your brief (NPC Act 2057, NPC Regulation 2059, Code of Conduct, Hospital Pharmacy Service Guideline 2072) are taken as you stated them. I have not re-read those texts, so the numbers you gave are also config defaults that you should verify.

---

## 1. Decisions confirmed with you

| Topic | Decision |
|---|---|
| First users | **Distributors / wholesalers first**, then **retail pharmacies**. Hospitals (government and private) come after. The data model covers all tiers from day one so nothing has to be re-designed later. |
| Pilot scale | About 20 sites, 300 users, 5,000 SKUs (roughly 50,000 active batches). Multi-tenant, so one deployment serves many organisations. |
| Backend | **Java 21 + Spring Boot 3**, PostgreSQL 16, Redis 7, Flyway migrations, OpenAPI via springdoc. |
| Frontend | React + TypeScript + Tailwind, installable PWA with offline queue (IndexedDB). |
| Code location | `github.com/DRISHYAM-OVEXA/medSupply`. Until my access works, deliverables are staged in project files under `medpro/`. |
| SMS / payments | Pluggable SMS adapter with Sparrow SMS as the first provider. Phase 1 takes cash and credit only; eSewa, Khalti and Fonepay come later through a payment adapter. |

### What "distributors first" changes

Your phase plan was written with hospitals in mind. With distributors first, I suggest this split. Nothing is dropped; some items move.

- **Moves into Phase 1:** wholesale sales orders and invoices, customer (retailer) credit limits and ageing, trading-party licence checks (block sale to a retailer whose pharmacy registration has lapsed), scheme and bonus stock from manufacturers, purchase and expiry returns to suppliers.
- **Stays in Phase 1 as you wrote it:** auth/RBAC with the NPC registry, product and batch master, inventory with FEFO and expiry alerts, procurement, dispensing and billing with price rules, audit trail, dashboard, Nepali/English and BS dates. Retail counter dispensing is built in Phase 1 but piloted after the distributor rollout.
- **Moves to Phase 2 (hospital-only rules):** formulary and committees, free and subsidised medicines, hospital pharmacy ledger rules (same-day deposit, dual signatory, committee-backed spending), Public Procurement Act workflow, staffing checker by bed count.

**[Q-1]** Is this re-ordering acceptable?

---

## 2. Stakeholders and roles

One personal account per person. No shared logins. Roles are data (configurable), seeded with defaults below. A person can hold several roles at different sites.

| Role | Tier | Default scope (configurable) |
|---|---|---|
| Admin (platform) | All | Tenants, global settings, NPC adapter config. No stock actions. |
| Org Admin | All | Users, roles, sites, settings for one organisation. |
| Pharmacist | Distributor, Retail, Hospital | Dispense, approve stock adjustments, release quarantined batches, sell scheduled items. Requires **active** NPC Pharmacist registration. |
| Pharmacy Assistant | Retail, Hospital | Dispense non-restricted items, receive goods, count stock. Requires **active** NPC Pharmacy Assistant registration. Restricted items need pharmacist co-sign. **[VERIFY-1]** |
| Clinical Pharmacist | Hospital | Pharmacist scope plus formulary review, ADR follow-up, antimicrobial review. |
| Pharmacy Head / Hospital Head | Hospital | Approvals, reports, committee chair functions. |
| Helper / Store keeper | All | Pick, pack, move stock, cycle count. No sale, no approval. |
| Procurement Officer | All | Suppliers, purchase requests, POs. |
| Accounts Officer | All | Ledger, receipts, payments, deposits, year-end. |
| Sales Representative | Distributor | Book orders for retailers (mobile, offline). *Added by me for the distributor tier.* |
| Committee Member | Hospital | Committee meetings, votes, decisions. |
| Distributor / Wholesaler user | External | Views own orders and invoices with a buyer; confirms dispatch. |
| Manufacturer / Importer user | External | Publishes products, batches, schemes, recalls; sees sell-through to distributors. |
| Logistics partner | External | Shipment status, proof of delivery, temperature uploads. |
| Regulator / Auditor | External | Read-only, time-boxed access to reports and audit log. |
| Patient (optional portal) | External | Phase 3. Own prescriptions, free-medicine availability. |

Scope-of-practice rule: every action that legally needs a registered professional is tagged with the **professional category required** (`PHARMACIST`, `PHARMACY_ASSISTANT`, `ANY_REGISTERED`, `NONE`). The permission check passes only if the user has the role **and** an active registration of that category. The mapping of actions to categories is a config table, seeded with my defaults, all marked **[VERIFY-1]**.

---

## 3. Functional requirements

### 3.1 Professional registry and access control (Phase 1)

- Two registers: **Pharmacist** and **Pharmacy Assistant**. Fields: NPC registration number, category, registration date, employer institution, last update date, status (`ACTIVE`, `SUSPENDED`, `REMOVED`, `LAPSED`), nationality, registration expiry (required for non-Nepalese), verification source and date, certificate scan.
- Any status other than `ACTIVE` immediately blocks dispensing, approval and stock actions. Suspension or removal sets the user's linked account to disabled, revokes their sessions, and notifies the employing institution's Org Admin.
- Non-Nepalese professionals: alerts at **60 and 35 days** before expiry (config); auto-lapse at expiry or on employment end.
- Reminders to update details, and to notify change of address or employer **within one month** (config: 30 days). **[VERIFY-2]** whether "one month" is 30 days or one BS calendar month.
- Verification via an adapter interface `NpcRegistryPort` with two implementations now: `ManualVerification` (staff record what they checked, with evidence upload) and `CertificateScan` (reads QR/barcode on certificate or digital ID and pre-fills the form for manual confirmation). An `NpcOfficialFeed` implementation is a stub for Phase 3. **[VERIFY-3]** whether NPC publishes any machine-readable registry or QR format.
- Duty roster: the registered pharmacist on duty per site per shift. If none is on duty, the site shows a banner and actions tagged `PHARMACIST` are blocked. **[Q-2]** Should "no pharmacist on duty" block those sales or only warn and log?
- Audit: every dispensing, approval, price and stock action is written to an append-only, hash-chained audit log with user, professional registration number at that moment, site, device, timestamp and before/after values.

### 3.2 Product and batch master (Phase 1)

- Product: brand name, generic (INN) and composition, strength, dosage form, pack and unit conversions (box → strip → tablet), manufacturer, marketing authorisation holder, **DDA product registration number** and its validity **[VERIFY-4]**, therapeutic class, schedule/category (placeholder list, see 3.10), storage condition (temperature range, humidity, light), cold-chain flag, essential-medicine flag, HS code, tax treatment.
- Batch: batch number, manufacturing date, expiry date, MRP, purchase rate, status (`QUARANTINE`, `APPROVED`, `REJECTED`, `RECALLED`, `EXPIRED`), COA upload, GS1 GTIN and serial (optional).
- Substitute lookup by same generic and strength.
- Date handling: stored in AD (UTC timestamps, `date` columns). Shown as BS and AD side by side. Expiry printed as "month/year" on packs is stored as the last day of that month. **[VERIFY-5]** that this is the accepted convention in Nepal.

### 3.3 Inventory (Phase 1)

- Multi-site, multi-location (store, rack, bin, **quarantine**, in-transit). Quarantine is a separate location type; quarantined batches cannot be picked for sale.
- **Stock derived only from an append-only movement ledger.** Corrections are reversing entries with a reason, never edits. A cached balance table is rebuilt from the ledger and checked nightly.
- FEFO picking by default; override needs a reason and is audited.
- Reorder level, safety stock, min/max per product per site.
- Scheduled checks: daily expiry scan; near-expiry alerts at 180/90/30 days (config) **[Q-3]**; auto-move to `EXPIRED` and quarantine on expiry date.
- Stock adjustment (damage, theft, sample, count variance) with reason codes and pharmacist approval above a configurable value.
- Cycle count and full physical verification with variance report.
- Transfers between sites with in-transit status.
- Barcode and GS1 DataMatrix / QR scanning in receiving, picking and billing (phone camera in PWA, plus USB scanners).

### 3.4 Procurement (Phase 1)

- Suppliers with licence documents and licence expiry; purchase blocked from a supplier whose licence has expired.
- Purchase request → approval → purchase order → goods receipt (batch, expiry, free qty, scheme) → supplier invoice → three-way match (PO, GRN, invoice) with tolerance in config.
- Purchase return and expiry return to supplier, with debit note and claim tracking.
- Government hospitals (Phase 2): Public Procurement Act 2063 / Regulation 2064 workflow with method chosen by value thresholds held in config. **[VERIFY-6]** thresholds and required approvals.

### 3.5 Sales, dispensing and billing (Phase 1)

- **Distributor:** sales order (from rep or office) → credit-limit check → licence check of buyer → FEFO pick list → invoice → delivery challan. Schemes and bonus quantities. Sales returns with batch validation and credit note. Customer ageing (0-30, 31-60, 61-90, 90+).
- **Retail:** counter billing by brand, generic or barcode; loose-unit sale; prescription capture for prescription-only items; counselling notes; returns.
- **Price rule:** selling price = purchase price × (1 + margin), **capped at MRP**. Margin is configurable per tenant, tier, product category or product; default 20% as you gave for hospitals. **[VERIFY-7]** and **[Q-4]** what margin applies to distributors and to retail pharmacies.
- The system never sells above MRP. If the computed price exceeds MRP, MRP is used and the event is logged.
- Tax: medicines treated as VAT-exempt by default with a per-product override. **[VERIFY-8]**
- Invoices numbered per fiscal year per site, no gaps. **[VERIFY-9]** IRD e-billing (CBMS) requirements and real-time reporting thresholds.

### 3.6 Audit trail (Phase 1)

- Append-only table, writes only via a database role that has `INSERT` but no `UPDATE`/`DELETE`.
- Each row stores the hash of the previous row, so tampering is detectable. A verification job reports any break.
- Viewer for Admin and Auditor with filters (user, action, entity, date range in BS or AD) and CSV export.

### 3.7 Dashboard and reports (Phase 1 basic)

Stock value by site, items below reorder level, near-expiry and expired value, top-selling items, receivables ageing, purchase vs sales, pending approvals, professional registrations expiring. CSV and PDF export.

### 3.8 Localisation (Phase 1)

- Full UI in **Nepali (Devanagari) and English**, switchable per user. All strings in resource files; no hard-coded text.
- **Bikram Sambat** shown beside AD on every date field and printout. BS conversion uses a lookup table of month lengths (BS months vary year to year), shipped as data and updatable. **[VERIFY-10]** source table for future years.
- Fiscal year runs Shrawan 1 to Ashadh end. **[VERIFY-11]**
- Currency NPR, formatted with Nepali digit grouping (1,00,000.00), Devanagari numerals optional.

### 3.9 Notifications (Phase 1 in-app + email, SMS adapter)

In-app, email and SMS (Sparrow). Events: near-expiry, reorder, registration expiry, suspension, cold-chain breach (Phase 2), approval requests, overdue receivables.

### 3.10 Modules not covered by your source documents (placeholders)

Per your rule 4.7, these are designed as **clearly marked placeholder modules**: data structures and hooks exist, but every rule value is empty config until you confirm it. Nothing here is enforced as law until you verify it.

| Area | What I expect the law covers (unconfirmed) | Placeholder design |
|---|---|---|
| Drug registration (Drugs Act 2035, DDA) | Products must be registered with the Department of Drug Administration before sale; registration has validity and renewal. **[VERIFY-4]** | Product has `registration_number`, `valid_until`, `status`. Config switch: block sale of unregistered or expired-registration products. |
| Import licensing | Importers need DDA import recommendation and product-wise permission. **[VERIFY-12]** | Document types `IMPORT_LICENCE`, `IMPORT_RECOMMENDATION` with expiry; GRN from import supplier warns if missing. |
| Wholesale and retail pharmacy registration | Pharmacies (wholesale and retail) register with DDA and renew periodically; a registered pharmacist or assistant must be in charge. **[VERIFY-13]** | Organisation and site licences with expiry; distributor cannot invoice a retailer whose licence is expired (config switch). |
| Recalls | DDA can order recall of a batch; holders must stop sale and return. **[VERIFY-14]** | Recall entity with class, scope (batches), deadline, notifications; affected stock auto-quarantined; trace list of every customer who received the batch. Built fully in Phase 3, data captured from Phase 1. |
| Narcotics and controlled substances (Narcotic Drugs (Control) Act 2033, Drugs Act 2035) | Separate register, prescription retention, quantity limits, periodic reporting. **[VERIFY-15]** | Product `control_class`; controlled-drug register (running balance per batch); pharmacist-only sale; prescription mandatory; report template left empty. |
| Drug Sale and Distribution Code 2071 | Storage, display, record keeping and staffing rules for sellers. **[VERIFY-16]** | Compliance checklist module with configurable items per site. |

---

## 4. Non-functional requirements

| Area | Requirement |
|---|---|
| Security | OAuth2 / OIDC with short-lived JWT access tokens and rotating refresh tokens; optional TOTP second factor for pharmacists and admins; TLS 1.2+; PostgreSQL encryption at rest (disk) plus column encryption for patient identifiers; secrets outside the repo. |
| Privacy | Patients stored with minimum data; patient name and phone encrypted; reports aggregate or pseudonymise. **No real patient data in seed or test data.** **[VERIFY-17]** applicable Nepali privacy law (Individual Privacy Act 2075). |
| RBAC | Role + professional-category + site scope checks on every endpoint, enforced server-side. |
| Audit | Hash-chained append-only log (3.6). |
| Offline | PWA works offline for scanning, counting, receiving, order booking and counter sales of non-restricted items. Writes queue locally with client-generated IDs and sync with retry. Stock conflicts resolved server-side against the ledger; any sale that would drive stock negative is flagged for review rather than silently rejected. **[Q-5]** whether offline sales are allowed at all for prescription-only items. |
| Multi-tenancy | Shared schema with `tenant_id` on every row and PostgreSQL row-level security. |
| Performance (pilot) | p95 API < 300 ms for reads, < 800 ms for invoice posting, at 300 users. |
| Availability | 99.5% monthly for the pilot. Nightly full backup plus WAL archiving (point-in-time recovery), restore tested monthly. |
| Observability | Structured JSON logs, Prometheus metrics, OpenTelemetry traces, error tracking (Sentry-compatible). |
| Accessibility | WCAG 2.1 AA; large touch targets for warehouse screens. |
| Retention | Records kept per legal retention periods held in config. **[VERIFY-18]** |

---

## 5. Phase 1 acceptance criteria (per module)

These are the tests I will write before calling each module done.

**Auth / RBAC / NPC registry**
1. A user whose registration is `LAPSED`, `SUSPENDED` or `REMOVED` gets HTTP 403 on dispense, approve and stock-move endpoints, with error code `REGISTRATION_INACTIVE`.
2. Suspending a registration disables the linked user and revokes active tokens within one request cycle.
3. A non-Nepalese registration expiring in 60 and 35 days generates one alert each; on the expiry date it becomes `LAPSED` automatically.
4. A Pharmacy Assistant cannot complete an action tagged `PHARMACIST` without a pharmacist co-sign.
5. Two people cannot share one account: concurrent sessions from different devices are visible to the Org Admin, and each action records the actor's registration number.

**Product and batch**
6. A batch cannot be created with expiry before manufacturing date, or without a batch number.
7. A new batch starts in `QUARANTINE` unless the supplier and product are configured for auto-release.

**Inventory**
8. Stock on hand always equals the sum of ledger movements; there is no API to update a balance directly.
9. A ledger row cannot be updated or deleted (database rejects it).
10. Picking returns the earliest-expiry approved batch first; an override needs a reason.
11. Expired and quarantined batches are never offered for sale.
12. Near-expiry alerts fire at each configured threshold once per batch.

**Procurement**
13. A GRN against a supplier with an expired licence is blocked (when the config switch is on).
14. Invoice matching flags quantity or price differences beyond tolerance.

**Sales and billing**
15. Selling price never exceeds MRP; a price computed above MRP is set to MRP and logged.
16. Default margin of 20% produces the expected price; changing the config changes the price without a deploy.
17. A distributor invoice to a customer over their credit limit is blocked unless approved by a role with that permission.
18. Invoice numbers are sequential per site per fiscal year with no gaps.

**Audit**
19. Every dispense, approval, price change and stock movement creates exactly one audit row; the hash-chain check passes, and fails if a row is altered.

**Localisation**
20. Every screen renders in Nepali and English with no missing keys; known AD/BS date pairs convert correctly in both directions.

---

## 6. Assumptions to verify

| ID | Assumption | Where it is used |
|---|---|---|
| VERIFY-1 | Which actions need a pharmacist vs an assistant (scope of practice mapping). | RBAC |
| VERIFY-2 | "Within one month" = 30 days. | Registry reminders |
| VERIFY-3 | NPC has no public machine-readable registry or certificate QR format yet. | NPC adapter |
| VERIFY-4 | DDA product registration number and validity must be held and checked. | Product master |
| VERIFY-5 | Pack expiry "MM/YYYY" means end of that month. | Batch, FEFO |
| VERIFY-6 | PPA 2063 / PPR 2064 thresholds and approvals. | Hospital procurement (Phase 2) |
| VERIFY-7 | Margin values per tier (20% given for hospitals). | Price rules |
| VERIFY-8 | Medicines are VAT-exempt in Nepal. | Billing |
| VERIFY-9 | IRD e-billing / CBMS obligations for distributors and pharmacies. | Invoicing |
| VERIFY-10 | Source for BS calendar month lengths for future years. | Dates |
| VERIFY-11 | Fiscal year Shrawan 1 to Ashadh end. | Invoicing, year-end |
| VERIFY-12 | Import licence and recommendation requirements. | Placeholder |
| VERIFY-13 | Wholesale/retail pharmacy registration and in-charge rules. | Placeholder |
| VERIFY-14 | Recall procedure and deadlines. | Placeholder |
| VERIFY-15 | Narcotic and psychotropic register and reporting rules. | Placeholder |
| VERIFY-16 | Drug Sale and Distribution Code 2071 obligations. | Placeholder |
| VERIFY-17 | Privacy obligations (Individual Privacy Act 2075). | Patient data |
| VERIFY-18 | Record retention periods. | Archiving |
| VERIFY-19 | All numbers in your brief (60/35-day alerts, 20% margin, deposit timing, staffing table, 10% free-care quota) as stated in the source documents. | Config defaults |

---

## 7. Open questions for you

- **Q-1** Accept the re-ordering in section 1 (distributor features into Phase 1, hospital-only rules into Phase 2)?
- **Q-2** When no pharmacist is on duty: **block pharmacist-only actions** (my default) or warn and log only?
- **Q-3** Near-expiry thresholds: **180 / 90 / 30 days** (my default)?
- **Q-4** What margin should distributors and retail pharmacies use, or is margin there set by the manufacturer's trade price and MRP rather than a percentage?
- **Q-5** Offline counter sales: **allow for OTC only, never for prescription-only or controlled items** (my default)?

---

## 8. Resolution of open questions (2026-10-09)

B confirmed the document as written, so the defaults apply: Q-1 re-ordering accepted; Q-2 block pharmacist-only actions; Q-3 180/90/30 days; Q-5 offline sales for OTC only. Q-4 had no default: margin is a configurable rule per tier with 20% as the seeded value everywhere, marked VERIFY-7 until B supplies the distributor and retail figures.

## 9. Next deliverable

After your sign-off: **Deliverable 2, architecture diagram and module breakdown** (Spring Boot modular monolith, PWA client, sync design, deployment topology).
