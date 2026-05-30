# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Car Dealership DMS · Created: 2026-05-25

## Philosophy

This model assigns a dedicated table to every first-class concept in the automotive retail domain — dealerships, vehicles, deals, F&I products, lender submissions, service repair orders, parts, accounting entries, and compliance records each live in their own table with strict foreign-key relationships. The design reflects the departmental structure of a dealership (sales, F&I, service, parts, accounting) while maintaining a unified customer and vehicle graph across departments.

The schema aligns with the STAR Automotive Retail Domain Model (2026) wherever possible, using STAR-aligned entity names and field semantics for deals, repair orders, service scheduling, and retail delivery reporting. VINs follow ISO 3779, compliance surfaces (OFAC SDN screening, IRS 8300, FTC Safeguards Rule) are first-class entities with their own queryable tables, and financial fields use BIGINT cents throughout.

**Best for:** Franchise dealerships and dealer groups that need full departmental separation, rich cross-dimensional reporting (gross profit by salesperson by vehicle segment, F&I penetration by product type, service absorption ratio), regulatory auditability, and deep integration with OEM data feeds, lender networks, and STAR-standard APIs.

**Trade-offs:**
- (+) Full referential integrity across sales, F&I, service, parts, and accounting
- (+) Every compliance surface (OFAC, IRS 8300, titling) has its own queryable table
- (+) STAR Domain Model alignment enables native interoperability with OEM and lender systems
- (+) Standard SQL joins for any KPI (ARO, gross margin, days-to-turn, F&I penetration)
- (-) 24 tables is significant migration surface
- (-) Cross-department queries (deal-to-service history) require multi-table joins
- (-) Departmental boundaries in schema may not reflect smaller dealerships' actual workflow

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| STAR Domain Model 2026 | Entity names and field semantics for deals, repair orders, parts, accounting align with STAR JSON schemas |
| STAR Sales Process API v6.1.4 | `deals` table structure mirrors STAR Deal API fields (deal_type, deal_status, vehicle, customer, pricing) |
| STAR Service Scheduling API | `appointments` table fields align with STAR scheduling resource schema |
| STAR Retail Delivery Report | `deals.rdr_submitted_at` tracks OEM delivery reporting per STAR RDR API |
| ISO 3779 (VIN) | `vehicles.vin` — 17-character CHECK constraint for inventory and service |
| FTC Safeguards Rule (16 CFR 314) | `audit_log` with MFA tracking, `integrations` with vendor security status |
| OFAC SDN (31 CFR 500+) | `compliance_screenings` table for SDN list checks with 10-year retention |
| IRS Form 8300 | `cash_reports` table for transactions exceeding USD 10,000 |
| GLBA Financial Privacy Rule | Customer data access controls enforced via `staff.role` and audit_log |
| AAMVA D20 | `titling_registrations` field names align with AAMVA data element dictionary |
| NHTSA vPIC/Recalls | `vehicles` enriched via vPIC decode; recall data in service workflows |
| PCI DSS v4.0 | No card data stored; `payment_methods` holds processor token references only |
| SAE J2540 | Entity naming follows SAE automotive retail terminology (deal jacket, repair order) |

---

## Dealership & Staff

```sql
CREATE TABLE dealerships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    dba_name        TEXT,
    dealer_number   TEXT,
    oem_franchise   TEXT[],
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_code      CHAR(2),
    postal_code     TEXT,
    country_code    CHAR(2) DEFAULT 'US',
    phone           TEXT,
    email           TEXT,
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    tax_rate_bps    INTEGER DEFAULT 0,
    labour_rate_cents BIGINT NOT NULL DEFAULT 15000,
    settings_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"sales_tax_rate_bps":825,"doc_fee_cents":49900,"bay_count":12,
    --   "operating_hours":{"mon":"07:30-18:00"},"fi_product_menu":true}
    qbo_realm_id    TEXT,
    stripe_account_id TEXT,
    star_dealer_id  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('dealer_principal','general_manager','sales_manager',
                        'salesperson','fi_manager','service_manager','service_advisor',
                        'technician','parts_manager','parts_clerk','accounting','bdc_agent')),
    phone           TEXT,
    hourly_rate_cents BIGINT,
    commission_pct  NUMERIC(5,2),
    certifications  TEXT[] DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    hired_date      DATE,
    mfa_enabled     BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dealership_id, email)
);

CREATE INDEX idx_staff_dealership ON staff(dealership_id) WHERE is_active;
```

