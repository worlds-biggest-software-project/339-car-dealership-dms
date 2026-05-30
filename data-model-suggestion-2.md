# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Car Dealership DMS · Created: 2026-05-25

## Philosophy

This model collapses the 22-table normalized schema into 8 core tables by embedding related data as JSONB documents within their parent entities. Dealerships embed their staff roster, F&I product catalogue, GL chart of accounts, and integration configuration. Deals embed the full deal jacket — line items, F&I products, lender submissions, disclosures, trade-in details, and compliance screenings — in a single document. The principle: a deal jacket is the atomic unit of a car sale, and loading one should be a single-row fetch.

The relational anchors — `vehicles`, `customers`, `payments`, and `compliance_screenings` — remain as standalone tables because they have independent lifecycles: vehicles persist across multiple deals and service visits, customers accumulate history across departments, payments reconcile with Stripe and QuickBooks independently, and compliance screenings require 10-year retention with independent queryability for OFAC audits.

**Best for:** Early-stage projects prioritising development speed, single-query page loads for deal desking and F&I menus, and schema flexibility as the product iterates on workflow stages. Ideal for independent and small multi-rooftop dealerships where cross-departmental analytics are secondary to transaction speed.

**Trade-offs:**
- (+) 8 tables — minimal migration surface, fast to iterate
- (+) Single-row fetch loads a complete deal jacket with all F&I, lender, and compliance context
- (+) JSONB flexibility accommodates state-specific titling fields without schema changes
- (+) GIN indexes on JSONB support containment queries for analytics
- (-) Cross-deal analytics (F&I penetration by product type, salesperson gross profit) require JSONB extraction
- (-) Embedded staff/parts arrays grow with dealership scale — large dealer groups may hit JSONB row size limits
- (-) No foreign-key enforcement on embedded references
- (-) Accounting journal entries embedded in deals limit standalone financial reporting

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| STAR Domain Model 2026 | Deal structure in `deals.deal_jacket_json` mirrors STAR Deal API field semantics |
| ISO 3779 (VIN) | `vehicles.vin` — standalone table with CHECK constraint for cross-deal history |
| FTC Safeguards Rule | `audit_log.mfa_verified` tracks MFA compliance per action |
| OFAC SDN (31 CFR 500+) | `compliance_screenings` remains relational for 10-year retention and audit queries |
| IRS Form 8300 | Cash report entries embedded in `deals.compliance_json` with `form_8300_filed` flag |
| AAMVA D20 | Titling fields in `deals.titling_json` follow AAMVA data element naming |
| PCI DSS v4.0 | No card data stored; `payments` references Stripe tokens only |

---

## Core Tables

### dealerships

```sql
CREATE TABLE dealerships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    dba_name        TEXT,
    dealer_number   TEXT,
    oem_franchise   TEXT[],
    address_json    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"line1":"1500 Auto Mall Dr","city":"Dallas","state":"TX","postal":"75201","country":"US"}
    phone           TEXT,
    email           TEXT,
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    settings_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"sales_tax_rate_bps":825,"doc_fee_cents":49900,"bay_count":12,
    --   "labour_rate_cents":15000,"fi_product_menu":true,
    --   "operating_hours":{"mon":"07:30-18:00","sat":"09:00-17:00"}}

    staff_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","email":"john@dealer.com","full_name":"John Smith",
    --   "role":"salesperson","phone":"555-0101","commission_pct":25.00,
    --   "certifications":["ASE_A1"],"is_active":true,"hired_date":"2024-06-01","mfa_enabled":true}]

    fi_products_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","name":"Extended Warranty 5yr/100k","product_type":"extended_warranty",
    --   "provider":"Zurich","cost_cents":85000,"retail_cents":199500,"max_markup_cents":120000,
    --   "term_months":60,"mileage_limit":100000,"is_active":true}]

    parts_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","part_number":"OEM-BRK-4521","description":"Front Brake Pad Set",
    --   "category":"brakes","brand":"OEM","is_oem":true,"cost_cents":4500,"retail_cents":11900,
    --   "qty_on_hand":8,"reorder_point":4,"bin_location":"B2-14"}]

    gl_accounts_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"account_number":"4100","account_name":"New Vehicle Sales","account_type":"revenue",
    --   "department":"sales_new","qbo_account_id":"acct_xxx"}]

    integrations_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"provider":"quickbooks","status":"active","qbo_realm_id":"123456"},
    --   {"provider":"routeone","status":"active"},{"provider":"nhtsa","status":"active"}]

    stripe_account_id TEXT,
    qbo_realm_id    TEXT,
    star_dealer_id  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dealerships_slug ON dealerships(slug);
CREATE INDEX idx_dealerships_staff ON dealerships USING GIN (staff_json);
CREATE INDEX idx_dealerships_fi ON dealerships USING GIN (fi_products_json);
```

