# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Car Dealership DMS · Created: 2026-05-25

## Philosophy

This model treats every state change in the dealership as an immutable event — a deal advancing from desking to F&I review, a lender returning an approval, an OFAC screening clearing, a vehicle arriving from auction, a service advisor opening a repair order. The `event_store` is the single source of truth; all queryable state is materialised into read-model tables via projections (CQRS pattern).

The automotive retail domain is uniquely well-suited to event sourcing because regulatory compliance demands a complete, tamper-proof history of every transaction. The FTC Safeguards Rule requires audit trails for all access to customer financial data. OFAC requires 10-year retention of screening records. IRS 8300 requires documentation of cash transaction reporting. Rather than bolting audit logging onto a mutable data model, this architecture makes the audit trail the data model itself — compliance is structural, not operational.

Events follow the CloudEvents specification for interoperability with integration partners (Fortellis, Tekion APC, RouteOne webhooks, Stripe events, QuickBooks change notifications). The event stream is the natural integration surface: inbound events from lenders and payment processors are written to the same store, and outbound events can be published to OEM systems via STAR-aligned schemas.

**Best for:** Franchise dealer groups and compliance-sensitive operations that need tamper-proof audit trails, temporal queries ("what did the deal look like when the lender approved it?"), and the ability to derive new analytics from historical events as reporting needs evolve.

**Trade-offs:**
- (+) Complete, immutable audit trail — FTC Safeguards Rule compliance by construction
- (+) 10-year OFAC retention is natural (events are never deleted)
- (+) Temporal queries — replay to any point in time for dispute resolution or compliance review
- (+) New read models can be built from existing events without schema migration
- (+) Natural integration surface for Fortellis events, Stripe webhooks, and lender notifications
- (-) Eventual consistency — read models lag behind event writes
- (-) Event schema evolution requires careful versioning (upcasting)
- (-) Higher storage volume than state-only models
- (-) Debugging requires event replay rather than inspecting current state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0.2 | `event_store` columns: `ce_source`, `ce_type`, `ce_specversion`, `ce_time` |
| STAR Domain Model 2026 | Event payloads use STAR-aligned field names for deals, ROs, and vehicles |
| ISO 3779 (VIN) | VIN in vehicle stream events and `rm_inventory` read model |
| FTC Safeguards Rule | Event store IS the audit trail — MFA status in event metadata |
| OFAC SDN (31 CFR 500+) | OFAC screening events on deal stream; immutable with 10-year natural retention |
| IRS Form 8300 | Cash reporting events on deal stream with filing status tracking |
| AAMVA D20 | Titling events use AAMVA-aligned field names |
| STAR Sales Process API | Deal events map to STAR Deal API lifecycle stages |
| PCI DSS v4.0 | Payment events reference Stripe tokens only — no card data in event store |
| Stripe | Payment events mirror Stripe webhook event types |
| RouteOne | Lender decision events capture RouteOne reference IDs |

---

## Infrastructure Tables

### event_store

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
                        'deal','vehicle','customer','service','part',
                        'payment','compliance','accounting','ai')),
    stream_id       UUID NOT NULL,
    version         INTEGER NOT NULL,
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',

    -- CloudEvents envelope
    ce_source       TEXT NOT NULL DEFAULT '/car-dealership-dms',
    ce_type         TEXT NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),

    dealership_id   UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('staff','customer','system','integration','ai')),
    mfa_verified    BOOLEAN,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_es_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_es_dealership ON event_store(dealership_id, created_at DESC);