## Customer & Vehicle Inventory

```sql
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    address_line1   TEXT,
    city            TEXT,
    state_code      CHAR(2),
    postal_code     TEXT,
    date_of_birth   DATE,
    drivers_licence_number TEXT,
    drivers_licence_state CHAR(2),
    is_business     BOOLEAN NOT NULL DEFAULT false,
    business_name   TEXT,
    tax_id          TEXT,
    preferred_contact TEXT NOT NULL DEFAULT 'phone' CHECK (preferred_contact IN ('phone','sms','email')),
    lead_source     TEXT,
    lead_score      SMALLINT,
    ai_churn_risk   NUMERIC(4,3),
    qbo_customer_id TEXT,
    ofac_cleared    BOOLEAN,
    ofac_cleared_at TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_dealership ON customers(dealership_id);
CREATE INDEX idx_customers_phone ON customers(dealership_id, phone);
CREATE INDEX idx_customers_email ON customers(dealership_id, email);

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
    acquisition_source TEXT CHECK (acquisition_source IN ('factory','auction','trade_in','purchase','consignment')),
    acquisition_date DATE,
    acquisition_cost_cents BIGINT,
    asking_price_cents BIGINT,
    internet_price_cents BIGINT,
    invoice_price_cents BIGINT,
    msrp_cents      BIGINT,
    days_in_stock   INTEGER,
    ai_market_price_cents BIGINT,
    ai_pricing_json JSONB,
    -- Example: {"market_avg_cents":2850000,"competitor_low":2699000,"competitor_high":3100000,
    --   "days_to_turn_estimate":28,"confidence":0.82,"updated_at":"2026-05-25T10:00:00Z"}
    nhtsa_decoded_json JSONB,
    photo_urls      TEXT[] DEFAULT '{}',
    features        TEXT[] DEFAULT '{}',
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dealership_id, vin)
);

CREATE INDEX idx_vehicles_status ON vehicles(dealership_id, status, condition);
CREATE INDEX idx_vehicles_make_model ON vehicles(dealership_id, make, model, year);
CREATE INDEX idx_vehicles_stock ON vehicles(dealership_id, stock_number);
CREATE INDEX idx_vehicles_features ON vehicles USING GIN (features);
```

## Deals & F&I