### vehicles

Vehicles remain relational — they accumulate service history across multiple deals, are traded between dealerships, and serve as the anchor for NHTSA recall checks.

```sql
CREATE TABLE vehicles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    vin             CHAR(17) NOT NULL CHECK (vin ~ '^[A-HJ-NPR-Z0-9]{17}$'),
    stock_number    TEXT,
    year            SMALLINT NOT NULL,
    make            TEXT NOT NULL,
    model           TEXT NOT NULL,
    trim_level      TEXT,
    body_style      TEXT,
    engine          TEXT,
    transmission    TEXT,
    drivetrain      TEXT,
    exterior_colour TEXT,
    interior_colour TEXT,
    odometer_miles  INTEGER,
    condition       TEXT NOT NULL CHECK (condition IN ('new','used','certified_preowned','demo')),
    status          TEXT NOT NULL DEFAULT 'in_stock'
                    CHECK (status IN ('in_transit','in_stock','on_lot','sold','wholesale','pending_recon')),
    acquisition_json JSONB,
    -- Example: {"source":"auction","date":"2026-05-10","cost_cents":1850000,"auction_name":"Manheim Dallas"}
    pricing_json    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"asking_cents":2399000,"internet_cents":2299000,"invoice_cents":2100000,
    --   "msrp_cents":2550000,"ai_market_cents":2350000,
    --   "ai_analysis":{"market_avg":2380000,"competitor_low":2199000,"competitor_high":2599000,
    --     "days_to_turn":25,"confidence":0.85,"updated":"2026-05-25"}}
    nhtsa_decoded_json JSONB,
    recall_checks_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"checked_at":"2026-05-20","campaigns":[],"status":"clear"}]
    photo_urls      TEXT[] DEFAULT '{}',
    features        TEXT[] DEFAULT '{}',
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dealership_id, vin)
);

CREATE INDEX idx_vehicles_status ON vehicles(dealership_id, status, condition);
CREATE INDEX idx_vehicles_make_model ON vehicles(dealership_id, make, model, year);
CREATE INDEX idx_vehicles_pricing ON vehicles USING GIN (pricing_json);
```

### customers

Customers remain relational — they appear across sales leads, deals, and service visits, and carry compliance state (OFAC clearance) that persists independently.

```sql
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    address_json    JSONB,
    date_of_birth   DATE,
    drivers_licence_json JSONB,
    -- Example: {"number":"DL12345","state":"TX","expiry":"2028-11-15"}
    is_business     BOOLEAN NOT NULL DEFAULT false,
    business_name   TEXT,
    tax_id          TEXT,
    preferred_contact TEXT NOT NULL DEFAULT 'phone' CHECK (preferred_contact IN ('phone','sms','email')),

    lead_json       JSONB,
    -- Example: {"source":"website","type":"sales","status":"qualified","assigned_to":"uuid",
    --   "ai_score":78,"ai_next_action":"follow_up_call","vehicle_interest_id":"uuid",
    --   "first_response_at":"2026-05-20T10:15:00Z","activities":[
    --     {"type":"email","at":"2026-05-20T10:15:00Z","by":"uuid","notes":"Sent brochure"}]}

    payment_methods_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"stripe_pm_id":"pm_xxx","type":"card","last_four":"4242","brand":"visa","is_default":true}]

    communication_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","direction":"outbound","channel":"sms","type":"reminder",
    --   "body":"Your service appointment is tomorrow at 9am","at":"2026-05-24T16:00:00Z"}]

    ofac_cleared    BOOLEAN,
    ofac_cleared_at TIMESTAMPTZ,
    ai_churn_risk   NUMERIC(4,3),
    qbo_customer_id TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_dealership ON customers(dealership_id);
CREATE INDEX idx_customers_phone ON customers(dealership_id, phone);
CREATE INDEX idx_customers_email ON customers(dealership_id, email);
CREATE INDEX idx_customers_lead ON customers USING GIN (lead_json);
```

