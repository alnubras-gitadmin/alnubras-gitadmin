# 1. System Architecture Overview

## 1.1 Architectural Style
A modular monolith with clean architecture boundaries (ready to split into microservices later) built around business workflows rather than pure CRUD.

- **Frontend (Next.js App Router):** ERP-style SPA shell, server components for initial data hydration, client components for interactive forms/tables.
- **Backend (NestJS):** Domain-driven modules with Application / Domain / Infrastructure layers.
- **Database (PostgreSQL):** Strict relational model, normalized entities, FK constraints, transactional integrity.
- **ORM (Prisma):** Type-safe schema, migrations, repository adapters.
- **API:** REST endpoints grouped by process flow + OpenAPI for contract governance.
- **Auth:** JWT with refresh token flow, OAuth-ready provider abstraction.
- **Cross-cutting:** Audit log, soft deletes, RBAC, idempotency for financial operations.

## 1.2 Core Layers (Backend)
- **Presentation Layer:** Controllers, DTO validation, auth guards.
- **Application Layer:** Use-cases (ConvertLeadToCustomer, ConvertQuotationToOrder, PostInvoice).
- **Domain Layer:** Aggregates, policies, status transition guards.
- **Infrastructure Layer:** Prisma repositories, transactional unit-of-work, integration adapters.

## 1.3 Core ERP Principles Implemented
- Workflow-driven transitions (Lead → Customer → Deal → Quote → Order → Invoice → Payment).
- Immutable accounting milestones (posted invoices/payments append logs, not silent overwrite).
- Soft delete for operational entities, hard constraints for financial integrity.
- Traceability through audit logs and activity timelines.

---

# 2. Module Breakdown with Responsibilities

## 2.1 Identity & Access Module
- User, role, permission matrix.
- Roles: `ADMIN`, `MANAGER`, `SALES`.
- JWT issuance/rotation, OAuth provider integration points.

## 2.2 Leads Module
- Capture inbound leads with source/channel attribution.
- Assignment rules to sales reps.
- Qualification pipeline and loss reasons.
- Conversion endpoint to create customer/contact and optional seed deal.

## 2.3 Party (Customer) Module
- B2B customer account (legal, credit, terms).
- Multiple contacts and addresses.
- Commercial profile: credit limit, payment terms, tax identifiers.

## 2.4 Sales Pipeline Module
- Configurable stage definitions (weighted probability).
- Deals linked to customer and optionally originating lead.
- Stage transition rules and expected-close forecasting.

## 2.5 Quotation Module
- Draft/sent/accepted/rejected lifecycle.
- Line items, taxes, currency, discounts.
- Versioning rules and conversion to Sales Order.

## 2.6 Sales Order Module
- Confirmed commercial commitment.
- Fulfillment progress (none/partial/full).
- Governs invoicing eligibility.

## 2.7 Invoice Module
- Invoice generation from approved sales orders.
- Posting lock (financially recognized state).
- Overdue tracking and accounts receivable visibility.

## 2.8 Payment Module
- Payment capture against posted invoices.
- Partial/full allocation handling.
- Residual balance and status recalculation.

## 2.9 Activity & Task Module
- Calls/meetings/notes/follow-ups.
- Link to lead/customer/deal.
- Timeline feed to support sales execution discipline.

## 2.10 Reporting & Analytics Module
- Funnel conversion by stage/source/owner.
- Revenue and collections trend.
- AR aging buckets (0-30/31-60/61-90/90+).

## 2.11 Audit & Compliance Module
- Append-only event log for key transitions.
- Before/after payload snapshots for sensitive actions.
- Actor + timestamp + correlation ID.

---

# 3. Full Database Schema (PostgreSQL SQL)