```sql
CREATE TABLE deals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    deal_number     SERIAL,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    vehicle_id      UUID NOT NULL REFERENCES vehicles(id),
    salesperson_id  UUID REFERENCES staff(id),
    fi_manager_id   UUID REFERENCES staff(id),
    sales_manager_id UUID REFERENCES staff(id),
    deal_type       TEXT NOT NULL CHECK (deal_type IN ('retail','lease','wholesale','fleet')),
    deal_status     TEXT NOT NULL DEFAULT 'pending'
                    CHECK (deal_status IN ('pending','desking','fi_review','contract_signed',
                                           'funded','delivered','unwound','cancelled')),
    selling_price_cents BIGINT NOT NULL,
    trade_allowance_cents BIGINT NOT NULL DEFAULT 0,
    trade_vehicle_id UUID REFERENCES vehicles(id),
    trade_payoff_cents BIGINT NOT NULL DEFAULT 0,
    doc_fee_cents   BIGINT NOT NULL DEFAULT 0,
    total_taxes_cents BIGINT NOT NULL DEFAULT 0,
    total_fees_cents BIGINT NOT NULL DEFAULT 0,
    total_fi_cents  BIGINT NOT NULL DEFAULT 0,
    total_deal_cents BIGINT NOT NULL DEFAULT 0,
    front_gross_cents BIGINT NOT NULL DEFAULT 0,
    back_gross_cents BIGINT NOT NULL DEFAULT 0,
    total_gross_cents BIGINT NOT NULL DEFAULT 0,
    down_payment_cents BIGINT NOT NULL DEFAULT 0,
    amount_financed_cents BIGINT NOT NULL DEFAULT 0,
    monthly_payment_cents BIGINT,
    term_months     SMALLINT,
    apr_bps         INTEGER,
    lender_id       UUID REFERENCES lender_submissions(id),
    buyers_guide_type TEXT CHECK (buyers_guide_type IN ('as_is','limited_warranty','full_warranty')),
    ftc_disclosures_json JSONB NOT NULL DEFAULT '{}',
    rdr_submitted_at TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    funded_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deals_dealership ON deals(dealership_id, deal_status);
CREATE INDEX idx_deals_customer ON deals(customer_id);
CREATE INDEX idx_deals_vehicle ON deals(vehicle_id);
CREATE INDEX idx_deals_salesperson ON deals(salesperson_id);

CREATE TABLE fi_products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    name            TEXT NOT NULL,
    product_type    TEXT NOT NULL CHECK (product_type IN ('extended_warranty','gap','tire_wheel',
                        'paint_protection','theft_deterrent','prepaid_maintenance',
                        'key_replacement','windshield','dent_repair','credit_life',
                        'disability','other')),
    provider        TEXT NOT NULL,
    cost_cents      BIGINT NOT NULL,
    retail_cents    BIGINT NOT NULL,
    max_markup_cents BIGINT,
    term_months     SMALLINT,
    mileage_limit   INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fi_products_dealership ON fi_products(dealership_id) WHERE is_active;

CREATE TABLE deal_fi_products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    fi_product_id   UUID NOT NULL REFERENCES fi_products(id),
    selling_price_cents BIGINT NOT NULL,
    cost_cents      BIGINT NOT NULL,
    term_months     SMALLINT,
    deductible_cents BIGINT,
    ai_recommended  BOOLEAN NOT NULL DEFAULT false,
    ai_confidence   NUMERIC(4,3),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deal_fi ON deal_fi_products(deal_id);

CREATE TABLE lender_submissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    lender_name     TEXT NOT NULL,
    lender_code     TEXT,
    submission_type TEXT NOT NULL CHECK (submission_type IN ('credit_app','contract','stipulation')),
    status          TEXT NOT NULL DEFAULT 'submitted'
                    CHECK (status IN ('submitted','conditionally_approved','approved',
                                      'declined','funded','cancelled')),
    apr_bps         INTEGER,
    term_months     SMALLINT,
    amount_financed_cents BIGINT,
    monthly_payment_cents BIGINT,
    conditions      TEXT[],
    routeone_ref    TEXT,
    dealertrack_ref TEXT,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    decision_at     TIMESTAMPTZ,
    funded_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lender_deal ON lender_submissions(deal_id);
CREATE INDEX idx_lender_status ON lender_submissions(dealership_id, status);
```

## Compliance & Regulatory

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
    screened_by     UUID REFERENCES staff(id),
    screened_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_customer ON compliance_screenings(customer_id);
CREATE INDEX idx_compliance_deal ON compliance_screenings(deal_id);
CREATE INDEX idx_compliance_dealership ON compliance_screenings(dealership_id, screening_type);

