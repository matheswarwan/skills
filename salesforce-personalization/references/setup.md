# Setup & Permissions — Salesforce Personalization

## Prerequisite Licenses

| License | Where to Check |
|---------|---------------|
| Salesforce Data Cloud (base) | Setup → Company Information → Licenses |
| Salesforce Personalization Add-On | Setup → Company Information → Licenses |

---

## Permission Sets

| Permission Set | Who Gets It |
|---------------|------------|
| `Personalization Admin` | Developers, implementation consultants — full access including sitemap editor, template builder, response templates, personalization points |
| `Personalization Business User` | Marketers using WPM — can build experiences, decisions, experiments; cannot edit sitemap or response templates |
| `Data Cloud Admin` | Required for Data Cloud config (data spaces, data graphs, CIs, segments, activations) |

Assign in Setup → Users → [User] → Permission Set Assignments.

---

## Datakit Deployment

The Personalization Datakit installs required DMOs, pre-built data graphs, and schema.

1. Data Cloud Setup → Datakit
2. Find **Salesforce Personalization Datakit** → Install
3. Confirm installation — this deploys:
   - Personalization-specific DMOs
   - Pre-built Profile Data Graph (Personalization template)
   - Pre-built Item Data Graphs for common object types (Goods Product, Knowledge Article)
   - Sample calculated insights

> ⚠️ Deploy the datakit before creating any response templates or personalization points. The datakit DMOs are required.

---

## Website Connector Setup

1. Data Cloud Setup → External Connections → Websites & Mobile Apps → New
2. Enter website name and domain
3. Copy the **SDK Beacon Script** — add to `<head>` of every page
4. Copy **Account Name** and **Dataset ID** — needed for sitemap initialization
5. The connector also has a **Sitemap** section — upload sitemap JS file here after it's built

---

## Mobile App Connector Setup

1. Data Cloud Setup → External Connections → Websites & Mobile Apps → New → Mobile App
2. iOS: download `SalesforcePersonalization.xcframework` or install via Swift Package Manager
3. Android: add Maven dependency `com.salesforce.mobilesdk:SalesforcePersonalization`
4. Copy app credentials for SDK initialization

---

## Data Space Planning

- Personalization defaults to the **default** data space
- Multi-brand or multi-region implementations may use separate data spaces
- All personalization points, response templates, and data graphs must be in the **same data space**
- The sitemap's `SalesforceInteractions.init({ dataspace: 'your_space' })` must match

---

## Full Setup Sequence Checklist

```
[ ] 1. Verify licenses (Data Cloud + Personalization Add-On)
[ ] 2. Assign permission sets to implementation team
[ ] 3. Deploy Personalization Datakit
[ ] 4. Create Website/Mobile App connector in Data Cloud Setup
[ ] 5. Install SDK beacon on website
[ ] 6. Build and deploy sitemap
[ ] 7. Validate event ingestion (Data Cloud → Data Explorer → check DLOs)
[ ] 8. Map DLOs to DMOs (Data Cloud → Data Streams → Field Mapping)
[ ] 9. Configure Identity Resolution rulesets
[ ] 10. Build Profile Data Graph
[ ] 11. Build Item Data Graph(s) for each item type
[ ] 12. Create Calculated Insights (for targeting and recommenders)
[ ] 13. Create Segments (for segment-based decision targeting)
[ ] 14. Create Recommenders (if using ML/rules-based recs)
[ ] 15. Create Response Templates
[ ] 16. Create Personalization Points
[ ] 17. Create Decisions (and/or Experiments)
[ ] 18. Use WPM to configure web experiences
[ ] 19. Validate personalization fetch in browser (Network tab → Decisioning API response)
[ ] 20. Configure engagement tracking and verify DMO writes
```