# hx-catalog Specification

Version: 1.0

## Overview

hx-catalog defines products and categories in YAML. The catalog service loads these definitions, applies visibility filtering based on caller context and channel, fetches price adjustments from hx-decisions, and returns the final product list.

## Directory Structure

```
catalogs/
└── {tenant}/
    ├── products/
    │   ├── weekly500.yaml
    │   ├── daily50.yaml
    │   └── bulk_10g.yaml
    └── categories/
        ├── data.yaml
        ├── voice.yaml
        └── agent_stock.yaml
```

## Product Schema

```yaml
id: string                    # Unique product identifier
name:                         # Display name (i18n)
  fr: string
  en: string
description:                  # Optional description (i18n)
  fr: string
  en: string
basePrice: number             # Base price before any adjustments
currency: string              # ISO 4217 code (default: XOF)
validity: string              # Duration: 24h, 7d, 30d
resources:                    # What the bundle provides
  data: string                # e.g., "500MB", "1GB", "unlimited"
  voice: string               # e.g., "100min", "unlimited"
  sms: string                 # e.g., "50", "unlimited"
tags: [string]                # For filtering/reporting

visible:                      # Visibility rules
  channels: [string]          # ussd, app, whatsapp, bank_app, agent_app
  caller_type: [string]       # subscriber, agent
  service_class: [string]     # prepaid, postpaid, hybrid
  segment: [string]           # mass, premium, vip, corporate
  agent_tier: [string]        # R, D, S (retail, distributor, super)
  modes: [string]             # self, gift, sale
```

### Complete Product Example

```yaml
# catalogs/moov-togo/products/weekly500.yaml

id: weekly500
name:
  fr: "Forfait Semaine 500MB"
  en: "Weekly 500MB Bundle"
description:
  fr: "500MB valable 7 jours"
  en: "500MB valid for 7 days"
basePrice: 500
currency: XOF
validity: 7d
resources:
  data: 500MB
tags: [data, weekly, popular]

visible:
  channels: [ussd, app, whatsapp]
  caller_type: [subscriber]
  service_class: [prepaid, hybrid]
  segment: [mass, premium, vip]
  modes: [self, gift]
```

### Agent-Only Product Example

```yaml
# catalogs/moov-togo/products/bulk_10g.yaml

id: bulk_10g
name:
  fr: "Stock Revendeur 10GB"
  en: "Reseller Stock 10GB"
basePrice: 15000
currency: XOF
validity: 30d
resources:
  data: 10GB
tags: [agent, bulk, data]

visible:
  channels: [ussd, agent_app]
  caller_type: [agent]
  agent_tier: [D, S]
  modes: [sale]
```

### Bank-Exclusive Product Example

```yaml
# catalogs/moov-togo/products/bank_promo.yaml

id: bank_promo
name:
  fr: "Promo Banque 1GB"
  en: "Bank Promo 1GB"
basePrice: 800
currency: XOF
validity: 7d
resources:
  data: 1GB
tags: [promo, bank, data]

visible:
  channels: [bank_app]
  caller_type: [subscriber]
  segment: [premium, vip]
  modes: [self]
```

## Category Schema

```yaml
id: string                    # Unique category identifier
name:                         # Display name (i18n)
  fr: string
  en: string
products: [string]            # List of product IDs
visible:                      # Optional category-level visibility
  channels: [string]
  caller_type: [string]
```

### Category Example

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
  - monthly5g
```

### Agent Category Example

```yaml
# catalogs/moov-togo/categories/agent_stock.yaml

id: agent_stock
name:
  fr: "Stock Agent"
  en: "Agent Stock"
products:
  - bulk_5g
  - bulk_10g
  - bulk_50g
visible:
  caller_type: [agent]
  channels: [ussd, agent_app]
```

## Visibility Rules

Visibility is evaluated as AND across all specified dimensions. If a dimension is not specified, it allows all values.

### Evaluation Logic

```
visible = true

for each dimension in [channels, caller_type, service_class, segment, agent_tier, modes]:
  if dimension is specified in product.visible:
    if caller.{dimension} not in product.visible.{dimension}:
      visible = false
      break