```sql
-- ==========================
-- ENUMS
-- ==========================
CREATE TYPE role_type AS ENUM ('ADMIN', 'MANAGER', 'SALES');
CREATE TYPE lead_source AS ENUM ('WEB', 'REFERRAL', 'ADS', 'EMAIL', 'PHONE', 'EVENT', 'OTHER');
CREATE TYPE lead_status AS ENUM ('NEW', 'CONTACTED', 'QUALIFIED', 'LOST', 'CONVERTED');
CREATE TYPE stage_type AS ENUM ('OPEN', 'WON', 'LOST');
CREATE TYPE quotation_status AS ENUM ('DRAFT', 'SENT', 'ACCEPTED', 'REJECTED', 'EXPIRED');
CREATE TYPE order_status AS ENUM ('DRAFT', 'CONFIRMED', 'PARTIALLY_FULFILLED', 'FULFILLED', 'CANCELLED');
CREATE TYPE invoice_status AS ENUM ('DRAFT', 'POSTED', 'PARTIALLY_PAID', 'PAID', 'OVERDUE', 'VOID');
CREATE TYPE payment_status AS ENUM ('PENDING', 'CLEARED', 'FAILED', 'REVERSED');
CREATE TYPE activity_type AS ENUM ('CALL', 'MEETING', 'NOTE', 'TASK', 'EMAIL');
CREATE TYPE address_type AS ENUM ('BILLING', 'SHIPPING', 'OTHER');

-- ==========================
-- SECURITY / USERS
-- ==========================
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  full_name VARCHAR(255) NOT NULL,
  role role_type NOT NULL,
  password_hash TEXT NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);

-- ==========================
-- CORE PARTY
-- ==========================
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_code VARCHAR(40) UNIQUE NOT NULL,
  legal_name VARCHAR(255) NOT NULL,
  trade_name VARCHAR(255),
  tax_number VARCHAR(100),
  website VARCHAR(255),
  phone VARCHAR(50),
  credit_limit NUMERIC(14,2) NOT NULL DEFAULT 0,
  payment_terms_days INT NOT NULL DEFAULT 30,
  account_owner_id UUID REFERENCES users(id),
  created_by UUID NOT NULL REFERENCES users(id),
  updated_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_customers_owner ON customers(account_owner_id);
CREATE INDEX idx_customers_deleted_at ON customers(deleted_at);

CREATE TABLE contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id),
  first_name VARCHAR(120) NOT NULL,
  last_name VARCHAR(120),
  email VARCHAR(255),
  phone VARCHAR(50),
  job_title VARCHAR(120),
  is_primary BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_contacts_customer ON contacts(customer_id);

CREATE TABLE customer_addresses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id),
  address_type address_type NOT NULL,
  line1 VARCHAR(255) NOT NULL,
  line2 VARCHAR(255),
  city VARCHAR(120) NOT NULL,
  state VARCHAR(120),
  postal_code VARCHAR(20),
  country VARCHAR(2) NOT NULL,
  is_default BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_addresses_customer_type ON customer_addresses(customer_id, address_type);

-- ==========================
-- LEADS
-- ==========================
CREATE TABLE leads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_code VARCHAR(40) UNIQUE NOT NULL,
  company_name VARCHAR(255) NOT NULL,
  contact_name VARCHAR(255),
  email VARCHAR(255),
  phone VARCHAR(50),
  source lead_source NOT NULL,
  status lead_status NOT NULL DEFAULT 'NEW',
  estimated_value NUMERIC(14,2),
  assigned_to UUID REFERENCES users(id),
  loss_reason TEXT,
  converted_customer_id UUID REFERENCES customers(id),
  converted_at TIMESTAMPTZ,
  created_by UUID NOT NULL REFERENCES users(id),
  updated_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_leads_status_assigned ON leads(status, assigned_to);
CREATE INDEX idx_leads_source ON leads(source);

-- ==========================
-- PIPELINE / DEALS
-- ==========================
CREATE TABLE pipeline_stages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stage_name VARCHAR(120) NOT NULL,
  stage_order INT NOT NULL,
  probability_pct INT NOT NULL CHECK (probability_pct BETWEEN 0 AND 100),
  stage_type stage_type NOT NULL DEFAULT 'OPEN',
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(stage_name),
  UNIQUE(stage_order)
);

CREATE TABLE deals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  deal_code VARCHAR(40) UNIQUE NOT NULL,
  title VARCHAR(255) NOT NULL,
  customer_id UUID NOT NULL REFERENCES customers(id),
  lead_id UUID REFERENCES leads(id),
  owner_id UUID NOT NULL REFERENCES users(id),
  pipeline_stage_id UUID NOT NULL REFERENCES pipeline_stages(id),
  amount NUMERIC(14,2) NOT NULL,
  currency CHAR(3) NOT NULL DEFAULT 'USD',
  probability_pct INT NOT NULL CHECK (probability_pct BETWEEN 0 AND 100),
  expected_close_date DATE,
  closed_at TIMESTAMPTZ,
  is_won BOOLEAN,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_deals_customer_stage ON deals(customer_id, pipeline_stage_id);
CREATE INDEX idx_deals_owner ON deals(owner_id);

-- ==========================
-- QUOTATIONS
-- ==========================
CREATE TABLE quotations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  quotation_no VARCHAR(50) UNIQUE NOT NULL,
  customer_id UUID NOT NULL REFERENCES customers(id),
  deal_id UUID REFERENCES deals(id),
  issue_date DATE NOT NULL,
  expiry_date DATE,
  status quotation_status NOT NULL DEFAULT 'DRAFT',
  subtotal NUMERIC(14,2) NOT NULL DEFAULT 0,
  tax_total NUMERIC(14,2) NOT NULL DEFAULT 0,
  discount_total NUMERIC(14,2) NOT NULL DEFAULT 0,
  grand_total NUMERIC(14,2) NOT NULL DEFAULT 0,
  currency CHAR(3) NOT NULL DEFAULT 'USD',
  notes TEXT,
  created_by UUID NOT NULL REFERENCES users(id),
  approved_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_quotation_customer_status ON quotations(customer_id, status);

CREATE TABLE quotation_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  quotation_id UUID NOT NULL REFERENCES quotations(id) ON DELETE CASCADE,
  line_no INT NOT NULL,
  item_code VARCHAR(80),
  description TEXT NOT NULL,
  qty NUMERIC(14,3) NOT NULL CHECK (qty > 0),
  unit_price NUMERIC(14,4) NOT NULL CHECK (unit_price >= 0),
  discount_pct NUMERIC(5,2) NOT NULL DEFAULT 0,
  tax_pct NUMERIC(5,2) NOT NULL DEFAULT 0,
  line_subtotal NUMERIC(14,2) NOT NULL,
  line_tax NUMERIC(14,2) NOT NULL,
  line_total NUMERIC(14,2) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(quotation_id, line_no)
);

-- ==========================
-- SALES ORDERS
-- ==========================
CREATE TABLE sales_orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_no VARCHAR(50) UNIQUE NOT NULL,
  quotation_id UUID REFERENCES quotations(id),
  customer_id UUID NOT NULL REFERENCES customers(id),
  deal_id UUID REFERENCES deals(id),
  status order_status NOT NULL DEFAULT 'DRAFT',
  order_date DATE NOT NULL,
  promised_date DATE,
  subtotal NUMERIC(14,2) NOT NULL DEFAULT 0,
  tax_total NUMERIC(14,2) NOT NULL DEFAULT 0,
  grand_total NUMERIC(14,2) NOT NULL DEFAULT 0,
  amount_fulfilled NUMERIC(14,2) NOT NULL DEFAULT 0,
  currency CHAR(3) NOT NULL DEFAULT 'USD',
  created_by UUID NOT NULL REFERENCES users(id),
  confirmed_by UUID REFERENCES users(id),
  confirmed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_orders_customer_status ON sales_orders(customer_id, status);

-- ==========================
-- INVOICES
-- ==========================
CREATE TABLE invoices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_no VARCHAR(50) UNIQUE NOT NULL,
  sales_order_id UUID NOT NULL REFERENCES sales_orders(id),
  customer_id UUID NOT NULL REFERENCES customers(id),
  issue_date DATE NOT NULL,
  due_date DATE NOT NULL,
  status invoice_status NOT NULL DEFAULT 'DRAFT',
  subtotal NUMERIC(14,2) NOT NULL DEFAULT 0,
  tax_total NUMERIC(14,2) NOT NULL DEFAULT 0,
  grand_total NUMERIC(14,2) NOT NULL DEFAULT 0,
  paid_total NUMERIC(14,2) NOT NULL DEFAULT 0,
  balance_due NUMERIC(14,2) NOT NULL DEFAULT 0,
  posted_at TIMESTAMPTZ,
  posted_by UUID REFERENCES users(id),
  currency CHAR(3) NOT NULL DEFAULT 'USD',
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_invoices_customer_status_due ON invoices(customer_id, status, due_date);
CREATE INDEX idx_invoices_order ON invoices(sales_order_id);

-- ==========================
-- PAYMENTS
-- ==========================
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  payment_no VARCHAR(50) UNIQUE NOT NULL,
  customer_id UUID NOT NULL REFERENCES customers(id),
  payment_date DATE NOT NULL,
  amount NUMERIC(14,2) NOT NULL CHECK (amount > 0),
  currency CHAR(3) NOT NULL DEFAULT 'USD',
  payment_method VARCHAR(50) NOT NULL,
  reference_no VARCHAR(120),
  status payment_status NOT NULL DEFAULT 'PENDING',
  received_by UUID REFERENCES users(id),
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_payments_customer_date ON payments(customer_id, payment_date);

CREATE TABLE payment_allocations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  payment_id UUID NOT NULL REFERENCES payments(id) ON DELETE CASCADE,
  invoice_id UUID NOT NULL REFERENCES invoices(id),
  applied_amount NUMERIC(14,2) NOT NULL CHECK (applied_amount > 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(payment_id, invoice_id)
);
CREATE INDEX idx_allocations_invoice ON payment_allocations(invoice_id);

-- ==========================
-- ACTIVITIES
-- ==========================
CREATE TABLE activities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  activity_type activity_type NOT NULL,
  subject VARCHAR(255) NOT NULL,
  details TEXT,
  due_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  owner_id UUID NOT NULL REFERENCES users(id),
  lead_id UUID REFERENCES leads(id),
  customer_id UUID REFERENCES customers(id),
  deal_id UUID REFERENCES deals(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ,
  CONSTRAINT ck_activity_link CHECK (
    lead_id IS NOT NULL OR customer_id IS NOT NULL OR deal_id IS NOT NULL
  )
);
CREATE INDEX idx_activities_owner_due ON activities(owner_id, due_at);
CREATE INDEX idx_activities_deal ON activities(deal_id);

-- ==========================
-- AUDIT LOG
-- ==========================
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  actor_user_id UUID REFERENCES users(id),
  entity_name VARCHAR(80) NOT NULL,
  entity_id UUID NOT NULL,
  action VARCHAR(80) NOT NULL,
  before_data JSONB,
  after_data JSONB,
  correlation_id VARCHAR(100),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_logs(entity_name, entity_id, created_at DESC);
```