### deals

The central document — embeds the full deal jacket: line items, F&I products, lender submissions, trade-in details, compliance records, titling, disclosures, and accounting entries.

```sql
CREATE TABLE deals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    deal_number     SERIAL,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    vehicle_id      UUID NOT NULL REFERENCES vehicles(id),
    salesperson_id  UUID,
    fi_manager_id   UUID,
    sales_manager_id UUID,
    deal_type       TEXT NOT NULL CHECK (deal_type IN ('retail','lease','wholesale','fleet')),
    deal_status     TEXT NOT NULL DEFAULT 'pending'
                    CHECK (deal_status IN ('pending','desking','fi_review','contract_signed',
                                           'funded','delivered','unwound','cancelled')),

    pricing_json    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"selling_cents":2850000,"trade_allowance_cents":1200000,"trade_payoff_cents":800000,
    --   "doc_fee_cents":49900,"total_taxes_cents":198000,"total_fees_cents":25000,
    --   "total_fi_cents":285000,"total_deal_cents":3207900,"down_payment_cents":500000,
    --   "amount_financed_cents":2707900,"front_gross_cents":180000,"back_gross_cents":200000,
    --   "total_gross_cents":380000}

    trade_in_json   JSONB,
    -- Example: {"vehicle_id":"uuid","vin":"1HGCM82633A004352","year":2020,"make":"Honda",
    --   "model":"Accord","odometer":45000,"allowance_cents":1200000,"payoff_cents":800000,
    --   "acv_cents":1100000,"condition_notes":"Minor scratches on rear bumper",
    --   "ai_appraisal":{"market_value_cents":1150000,"confidence":0.79}}

    fi_products_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"fi_product_id":"uuid","name":"Extended Warranty 5yr/100k",
    --   "product_type":"extended_warranty","selling_price_cents":199500,"cost_cents":85000,
    --   "term_months":60,"deductible_cents":10000,"ai_recommended":true,"ai_confidence":0.82}]

    lender_submissions_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","lender_name":"Chase Auto","lender_code":"CHASE",
    --   "submission_type":"credit_app","status":"approved","apr_bps":549,"term_months":72,
    --   "amount_financed_cents":2707900,"monthly_payment_cents":44200,
    --   "conditions":["Proof of income","Proof of residence"],
    --   "routeone_ref":"RO-2026-12345","submitted_at":"2026-05-20T14:00:00Z",
    --   "decision_at":"2026-05-20T14:05:00Z"}]

    compliance_json JSONB NOT NULL DEFAULT '{}',
    -- Example: {"ofac_screening":{"result":"clear","screened_at":"2026-05-20T13:00:00Z",
    --   "sdn_list_date":"2026-05-19"},
    --   "cash_report":{"cash_amount_cents":1500000,"form_8300_filed":true,
    --     "filed_at":"2026-05-22T09:00:00Z","irs_confirmation":"8300-2026-001"},
    --   "buyers_guide":"limited_warranty",
    --   "ftc_disclosures":{"equal_credit":true,"privacy_notice":true,"risk_based_pricing":true}}

    titling_json    JSONB,
    -- Example: {"state":"TX","title_status":"submitted","submission_method":"electronic",
    --   "fees_cents":7500,"taxes_cents":198000,"submitted_at":"2026-05-21T10:00:00Z",
    --   "aamva_ref":"TX-2026-98765"}

    messages_json   JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","direction":"outbound","channel":"email","type":"contract",
    --   "subject":"Your purchase agreement","body":"...","at":"2026-05-20T16:00:00Z"}]

    journal_entries_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"entry_date":"2026-05-20","memo":"Vehicle sale #1042","lines":[
    --   {"account":"4100","debit_cents":0,"credit_cents":2850000,"department":"sales_used"},
    --   {"account":"1200","debit_cents":2850000,"credit_cents":0,"department":"sales_used"}]}]

    rdr_submitted_at TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    funded_at       TIMESTAMPTZ,
    star_deal_id    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deals_dealership ON deals(dealership_id, deal_status);
CREATE INDEX idx_deals_customer ON deals(customer_id);
CREATE INDEX idx_deals_vehicle ON deals(vehicle_id);
CREATE INDEX idx_deals_fi ON deals USING GIN (fi_products_json);
CREATE INDEX idx_deals_lender ON deals USING GIN (lender_submissions_json);
```

