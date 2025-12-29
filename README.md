# hx-catalog

Product catalog service for HxLumin. Provides bundle definitions, categories, and visibility rules to all channels (USSD, mobile apps, bank integrations, agent apps).

## Overview

hx-catalog is the single source of truth for "what can this caller buy through this channel". All customer-facing channels call hx-catalog to get available products.

```
Bank App ──────┐
               │
Mobile Money ──┼──▶ hx-catalog ──▶ hx-decisions (price mods)
               │       │
USSD ──────────┤       ▼
               │   hx-orchestration (execute)
Agent App ─────┘
```

## What hx-catalog owns

- **Products** - name, base price, validity, resources
- **Categories** - product groupings
- **Visibility** - who can see what, on which channel, in which mode

## What hx-catalog does NOT own

- **Price modifications** - promos, discounts (hx-decisions)
- **Eligibility rules** - can this person buy now (hx-decisions)
- **Provisioning** - how to deliver the bundle (hx-orchestration)

## API

```
GET /products?category=data&caller={context}&channel=ussd

Response:
[
  { "id": "weekly500", "name": "Weekly 500MB", "price": 450, "validity": "7d" },
  { "id": "daily50", "name": "Daily 50MB", "price": 100, "validity": "24h" }
]
```

Products are filtered by visibility rules and prices adjusted by hx-decisions before returning.

## YAML Structure

See [docs/CATALOG_SPEC.md](docs/CATALOG_SPEC.md) for full specification.

### Products

```yaml
# catalogs/moov-togo/products/weekly500.yaml
id: weekly500
name:
  fr: "Forfait Semaine 500MB"
  en: "Weekly 500MB Bundle"
basePrice: 500
validity: 7d
resources:
  data: 500MB

visible:
  channels: [ussd, app, whatsapp]
  caller_type: [subscriber]
  service_class: [prepaid, hybrid]
  modes: [self, gift]
```

### Categories

```yaml
# catalogs/moov-togo/categories/data.yaml
id: data
name:
  fr: "Forfaits Data"
  en: "Data Bundles"
products:
  - daily50
  - weekly500
  - monthly1g
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ hx-catalog                                                  │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐  │
│  │ REST API    │───▶│ Catalog     │───▶│ hx-decisions    │  │
│  │             │    │ Engine      │    │ (price mods)    │  │
│  └─────────────┘    └──────┬──────┘    └─────────────────┘  │
│                            │                                │
│                     ┌──────▼──────┐                         │
│                     │ Redis Cache │                         │
│                     │ (products)  │                         │
│                     └─────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

## Related Components

- **hx-dialog** - USSD menu engine, consumes catalog API
- **hx-decisions** - Rules engine for pricing, eligibility, incentives
- **hx-orchestration** - Execution engine for payment and provisioning