CREATE INDEX idx_es_type ON event_store(event_type, created_at DESC);
CREATE INDEX idx_es_actor ON event_store(actor_id, created_at DESC);
```

### stream_snapshots

```sql
CREATE TABLE stream_snapshots (
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    version         INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

### projection_checkpoints

```sql
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Catalogue

### Deal Stream (`stream_type = 'deal'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `deal_created` | dealership_id, customer_id, vehicle_id, salesperson_id, deal_type | Initial deal creation |
| `deal_desked` | selling_price_cents, trade_allowance_cents, trade_payoff_cents, doc_fee_cents, taxes, fees | Pricing structure set |
| `trade_in_appraised` | trade_vehicle_id, vin, acv_cents, allowance_cents, payoff_cents, ai_appraisal | Trade-in evaluation |
| `fi_manager_assigned` | fi_manager_id | F&I handoff |
| `fi_product_added` | product_id, product_type, name, selling_price_cents, cost_cents, term_months, ai_recommended | F&I product sold |
| `fi_product_removed` | product_id, reason | F&I product removed from deal |
| `lender_submitted` | lender_name, lender_code, submission_type, amount_financed_cents, routeone_ref, dealertrack_ref | Credit app sent |
| `lender_approved` | lender_name, apr_bps, term_months, monthly_payment_cents, conditions[] | Lender decision |
| `lender_declined` | lender_name, reason | Lender decline |
| `lender_funded` | lender_name, funded_at, amount_cents | Deal funded |
| `ofac_screened` | result, screened_name, sdn_list_date, match_details | OFAC SDN check |
| `cash_reported` | cash_amount_cents, filing_deadline | IRS 8300 flagged |
| `form_8300_filed` | irs_confirmation, filed_at | IRS 8300 submitted |
| `ftc_disclosure_signed` | disclosure_type, signed_at | Buyers Guide, privacy notice, etc. |
| `contract_signed` | signed_at, contract_type | Purchase/lease agreement executed |
| `title_submitted` | state, method, aamva_ref, fees_cents, taxes_cents | Title application |
| `title_completed` | title_number, registration_number, plate_number | Title issued |
| `rdr_submitted` | star_rdr_id, submitted_at | Retail Delivery Report to OEM |
| `vehicle_delivered` | delivered_at, odometer | Customer takes delivery |
| `deal_unwound` | reason, unwound_at | Deal reversal |
| `deal_cancelled` | reason | Deal abandoned |

### Vehicle Stream (`stream_type = 'vehicle'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `vehicle_acquired` | vin, year, make, model, trim, condition, source, cost_cents | Added to inventory |
| `vehicle_decoded` | nhtsa_data, year, make, model, body_style, engine | NHTSA vPIC decode |
| `vehicle_priced` | asking_cents, internet_cents, ai_market_cents, ai_analysis | Pricing set |
| `vehicle_repriced` | old_price_cents, new_price_cents, reason, days_in_stock | Price adjustment |
| `vehicle_reconditioned` | recon_cost_cents, work_performed[], completed_at | Recon completed |
| `vehicle_photographed` | photo_urls[], ai_photo_analysis | Photos uploaded |
| `vehicle_listed` | listing_channels[], listed_at | Published to marketplaces |
| `recall_checked` | nhtsa_campaigns[], checked_at | NHTSA recall lookup |
| `vehicle_sold` | deal_id, sold_at | Marked sold |
| `vehicle_wholesaled` | buyer, price_cents, auction_name | Sent to wholesale |
| `vehicle_traded_in` | deal_id, customer_id, allowance_cents, acv_cents | Received as trade |

### Customer Stream (`stream_type = 'customer'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `customer_created` | first_name, last_name, email, phone, source | Initial record |
| `customer_updated` | changed_fields | Profile changes |
| `lead_created` | source, type, vehicle_interest_id, assigned_to | Lead capture |
| `lead_scored` | ai_score, ai_next_action, factors[] | AI lead scoring |
| `lead_contacted` | channel, by, notes | Follow-up activity |
| `lead_converted` | deal_id | Lead became a deal |
| `lead_lost` | reason | Lead abandoned |
| `appointment_scheduled` | department, scheduled_start, staff_id, vehicle_id | Appointment booked |
| `appointment_completed` | completed_at | Appointment finished |
| `appointment_no_show` | | No-show recorded |
| `churn_risk_flagged` | risk_score, factors[], last_visit | AI churn detection |
| `message_sent` | channel, type, body, external_id | Outbound communication |
| `message_received` | channel, body | Inbound communication |

### Service Stream (`stream_type = 'service'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `ro_opened` | vehicle_id, customer_id, service_advisor_id, odometer_in, concern | Repair order created |
| `line_item_added` | line_id, item_type, op_code, description, technician_id, price_cents, hours_estimated | Service line added |
| `line_approved` | line_id, approved_at | Customer approved line |
| `work_started` | technician_id, line_id, started_at | Tech begins task |
| `work_completed` | technician_id, line_id, completed_at, hours_actual | Task finished |
| `parts_requested` | line_id, part_number, quantity | Parts pull request |
| `parts_installed` | line_id, part_number, quantity | Parts used |
| `ro_completed` | completed_at, odometer_out | RO finished |
| `ro_invoiced` | subtotal_cents, tax_cents, total_cents, qbo_invoice_id | Invoice generated |
| `warranty_claimed` | line_id, claim_number, oem | Warranty claim submitted |

### Payment Stream (`stream_type = 'payment'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `payment_initiated` | deal_id or ro_id, amount_cents, method, stripe_pi_id | Payment started |
| `payment_succeeded` | amount_cents, paid_at, stripe_pi_id | Payment confirmed |
| `payment_failed` | amount_cents, failure_reason | Payment failed |
| `payment_refunded` | refund_cents, reason, stripe_refund_id | Refund issued |
| `qbo_payment_synced` | qbo_payment_id, synced_at | QuickBooks sync |

### Compliance Stream (`stream_type = 'compliance'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `ofac_screening_requested` | customer_id, deal_id, screened_name | SDN check initiated |
| `ofac_screening_completed` | result, match_details, sdn_list_date | SDN result received |
| `safeguards_audit_generated` | audit_period, findings[], generated_at | FTC Safeguards audit |
| `data_access_logged` | staff_id, data_type, customer_id, mfa_verified | NPI access record (GLBA) |
| `privacy_notice_delivered` | customer_id, channel, notice_type | GLBA privacy notice |

### Accounting Stream (`stream_type = 'accounting'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `journal_entry_posted` | entry_number, entry_date, memo, source_type, source_id, lines[] | GL entry |
| `account_created` | account_number, account_name, account_type, department | New GL account |
| `qbo_journal_synced` | qbo_journal_id, synced_at | QuickBooks sync |
| `month_closed` | period, closed_at, closed_by | Period close |

### AI Stream (`stream_type = 'ai'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `pricing_recommendation` | vehicle_id, market_avg_cents, recommended_cents, confidence | AI pricing |
| `appraisal_recommendation` | vehicle_id, acv_cents, confidence, factors[] | AI trade appraisal |
| `fi_recommendation` | deal_id, product_types[], confidence, customer_profile_factors[] | F&I product suggestion |
| `lead_scoring_completed` | customer_id, score, next_action, factors[] | AI lead score |
| `churn_prediction` | customer_id, risk_score, factors[] | Churn risk |
| `service_demand_forecast` | dealership_id, period, forecast_json | Service lane prediction |
| `photo_optimisation` | vehicle_id, original_urls[], enhanced_urls[], captions[] | CV photo enhancement |

---

## Read Model Tables

### rm_deals

```sql
CREATE TABLE rm_deals (
    id              UUID PRIMARY KEY,
    dealership_id   UUID NOT NULL,
    deal_number     INTEGER NOT NULL,
    customer_id     UUID NOT NULL,
    customer_name   TEXT,
    customer_phone  TEXT,
    vehicle_id      UUID NOT NULL,
    vin             TEXT,
    vehicle_display TEXT,
    salesperson_id  UUID,
    salesperson_name TEXT,
    fi_manager_id   UUID,
    deal_type       TEXT NOT NULL,
    deal_status     TEXT NOT NULL,
    pricing         JSONB NOT NULL DEFAULT '{}',
    trade_in        JSONB,
    fi_products     JSONB NOT NULL DEFAULT '[]',
    lender_submissions JSONB NOT NULL DEFAULT '[]',
    selected_lender TEXT,
    compliance      JSONB NOT NULL DEFAULT '{}',
    titling         JSONB,
    disclosures     JSONB NOT NULL DEFAULT '{}',
    front_gross_cents BIGINT NOT NULL DEFAULT 0,
    back_gross_cents BIGINT NOT NULL DEFAULT 0,
    total_gross_cents BIGINT NOT NULL DEFAULT 0,
    paid_cents      BIGINT NOT NULL DEFAULT 0,
    rdr_submitted_at TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    funded_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_deals_dealership ON rm_deals(dealership_id, deal_status);
CREATE INDEX idx_rm_deals_salesperson ON rm_deals(salesperson_id);
CREATE INDEX idx_rm_deals_vehicle ON rm_deals(vehicle_id);
```

### rm_inventory

```sql
CREATE TABLE rm_inventory (
    id              UUID PRIMARY KEY,
    dealership_id   UUID NOT NULL,
    vin             CHAR(17) NOT NULL,
    stock_number    TEXT,
    year            SMALLINT NOT NULL,
    make            TEXT NOT NULL,
    model           TEXT NOT NULL,
    trim_level      TEXT,
    condition       TEXT NOT NULL,
    status          TEXT NOT NULL,
    odometer_miles  INTEGER,
    exterior_colour TEXT,
    acquisition_source TEXT,
    acquisition_cost_cents BIGINT,
    asking_price_cents BIGINT,
    internet_price_cents BIGINT,
    ai_market_price_cents BIGINT,
    days_in_stock   INTEGER,
    recon_cost_cents BIGINT NOT NULL DEFAULT 0,
    photo_urls      TEXT[],
    features        TEXT[],
    recall_status   TEXT,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_inv_dealership ON rm_inventory(dealership_id, status, condition);
CREATE INDEX idx_rm_inv_make ON rm_inventory(dealership_id, make, model);
CREATE INDEX idx_rm_inv_days ON rm_inventory(dealership_id, days_in_stock DESC) WHERE status = 'in_stock';
```

### rm_revenue

```sql
CREATE TABLE rm_revenue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dealership_id   UUID NOT NULL,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','weekly','monthly')),
    department      TEXT NOT NULL CHECK (department IN ('sales_new','sales_used','service','parts','fi')),

    revenue_cents        BIGINT NOT NULL DEFAULT 0,
    cost_cents           BIGINT NOT NULL DEFAULT 0,
    gross_profit_cents   BIGINT NOT NULL DEFAULT 0,
    gross_margin_pct     NUMERIC(5,2) NOT NULL DEFAULT 0,
    unit_count           INTEGER NOT NULL DEFAULT 0,
    avg_gross_per_unit   BIGINT NOT NULL DEFAULT 0,

    -- Sales-specific
    deals_closed    INTEGER NOT NULL DEFAULT 0,
    avg_selling_price_cents BIGINT NOT NULL DEFAULT 0,
    avg_days_to_turn INTEGER,
    fi_penetration_pct NUMERIC(5,2),
    fi_per_deal_cents BIGINT,

    -- Service-specific
    car_count       INTEGER NOT NULL DEFAULT 0,
    avg_ro_cents    BIGINT NOT NULL DEFAULT 0,
    hours_billed    NUMERIC(8,2) NOT NULL DEFAULT 0,
    hours_available NUMERIC(8,2) NOT NULL DEFAULT 0,
    service_absorption_pct NUMERIC(5,2),

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dealership_id, period, period_type, department)
);

CREATE INDEX idx_rm_rev ON rm_revenue(dealership_id, period_type, department, period DESC);
```

### rm_compliance

```sql
CREATE TABLE rm_compliance (
    dealership_id   UUID PRIMARY KEY,

    -- OFAC
    total_screenings_ytd INTEGER NOT NULL DEFAULT 0,
    clear_screenings_ytd INTEGER NOT NULL DEFAULT 0,
    potential_matches_ytd INTEGER NOT NULL DEFAULT 0,
    last_screening_at TIMESTAMPTZ,

    -- IRS 8300
    cash_reports_ytd INTEGER NOT NULL DEFAULT 0,
    unfiled_8300s   INTEGER NOT NULL DEFAULT 0,
    next_filing_deadline TIMESTAMPTZ,

    -- FTC Safeguards
    staff_with_mfa  INTEGER NOT NULL DEFAULT 0,
    staff_without_mfa INTEGER NOT NULL DEFAULT 0,
    last_safeguards_audit TIMESTAMPTZ,
    npi_access_events_30d INTEGER NOT NULL DEFAULT 0,

    -- Titling
    titles_pending  INTEGER NOT NULL DEFAULT 0,
    titles_overdue  INTEGER NOT NULL DEFAULT 0,
    avg_title_days  NUMERIC(5,1),

    -- Privacy
    privacy_notices_sent_ytd INTEGER NOT NULL DEFAULT 0,

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### rm_dealership_dashboard

```sql
CREATE TABLE rm_dealership_dashboard (
    dealership_id   UUID PRIMARY KEY,
    dealership_name TEXT NOT NULL,

    -- Sales today
    active_deals    INTEGER NOT NULL DEFAULT 0,
    deals_in_fi     INTEGER NOT NULL DEFAULT 0,
    deals_pending_funding INTEGER NOT NULL DEFAULT 0,
    vehicles_delivered_today INTEGER NOT NULL DEFAULT 0,

    -- Inventory
    total_new_inventory INTEGER NOT NULL DEFAULT 0,
    total_used_inventory INTEGER NOT NULL DEFAULT 0,
    avg_days_in_stock_new NUMERIC(5,1),
    avg_days_in_stock_used NUMERIC(5,1),
    vehicles_over_60_days INTEGER NOT NULL DEFAULT 0,

    -- Service
    open_ros        INTEGER NOT NULL DEFAULT 0,
    waiting_parts   INTEGER NOT NULL DEFAULT 0,
    techs_clocked_in INTEGER NOT NULL DEFAULT 0,
    today_service_revenue_cents BIGINT NOT NULL DEFAULT 0,

    -- Leads
    new_leads_today INTEGER NOT NULL DEFAULT 0,
    leads_awaiting_response INTEGER NOT NULL DEFAULT 0,
    avg_response_time_minutes NUMERIC(8,1),

    -- Compliance alerts
    compliance_alerts JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type":"ofac_potential_match","deal_id":"uuid","severity":"high"},
    --   {"type":"8300_filing_due","deadline":"2026-06-01","severity":"medium"}]

    -- AI insights
    ai_insights     JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type":"repricing","message":"5 used vehicles over 45 days — recommend 5% reduction"},
    --   {"type":"churn_risk","message":"12 customers flagged for retention outreach"}]

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Event Replay Queries