### repair_orders

Service repair orders embed line items, technician time, and parts used.

```sql
CREATE TABLE repair_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    ro_number       SERIAL,
    vehicle_id      UUID NOT NULL REFERENCES vehicles(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    service_advisor_id UUID,
    status          TEXT NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','in_progress','waiting_parts','waiting_approval',
                                      'ready','invoiced','paid','closed','void')),
    odometer_in     INTEGER,
    odometer_out    INTEGER,
    customer_concern TEXT,
    promised_date   TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    is_warranty     BOOLEAN NOT NULL DEFAULT false,
    is_internal     BOOLEAN NOT NULL DEFAULT false,

    line_items_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","sort_order":1,"item_type":"labour","op_code":"DIAG-001",
    --   "description":"Diagnose engine misfire","technician_id":"uuid","hours_estimated":1.0,
    --   "hours_actual":0.8,"unit_price_cents":15000,"cost_cents":4000,"total_cents":15000,
    --   "is_approved":true,"is_warranty":false,"taxable":true}]

    subtotal_cents  BIGINT NOT NULL DEFAULT 0,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL DEFAULT 0,
    qbo_invoice_id  TEXT,
    star_ro_id      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ro_dealership ON repair_orders(dealership_id, status);
CREATE INDEX idx_ro_vehicle ON repair_orders(vehicle_id);
CREATE INDEX idx_ro_line_items ON repair_orders USING GIN (line_items_json);
```

### compliance_screenings

OFAC and regulatory screenings remain relational for 10-year retention and independent audit queries.

```sql
CREATE TABLE compliance_screenings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    deal_id         UUID REFERENCES deals(id),
    screening_type  TEXT NOT NULL CHECK (screening_type IN ('ofac_sdn','pep','adverse_media')),
    screened_name   TEXT NOT NULL,
    result          TEXT NOT NULL CHECK (result IN ('clear','potential_match','confirmed_match','error')),
    match_details   JSONB,
    sdn_list_date   DATE,
    screened_by     UUID,
    screened_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_customer ON compliance_screenings(customer_id);
CREATE INDEX idx_compliance_deal ON compliance_screenings(deal_id);
CREATE INDEX idx_compliance_type ON compliance_screenings(dealership_id, screening_type, screened_at DESC);
```

### payments

```sql
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    deal_id         UUID REFERENCES deals(id),
    repair_order_id UUID REFERENCES repair_orders(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    amount_cents    BIGINT NOT NULL,
    method          TEXT NOT NULL CHECK (method IN ('cash','cheque','card','ach','wire',
                                                    'financing','trade_in','terminal')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','succeeded','failed','refunded','partial_refund')),
    stripe_payment_intent_id TEXT,
    refund_cents    BIGINT NOT NULL DEFAULT 0,
    qbo_payment_id  TEXT,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_deal ON payments(deal_id);
CREATE INDEX idx_payments_ro ON payments(repair_order_id);
CREATE INDEX idx_payments_dealership ON payments(dealership_id, status);
```