---

# 4. API Design (Grouped by Workflow)

## 4.1 Lead Intake & Qualification
- `POST /api/v1/leads/create`
- `POST /api/v1/leads/{leadId}/assign`
- `POST /api/v1/leads/{leadId}/mark-contacted`
- `POST /api/v1/leads/{leadId}/qualify`
- `POST /api/v1/leads/{leadId}/mark-lost`
- `POST /api/v1/leads/{leadId}/convert-to-customer`

## 4.2 Deal Creation & Pipeline Progression
- `POST /api/v1/deals/create-from-lead`
- `POST /api/v1/deals/create-for-customer`
- `POST /api/v1/deals/{dealId}/move-stage`
- `POST /api/v1/deals/{dealId}/mark-won`
- `POST /api/v1/deals/{dealId}/mark-lost`

## 4.3 Quotation Workflow
- `POST /api/v1/quotations/create`
- `POST /api/v1/quotations/{quotationId}/add-item`
- `POST /api/v1/quotations/{quotationId}/send`
- `POST /api/v1/quotations/{quotationId}/accept`
- `POST /api/v1/quotations/{quotationId}/reject`
- `POST /api/v1/quotations/{quotationId}/convert-to-order`

## 4.4 Sales Order Workflow
- `POST /api/v1/orders/create-from-quotation`
- `POST /api/v1/orders/{orderId}/confirm`
- `POST /api/v1/orders/{orderId}/record-fulfillment`
- `POST /api/v1/orders/{orderId}/close-fulfilled`