### Reconstruct deal state at lender approval time

```sql
SELECT payload
FROM event_store
WHERE stream_type = 'deal' AND stream_id = $1
  AND created_at <= (
      SELECT ce_time FROM event_store
      WHERE stream_type = 'deal' AND stream_id = $1 AND event_type = 'lender_approved'
      ORDER BY version DESC LIMIT 1
  )
ORDER BY version ASC;
```

### OFAC compliance audit — all screenings in date range

```sql
SELECT ce_time, payload->>'customer_id' AS customer_id,
       payload->>'screened_name' AS screened_name,
       payload->>'result' AS result,
       payload->>'sdn_list_date' AS sdn_list_date,
       metadata->>'mfa_verified' AS mfa_verified
FROM event_store
WHERE stream_type = 'compliance'
  AND event_type = 'ofac_screening_completed'
  AND dealership_id = $1
  AND ce_time BETWEEN $2 AND $3
ORDER BY ce_time;
```

### F&I product penetration trend

```sql
SELECT date_trunc('month', ce_time) AS month,
       payload->>'product_type' AS product_type,
       COUNT(*) AS products_sold,
       SUM((payload->>'selling_price_cents')::bigint) AS revenue_cents,
       SUM((payload->>'selling_price_cents')::bigint - (payload->>'cost_cents')::bigint) AS margin_cents
FROM event_store
WHERE stream_type = 'deal'
  AND event_type = 'fi_product_added'
  AND dealership_id = $1
  AND ce_time >= now() - interval '12 months'
GROUP BY month, payload->>'product_type'
ORDER BY month, revenue_cents DESC;
```