### audit_log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('staff','customer','system','integration','ai')),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID,
    changes_json    JSONB,
    ip_address      INET,
    mfa_verified    BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_dealership ON audit_log(dealership_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Example Queries

### Load a complete deal jacket

```sql
SELECT d.*, v.vin, v.year, v.make, v.model, v.condition,
       c.first_name, c.last_name, c.email, c.phone,
       COALESCE(p.total_paid, 0) AS total_paid_cents
FROM deals d
JOIN vehicles v ON v.id = d.vehicle_id
JOIN customers c ON c.id = d.customer_id
LEFT JOIN LATERAL (
    SELECT SUM(amount_cents) AS total_paid
    FROM payments WHERE deal_id = d.id AND status = 'succeeded'
) p ON true
WHERE d.id = $1;
```

### F&I penetration by product type (JSONB extraction)

```sql
SELECT elem->>'product_type' AS product_type,
       COUNT(*) AS deals_with_product,
       SUM((elem->>'selling_price_cents')::bigint) AS total_revenue_cents,
       SUM((elem->>'selling_price_cents')::bigint - (elem->>'cost_cents')::bigint) AS total_margin_cents,
       ROUND(100.0 * COUNT(*) / NULLIF(total_deals.cnt, 0), 1) AS penetration_pct
FROM deals d,
     jsonb_array_elements(d.fi_products_json) AS elem,
     (SELECT COUNT(*) AS cnt FROM deals WHERE dealership_id = $1 AND deal_status IN ('funded','delivered')) AS total_deals
WHERE d.dealership_id = $1 AND d.deal_status IN ('funded','delivered')
GROUP BY elem->>'product_type', total_deals.cnt
ORDER BY total_revenue_cents DESC;
```

### Salesperson gross profit ranking

```sql
SELECT d.salesperson_id,
       s.elem->>'full_name' AS salesperson_name,
       COUNT(*) AS deals_closed,
       SUM((d.pricing_json->>'total_gross_cents')::bigint) AS total_gross_cents,
       AVG((d.pricing_json->>'total_gross_cents')::bigint) AS avg_gross_per_deal
FROM deals d,
     dealerships dl,
     jsonb_array_elements(dl.staff_json) AS s(elem)
WHERE d.dealership_id = $1
  AND dl.id = $1
  AND d.deal_status IN ('funded','delivered')
  AND (s.elem->>'id') = d.salesperson_id::text
  AND d.created_at >= date_trunc('month', now())
GROUP BY d.salesperson_id, s.elem->>'full_name'
ORDER BY total_gross_cents DESC;
```

### Unfiled IRS 8300 reports

```sql
SELECT d.deal_number, d.customer_id, c.first_name, c.last_name,
       (d.compliance_json->'cash_report'->>'cash_amount_cents')::bigint AS cash_cents,
       d.compliance_json->'cash_report'->>'filing_deadline' AS deadline
FROM deals d
JOIN customers c ON c.id = d.customer_id
WHERE d.dealership_id = $1
  AND d.compliance_json->'cash_report' IS NOT NULL
  AND (d.compliance_json->'cash_report'->>'form_8300_filed')::boolean = false;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Dealership (with staff, F&I products, parts, GL, integrations) | 1 | dealerships |
| Vehicles (with pricing, recalls) | 1 | vehicles |
| Customers (with leads, communication, payment methods) | 1 | customers |
| Deals (with F&I, lenders, compliance, titling, journal entries) | 1 | deals |
| Repair Orders (with line items) | 1 | repair_orders |
| Compliance Screenings | 1 | compliance_screenings |
| Payments | 1 | payments |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Deal jacket as a single JSONB-rich document** — a deal in the automotive retail world is a self-contained transaction packet (the "deal jacket"). Embedding F&I products, lender submissions, compliance records, trade-in details, disclosures, and accounting entries inside the deal mirrors how dealership staff actually work: they open a deal and see everything.

2. **Compliance screenings remain relational** — OFAC SDN checks require 10-year independent retention and must be queryable by regulatory auditors without parsing deal documents. This is the compliance anchor of the model.

3. **Vehicles remain relational** — a vehicle's lifecycle spans multiple owners, deals, and service visits. It may be traded in, reconditioned, and resold. Its NHTSA recall history accumulates independently. The vehicle table is the cross-departmental join point.

4. **Customers remain relational** — a customer interacts with sales, service, and parts departments. Lead tracking, communication history, and payment methods embed inside the customer document, but the customer record itself needs to be joinable across deals and repair orders.

5. **Leads embedded in customers** — rather than a separate leads table, lead tracking is a JSONB field on the customer. This reflects the reality that a lead is always a person, and the lead-to-customer conversion is a state change on the same record rather than a data migration.

6. **F&I product catalogue embedded in dealership** — the product menu is dealership-scoped and changes infrequently. When a product is sold on a deal, a snapshot is embedded in `deals.fi_products_json` — the sold instance is independent of future catalogue changes.

7. **Journal entries embedded in deals** — accounting entries generated from a deal (vehicle sale debit/credit, trade-in entries) are embedded in the deal document. This simplifies the deal-to-accounting trail but requires JSONB extraction for period-level financial reporting.

8. **Payments remain relational** — Stripe webhooks and QuickBooks sync operate on individual payment records. Payments also need to be queryable across deals and repair orders for cash flow reporting.

9. **Recall checks on vehicle** — NHTSA recall lookups are appended to `vehicles.recall_checks_json` as they occur, building a longitudinal safety record per vehicle.

10. **GIN indexes on deal JSONB** — `fi_products_json` and `lender_submissions_json` get GIN indexes to support containment queries (find all deals with GAP insurance, find all deals submitted to Chase Auto) without full-table scans.