## 4.5 Invoicing Workflow
- `POST /api/v1/invoices/create-from-order`
- `POST /api/v1/invoices/{invoiceId}/recalculate-taxes`
- `POST /api/v1/invoices/{invoiceId}/post`
- `POST /api/v1/invoices/{invoiceId}/mark-overdue`

## 4.6 Payment & Allocation Workflow
- `POST /api/v1/payments/record`
- `POST /api/v1/payments/{paymentId}/apply`
- `POST /api/v1/payments/{paymentId}/reverse`

## 4.7 Activities & Tasks
- `POST /api/v1/activities/create`
- `POST /api/v1/activities/{activityId}/complete`
- `POST /api/v1/activities/{activityId}/reschedule`

## 4.8 Reporting
- `GET /api/v1/reports/sales-funnel?from=YYYY-MM-DD&to=YYYY-MM-DD`
- `GET /api/v1/reports/revenue?groupBy=month`
- `GET /api/v1/reports/ar-aging?asOf=YYYY-MM-DD`
- `GET /api/v1/reports/conversion-rates?dimension=source`

## 4.9 Workflow Validation Rules (server-enforced)
- Can convert lead only if lead status is `QUALIFIED`.
- Can create order from quotation only if quotation status is `ACCEPTED`.
- Can post invoice only when sales order is `CONFIRMED` or later.
- Can apply payment only to `POSTED`, `PARTIALLY_PAID`, or `OVERDUE` invoices.
- Can mark invoice `PAID` only when `balance_due = 0`.