return visible
```

### Examples

**Product visible to prepaid subscribers on USSD:**
```yaml
visible:
  channels: [ussd]
  caller_type: [subscriber]
  service_class: [prepaid]
```

**Product visible to VIP subscribers on any channel:**
```yaml
visible:
  caller_type: [subscriber]
  segment: [vip]
```

**Product visible to distributor and super agents only:**
```yaml
visible:
  caller_type: [agent]
  agent_tier: [D, S]
```

## API Specification

### GET /categories

Returns categories visible to caller on channel.

**Request:**
```
GET /categories?channel=ussd
X-Caller-Context: {"msisdn":"22890123456","type":"subscriber","service_class":"prepaid","segment":"mass"}
```

**Response:**
```json
{
  "categories": [
    { "id": "data", "name": "Forfaits Data" },
    { "id": "voice", "name": "Forfaits Voix" },
    { "id": "combo", "name": "Forfaits Mixtes" }
  ]
}
```

### GET /products

Returns products in category visible to caller on channel, with adjusted prices.

**Request:**
```
GET /products?category=data&channel=ussd&mode=self
X-Caller-Context: {"msisdn":"22890123456","type":"subscriber","service_class":"prepaid","segment":"mass"}
```

**Response:**
```json
{
  "products": [
    {
      "id": "daily50",
      "name": "Daily 50MB",
      "price": 100,
      "basePrice": 100,
      "validity": "24h",
      "resources": { "data": "50MB" }
    },
    {
      "id": "weekly500",
      "name": "Weekly 500MB",
      "price": 450,
      "basePrice": 500,
      "validity": "7d",
      "resources": { "data": "500MB" },
      "promo": true
    }
  ]
}
```

Note: `price` may differ from `basePrice` due to hx-decisions adjustments.

### GET /products?ids=...

Returns specific products by ID (for curated lists).

**Request:**
```
GET /products?ids=weekly500,daily50&channel=ussd&mode=self
X-Caller-Context: {...}
```

### GET /product/{id}

Returns single product details.

**Request:**
```
GET /product/weekly500?channel=ussd&mode=self
X-Caller-Context: {...}
```

**Response:**
```json
{
  "id": "weekly500",
  "name": "Weekly 500MB",
  "description": "500MB valid for 7 days",
  "price": 450,
  "basePrice": 500,
  "validity": "7d",
  "resources": { "data": "500MB" },
  "promo": true
}
```

Returns 404 if product doesn't exist or is not visible to caller.

## Integration with hx-decisions

After filtering by visibility, hx-catalog calls hx-decisions to get price adjustments:

```
POST hx-decisions/evaluate
{
  "entityId": "hx-catalog",
  "contextId": "PRODUCT_PRICING",
  "properties": {
    "callerId": "22890123456",
    "segment": "mass",
    "channel": "ussd",
    "products": ["daily50", "weekly500"]
  }
}
```

Response includes price modifications:
```json
{
  "modifications": {
    "weekly500.price": { "operation": "DECREASE_BY_PERCENTAGE", "value": 10 }
  }
}
```

Catalog applies modifications and returns final prices.

## Caller Context

The caller context is provided by the channel (hx-dialog, mobile app, etc.) and includes:

| Field | Type | Description |
|-------|------|-------------|
| msisdn | string | Caller's phone number |
| type | string | `subscriber` or `agent` |
| service_class | string | `prepaid`, `postpaid`, `hybrid` |
| segment | string | `mass`, `premium`, `vip`, `corporate` |
| agent_tier | string | `R`, `D`, `S` (if agent) |

## Caching

Products and categories are cached in Redis for fast retrieval:

```
catalog:{tenant}:products          # Hash of all products
catalog:{tenant}:categories        # Hash of all categories
catalog:{tenant}:product:{id}      # Individual product
```

Cache is invalidated on YAML deployment.

## Deployment

1. YAML files are version controlled
2. On merge to main, CI validates YAML and pushes to Redis
3. Catalog service reads from Redis, falls back to YAML if needed