### Vehicle days-to-turn analysis

```sql
WITH acquired AS (
    SELECT stream_id, ce_time AS acquired_at,
           payload->>'condition' AS condition,
           (payload->>'cost_cents')::bigint AS cost_cents
    FROM event_store
    WHERE stream_type = 'vehicle' AND event_type = 'vehicle_acquired'
      AND dealership_id = $1 AND ce_time >= now() - interval '6 months'
),
sold AS (
    SELECT stream_id, ce_time AS sold_at
    FROM event_store
    WHERE stream_type = 'vehicle' AND event_type = 'vehicle_sold'
      AND dealership_id = $1 AND ce_time >= now() - interval '6 months'
)
SELECT a.condition,
       COUNT(*) AS vehicles_sold,
       AVG(EXTRACT(DAY FROM s.sold_at - a.acquired_at))::integer AS avg_days_to_turn,
       AVG(a.cost_cents) AS avg_acquisition_cost
FROM acquired a
JOIN sold s ON s.stream_id = a.stream_id
GROUP BY a.condition;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_deals, rm_inventory, rm_revenue, rm_compliance, rm_dealership_dashboard |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Nine stream types cover the full dealership** — deal, vehicle, customer, service, part, payment, compliance, accounting, and ai. Each maps to a department or cross-cutting concern. The compliance and accounting streams are separated from deals to enable independent auditing and financial reporting.

2. **Compliance stream as independent audit surface** — OFAC screenings, Safeguards Rule audits, GLBA data access logs, and privacy notices live in their own stream rather than embedded in deal events. Regulatory auditors can query this stream without accessing deal data, and 10-year retention is a natural property of the immutable event store.

3. **Deal events mirror the real workflow** — the deal stream progresses through desking → F&I → lender submission → approval → contract → funding → delivery → RDR. Each step is an event, making the deal timeline auditable and replayable. Unwinding a deal is an event, not a deletion.

4. **FTC Safeguards Rule compliance by construction** — every event records `actor_id`, `actor_type`, `mfa_verified`, and an IP address in metadata. The audit trail required by the Safeguards Rule is the event store itself — there is no separate logging layer to maintain or synchronise.

5. **IRS 8300 as event pair** — `cash_reported` (flagging) and `form_8300_filed` (submission) are separate events on the deal stream. The compliance read model tracks unfiled reports with approaching deadlines.

6. **Vehicle lifecycle from acquisition to disposition** — `vehicle_acquired` → `vehicle_decoded` → `vehicle_priced` → `vehicle_photographed` → `vehicle_listed` → `vehicle_sold` (or `vehicle_wholesaled`) provides complete inventory analytics: days-to-turn, recon cost to gross profit, repricing frequency, and photo impact on time-to-sale.

7. **Revenue read model by department** — `rm_revenue` aggregates by department (sales_new, sales_used, service, parts, fi) and period, enabling the standard DMS departmental P&L view. Service absorption ratio (service gross / total fixed overhead) is derived from service + parts revenue events.

8. **Compliance read model as dashboard** — `rm_compliance` provides at-a-glance regulatory status: OFAC screening counts, unfiled 8300s, MFA coverage, title processing times, and privacy notice delivery. All derived from compliance stream events.

9. **AI events enable feedback loops** — `pricing_recommendation` → `vehicle_priced` → `vehicle_sold` event sequences enable pricing accuracy tracking. `fi_recommendation` → `fi_product_added` sequences measure F&I AI acceptance rates. `lead_scoring_completed` → `lead_converted` (or `lead_lost`) measures lead scoring model accuracy.

10. **MFA tracking in event metadata** — every event records whether the actor had MFA verified, enabling the Safeguards Rule compliance dashboard to report MFA coverage across all DMS interactions without a separate authentication audit system.