---

# 5. Backend Folder Structure (NestJS)

```text
backend/
  src/
    main.ts
    app.module.ts
    config/
      env.validation.ts
      swagger.config.ts
      database.config.ts
    common/
      guards/
        jwt-auth.guard.ts
        roles.guard.ts
      decorators/
        current-user.decorator.ts
        roles.decorator.ts
      filters/
        http-exception.filter.ts
      interceptors/
        audit.interceptor.ts
      pipes/
        validation.pipe.ts
      types/
    modules/
      auth/
        auth.module.ts
        application/
          use-cases/
        presentation/
          auth.controller.ts
        infrastructure/
          jwt.service.ts
      users/
      leads/
        leads.module.ts
        domain/
          entities/lead.entity.ts
          policies/lead-transition.policy.ts
        application/
          dto/
          use-cases/
            create-lead.use-case.ts
            convert-lead-to-customer.use-case.ts
        infrastructure/
          repositories/prisma-lead.repository.ts
        presentation/
          leads.controller.ts
      customers/
      deals/
      pipeline/
      quotations/
      sales-orders/
      invoices/
      payments/
      activities/
      reports/
      audit/
    prisma/
      schema.prisma
      migrations/
    test/
      integration/
      e2e/
  Dockerfile
  package.json
```

### Key Implementation Notes
- All financial transitions wrapped in DB transactions (`$transaction`).
- Domain policies enforce status transition matrices.
- Repositories abstract Prisma details from domain/application services.
- OpenAPI tags per module + workflow examples.

---

# 6. Frontend Structure (Next.js + shadcn/ui)

