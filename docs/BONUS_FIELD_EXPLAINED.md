# Understanding the "Bonus" Field in Product Catalog

**To:** Faraz (PM)
**From:** Engineering
**Re:** How bonus data works in SmartShop and implications for hx-catalog

---

## Summary

The "bonus" in products like iZi'Kif (180MB + 120MB bonus) is **purely a marketing concept**. The OCS receives a single 300MB allocation - there is no separate "bonus bucket" in provisioning.

---

## How It Works Today (SmartShop)

### XML Config
```xml
<description>iZI'Kif (180Mo+120Mo)</description>
<usageThreshold id="134">
    <defaultValue>300000000</defaultValue>  <!-- 300MB total in bytes -->
</usageThreshold>
```

### Success Message (Hardcoded)
```xml
<language id="FRA">Souscription reussie au forfait "iZI'Kif" de "180Mo", plus bonus 120Mo valables 3 jours.</language>
```

### What Gets Sent to OCS
One UCIP4 call:
```
UpdateUsageThresholdsAndCounters
  - thresholdId: 134
  - thresholdValue: 300000000 (300MB)
```

The OCS just sees 300MB. The "180 + 120 bonus" split only exists in the customer-facing SMS.

---

## Implications for hx-catalog

The `bonus` field in our YAML is **for display/messaging only**:

```yaml
izikif:
  resources:
    data: 180MB
    bonus: 120MB   # <-- display only, not sent to OCS
```

hx-orchestration should provision: `data + bonus = 300MB`

hx-dialog can use both fields for templated messages:
```
"{{product.resources.data}} + bonus {{product.resources.bonus}}"
```

---

## Why Keep Them Separate?

1. **Marketing flexibility** - change bonus without editing message templates
2. **Display logic** - show breakdown on menus, confirmations, receipts
3. **Future proofing** - if we ever need separate buckets, schema is ready

---

## Dynamic Bonuses (Future)

SmartShop has a `benefitsMultiplier` mechanism for campaigns:
- Crediverse or RulesEngine can return a multiplier (e.g., 1.5)
- Provisioning applies it: `300MB * 1.5 = 450MB`

This is not used for Moov Togo's static products today, but hx-decisions (RuleForge) could implement similar logic.

---

## Bottom Line

- **Provisioning**: Always send total (data + bonus combined)
- **Display**: Keep separate for flexible messaging
- **No OCS changes needed**: Bonus is a presentation layer concept