CREATE TABLE cash_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    deal_id         UUID REFERENCES deals(id),
    cash_amount_cents BIGINT NOT NULL,
    form_8300_filed BOOLEAN NOT NULL DEFAULT false,
    filed_at        TIMESTAMPTZ,
    filing_deadline TIMESTAMPTZ NOT NULL,
    irs_confirmation TEXT,
    prepared_by     UUID REFERENCES staff(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cash_reports_dealership ON cash_reports(dealership_id);
CREATE INDEX idx_cash_reports_unfiled ON cash_reports(dealership_id) WHERE NOT form_8300_filed;

CREATE TABLE titling_registrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    vehicle_id      UUID NOT NULL REFERENCES vehicles(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    state_code      CHAR(2) NOT NULL,
    title_number    TEXT,
    registration_number TEXT,
    plate_number    TEXT,
    title_status    TEXT NOT NULL DEFAULT 'pending'
                    CHECK (title_status IN ('pending','submitted','processing','completed','rejected')),
    submission_method TEXT CHECK (submission_method IN ('electronic','mail','in_person')),
    fees_cents      BIGINT NOT NULL DEFAULT 0,
    taxes_cents     BIGINT NOT NULL DEFAULT 0,
    submitted_at    TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    aamva_ref       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_titling_deal ON titling_registrations(deal_id);
CREATE INDEX idx_titling_status ON titling_registrations(dealership_id, title_status);
```

## Service & Parts

```sql
CREATE TABLE repair_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    ro_number       SERIAL,
    vehicle_id      UUID NOT NULL REFERENCES vehicles(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    service_advisor_id UUID REFERENCES staff(id),
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
CREATE INDEX idx_ro_customer ON repair_orders(customer_id);

CREATE TABLE service_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repair_order_id UUID NOT NULL REFERENCES repair_orders(id) ON DELETE CASCADE,
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    item_type       TEXT NOT NULL CHECK (item_type IN ('labour','part','sublet','fee','discount')),
    op_code         TEXT,
    description     TEXT NOT NULL,
    technician_id   UUID REFERENCES staff(id),
    part_id         UUID REFERENCES parts(id),
    quantity        NUMERIC(10,2) NOT NULL DEFAULT 1,
    unit_price_cents BIGINT NOT NULL DEFAULT 0,
    cost_cents      BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL DEFAULT 0,
    hours_estimated NUMERIC(5,2),
    hours_actual    NUMERIC(5,2),
    is_warranty     BOOLEAN NOT NULL DEFAULT false,
    warranty_claim_number TEXT,
    is_approved     BOOLEAN NOT NULL DEFAULT false,
    approved_at     TIMESTAMPTZ,
    taxable         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sli_ro ON service_line_items(repair_order_id);
CREATE INDEX idx_sli_technician ON service_line_items(technician_id);

CREATE TABLE parts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    part_number     TEXT NOT NULL,
    description     TEXT NOT NULL,
    category        TEXT,
    brand           TEXT,
    supplier        TEXT,
    is_oem          BOOLEAN NOT NULL DEFAULT false,
    cost_cents      BIGINT NOT NULL DEFAULT 0,
    retail_cents    BIGINT NOT NULL DEFAULT 0,
    qty_on_hand     INTEGER NOT NULL DEFAULT 0,
    qty_reserved    INTEGER NOT NULL DEFAULT 0,
    reorder_point   INTEGER NOT NULL DEFAULT 0,
    reorder_qty     INTEGER NOT NULL DEFAULT 0,
    bin_location    TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dealership_id, part_number)
);

CREATE INDEX idx_parts_dealership ON parts(dealership_id) WHERE is_active;
```

## Leads & Appointments

```sql
CREATE TABLE leads (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    customer_id     UUID REFERENCES customers(id),
    assigned_to     UUID REFERENCES staff(id),
    source          TEXT CHECK (source IN ('website','phone','walk_in','referral','third_party',
                                           'oem','social','chat','email')),
    lead_type       TEXT NOT NULL CHECK (lead_type IN ('sales','service','parts','trade_in')),
    status          TEXT NOT NULL DEFAULT 'new'
                    CHECK (status IN ('new','contacted','qualified','appointment_set',
                                      'shown','demo','negotiating','sold','lost')),
    vehicle_interest_id UUID REFERENCES vehicles(id),
    notes           TEXT,
    ai_score        SMALLINT,
    ai_next_action  TEXT,
    first_response_at TIMESTAMPTZ,
    last_activity_at TIMESTAMPTZ,
    lost_reason     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leads_dealership ON leads(dealership_id, status);
CREATE INDEX idx_leads_assigned ON leads(assigned_to) WHERE status NOT IN ('sold','lost');

CREATE TABLE appointments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    vehicle_id      UUID REFERENCES vehicles(id),
    staff_id        UUID REFERENCES staff(id),
    department      TEXT NOT NULL CHECK (department IN ('sales','service','fi')),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'scheduled'
                    CHECK (status IN ('scheduled','confirmed','checked_in','completed','no_show','cancelled')),
    notes           TEXT,
    reminder_sent_at TIMESTAMPTZ,
    star_appointment_id TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appt_dealership_date ON appointments(dealership_id, department, scheduled_start);
CREATE INDEX idx_appt_customer ON appointments(customer_id);
```

## Payments & Accounting

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
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (deal_id IS NOT NULL OR repair_order_id IS NOT NULL)
);

CREATE INDEX idx_payments_deal ON payments(deal_id);
CREATE INDEX idx_payments_ro ON payments(repair_order_id);
CREATE INDEX idx_payments_dealership ON payments(dealership_id, status);

CREATE TABLE gl_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    account_number  TEXT NOT NULL,
    account_name    TEXT NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN ('asset','liability','equity','revenue','expense')),
    parent_id       UUID REFERENCES gl_accounts(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    qbo_account_id  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dealership_id, account_number)
);

CREATE TABLE journal_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    entry_number    SERIAL,
    entry_date      DATE NOT NULL,
    memo            TEXT,
    source_type     TEXT CHECK (source_type IN ('deal','repair_order','payment','adjustment','payroll')),
    source_id       UUID,
    posted          BOOLEAN NOT NULL DEFAULT false,
    posted_at       TIMESTAMPTZ,
    qbo_journal_id  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE journal_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_entry_id UUID NOT NULL REFERENCES journal_entries(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES gl_accounts(id),
    debit_cents     BIGINT NOT NULL DEFAULT 0,
    credit_cents    BIGINT NOT NULL DEFAULT 0,
    memo            TEXT,
    department      TEXT CHECK (department IN ('sales_new','sales_used','service','parts','fi','admin'))
);

CREATE INDEX idx_jl_entry ON journal_lines(journal_entry_id);
CREATE INDEX idx_jl_account ON journal_lines(account_id);
```

## Integrations & Audit

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL REFERENCES dealerships(id),
    provider        TEXT NOT NULL CHECK (provider IN ('quickbooks','stripe','routeone','dealertrack_fi',
                        'fortellis','tekion_apc','nhtsa','partstech','carfax','smartcar',
                        'vauto','twilio','sendgrid','oem_data')),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','inactive','error')),
    credentials_json JSONB NOT NULL DEFAULT '{}',
    config_json     JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dealership_id, provider)
);

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
    user_agent      TEXT,
    mfa_verified    BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_dealership ON audit_log(dealership_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_id, created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Dealership & Staff | 2 | dealerships, staff |
| Customer & Vehicle | 2 | customers, vehicles |
| Deals & F&I | 4 | deals, fi_products, deal_fi_products, lender_submissions |
| Compliance | 3 | compliance_screenings, cash_reports, titling_registrations |
| Service & Parts | 3 | repair_orders, service_line_items, parts |
| Leads & Appointments | 2 | leads, appointments |
| Payments & Accounting | 4 | payments, gl_accounts, journal_entries, journal_lines |
| Integrations & Audit | 2 | integrations, audit_log |
| **Total** | **22** | |

---

## Key Design Decisions

1. **STAR Domain Model alignment** — deals, repair orders, appointments, and vehicle inventory use field names and status progressions that map to the STAR 2026 JSON schemas. This enables native data exchange with OEM systems and certified integration partners without field-level translation layers.

2. **Deals as the central sales entity** — a deal links customer, vehicle, salesperson, F&I manager, lender submission, trade-in vehicle, and compliance records. Front/back gross fields enable the fundamental DMS metric: total gross per deal per salesperson.

3. **F&I products as a catalogue with deal-level junction** — `fi_products` defines the dealership's available products (warranties, GAP, maintenance plans); `deal_fi_products` records which products were sold on each deal with actual selling price and cost, enabling F&I penetration and profit-per-product reporting.

4. **Lender submissions track the full financing lifecycle** — from credit application through conditional approval, final approval, and funding. RouteOne and Dealertrack references enable reconciliation with external F&I platforms.

5. **OFAC screening with 10-year retention** — `compliance_screenings` stores every SDN check with results and match details. The 10-year OFAC statute of limitations (extended March 2025) requires long-term retention; the table supports this without relying on audit log retention alone.

6. **IRS 8300 as a first-class entity** — `cash_reports` tracks cash transactions over $10,000 with filing deadlines, confirmation numbers, and filed status. A dashboard query for unfiled reports with approaching deadlines is a simple WHERE clause.

7. **Titling aligned with AAMVA D20** — field names (`title_number`, `registration_number`, `plate_number`, `state_code`) match the AAMVA data element dictionary for interoperability with state DMV electronic titling systems.

8. **Vehicle inventory with AI pricing** — `ai_market_price_cents` and `ai_pricing_json` capture market-based pricing intelligence (competitor range, days-to-turn estimate, confidence) without embedding the pricing model in the schema.

9. **Department-level journal lines** — `journal_lines.department` enables the chart of accounts to generate departmental P&L statements (new vehicles, used vehicles, service, parts, F&I, admin) — the standard DMS accounting view.

10. **MFA tracking in audit log** — `audit_log.mfa_verified` supports FTC Safeguards Rule compliance by documenting whether multi-factor authentication was in effect for each DMS action.

11. **Lead scoring with AI** — `leads.ai_score` and `leads.ai_next_action` capture AI-generated lead prioritisation and recommended follow-up actions, enabling the BDC (Business Development Centre) to route leads based on predicted conversion probability.

12. **Partitioned audit log with 10+ year retention** — time-range partitioning supports the OFAC/Safeguards Rule requirement for long-term audit trail retention while keeping current-period queries fast.