```text
frontend/
  src/
    app/
      (auth)/
        login/page.tsx
      (erp)/
        layout.tsx
        dashboard/page.tsx
        leads/
          page.tsx
          [id]/page.tsx
        deals/
          page.tsx
          [id]/page.tsx
        quotations/
          page.tsx
          [id]/page.tsx
        orders/
          page.tsx
          [id]/page.tsx
        invoices/
          page.tsx
          [id]/page.tsx
        payments/
          page.tsx
        reports/
          page.tsx
      api/health/route.ts
    components/
      ui/                # shadcn primitives
      layout/
      kpi/
      tables/
      forms/
        lead-form.tsx
        deal-stage-form.tsx
        invoice-post-form.tsx
    features/
      leads/
        api.ts
        hooks.ts
        schemas.ts
        components/
      deals/
      quotations/
      orders/
      invoices/
      payments/
      reports/
    lib/
      api-client.ts
      auth.ts
      permissions.ts
      query-client.ts
      formatters.ts
    styles/
      globals.css
```

### UX Patterns
- Left sidebar + top context action bar.
- Master-detail tables with sticky filters and saved views.
- Timeline widget shared across lead/customer/deal screens.
- Wizard-like action drawers for conversion steps (Lead→Customer, Quote→Order, Order→Invoice).

---

# 7. Key Business Workflows (Step-by-Step)

## 7.1 Lead to Customer Conversion
1. Sales creates lead (`NEW`).
2. Assigned rep performs call activity and updates status to `CONTACTED`.
3. Rep qualifies lead (`QUALIFIED`) with minimum data checks.
4. System executes `convert-to-customer` transaction:
   - create customer,
   - create primary contact,
   - update lead status `CONVERTED`, link customer.
5. Optional: create initial deal from converted lead.

## 7.2 Deal to Quotation to Order
1. Deal created and stage advanced through pipeline.
2. Quotation drafted from deal context with line items.
3. Quote sent then accepted by customer.
4. Conversion to sales order allowed only for accepted quote.
5. Sales order confirmed (commercial commitment).

## 7.3 Order to Invoice to Payment
1. Finance creates invoice from confirmed order (partial fulfillment respected).
2. Invoice posted (`POSTED`) — immutable financial snapshot.
3. Payment recorded and allocated (full or partial) to one/many invoices.
4. Invoice status recalculated:
   - `PARTIALLY_PAID` if residual > 0,
   - `PAID` if residual = 0,
   - `OVERDUE` if due date passed with residual > 0.

## 7.4 Activity-Driven Follow-up
1. Every stage change can auto-create next activity task.
2. Reminders shown on dashboard by due date and owner.
3. Completed activities stamped and retained for performance analytics.

## 7.5 Control & Compliance
1. Every key transition inserts `audit_logs` row.
2. Soft-deleted records excluded by default query scope.
3. Manager/Admin can access historical trails and restore records (except posted financial docs, which are reversed instead).

---

# 8. Scaling & Production Considerations

## 8.1 Performance
- Add read-optimized reporting views/materialized views for heavy aggregates.
- Use cursor pagination for large tables.
- Add Redis for cache/session/rate limiting.

## 8.2 Reliability
- Transactional outbox for integration events (ERP/accounting integrations).
- Idempotency keys for payment/invoice posting endpoints.
- Background workers (BullMQ) for reminders, overdue jobs, report snapshots.

## 8.3 Security
- JWT short-lived access + rotating refresh tokens.
- Field-level permission checks (credit limits, posting rights).
- PII protection, encrypted secrets, audit retention policies.

## 8.4 Observability
- Structured logs with correlation IDs.
- Metrics: conversion rates, posting failures, payment mismatch counts.
- Tracing for workflow latency (lead conversion, invoice posting).

## 8.5 Deployment & DevOps
- Dockerized backend/frontend + PostgreSQL service.
- Environment-specific config (`.env.development`, `.env.production`).
- CI: lint/test/migration check/OpenAPI diff.
- CD: blue-green/rolling deploy with zero-downtime migrations.

## 8.6 Data Governance
- Enforce timezone strategy (store UTC, display locale aware).
- Reference data management for taxes, terms, currencies.
- Backup strategy with PITR and periodic restore drills.
