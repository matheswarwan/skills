---
name: salesforce-personalization
description: >
  Use this skill for any task involving Salesforce Personalization (formerly Einstein Personalization) — the native Data Cloud personalization product. Covers: Web Personalization Manager (WPM) setup, personalization points, response templates, sitemap + Interactions SDK, Calculated Insights, Data Graphs (profile & item), Decisioning API (real-time & batch), mobile SDK, SFMC/Journey Builder activation, and scoping deliverables.

  Trigger whenever the user mentions: personalization points, response templates, WPM, Web Personalization Manager, Salesforce Personalization, Einstein Personalization, sitemap for personalization, Interactions SDK, recommenders, decisions, engagement destinations, Decisioning API, batch personalization, or Data Cloud + web/mobile experience delivery. Also trigger when scoping a personalization implementation.

  Do NOT confuse with Marketing Cloud Personalization (legacy Evergage/Interaction Studio) — this covers the modern Data Cloud-native product only.
---

# Salesforce Personalization Skill

Salesforce Personalization (SP) is the native Data Cloud personalization product. It replaced Einstein Personalization (name changed Spring '25). It uses real-time behavioral data from the Salesforce Interactions SDK, unified profiles via Data Cloud Identity Resolution, and pre-calculated Data Graphs to deliver personalized decisions across web, mobile, email, server-side, and agentic channels.

**Key concept:** SP is not a standalone product — it is Personalization-as-a-Service layered on top of Data Cloud. Every configuration object (data space, data graph, segment, calculated insight) is a Data Cloud object.

---

## Quick Reference: Implementation Order

```
1. Data Cloud Setup
   └── License & permission sets → Datakit deploy → Data spaces

2. Data Capture (Interactions SDK)
   └── Web SDK (sitemap) and/or Mobile SDK
   └── DLO-DMO mapping → Identity Resolution → Data Graphs

3. Decisioning Config (in Personalization App)
   └── Recommenders (if using ML recs)
   └── Response Templates
   └── Personalization Points
   └── Decisions / Experiments

4. Delivery
   └── Web: WPM + sitemap personalization experience config
   └── Mobile: Mobile SDK fetch
   └── Server-side / headless: Decisioning API
   └── Email: Marketing Cloud Growth/Advanced email builder
   └── Agentforce: Invocable actions
```

---

## 1. Platform Setup

See `references/setup.md` for full setup checklist.

**Key steps:**
- Assign **Salesforce Personalization** license in Setup → Users
- Assign permission sets: `Personalization Admin` (developers/admins), `Personalization Business User` (marketers using WPM)
- Deploy the **Personalization Datakit** via Data Cloud Setup → Datakit — this installs the required DMOs, data streams, and pre-built data graphs
- Confirm Data Space: Personalization uses the **default** data space unless a custom one is configured
- Configure **Websites & Mobile Apps connector** in Data Cloud Setup → External Connections → this is the entry point for web/mobile SDK integration

---

## 2. Data Capture — Salesforce Interactions SDK

The SDK is the behavioral data capture layer. It has two modules:

| Module | Use Case |
|--------|----------|
| `SalesforceInteractions` | Web + Personalization (recommended for new implementations) |
| `Evergage` | Legacy namespace — avoid for new work |

### SDK Installation (Web)

Add the beacon script to every page in `<head>`:

```html
<script type="text/javascript">
  var _sfmc = _sfmc || [];
  (function(e, t, a, n, s) {
    s = t.createElement(a);
    var r = t.getElementsByTagName(a)[0];
    s.async = 1;
    s.src = n;
    r.parentNode.insertBefore(s, r);
    _sfmc.push(['setOrgId', 'YOUR_ACCOUNT_NAME']);
  })(window, document, 'script',
    'https://cdn.evergage.com/beacon/YOUR_ACCOUNT_NAME/YOUR_DATASET_ID/scripts/evergage.min.js'
  );
</script>
```

Replace `YOUR_ACCOUNT_NAME` and `YOUR_DATASET_ID` with values from Data Cloud Setup → Websites & Mobile Apps.

### SDK Initialization in Sitemap

```javascript
SalesforceInteractions.init({
  consents: [],          // populate with user consent status
  dataspace: 'default',  // required if multiple data spaces
});
```

---

## 3. Sitemap Configuration

The **sitemap** is a JavaScript configuration file that maps your website's pages and user interactions to Personalization's data model. It must be built and deployed before any templates or WPM experiences can work.

See `references/sitemap.md` for full sitemap reference with code patterns.

### Sitemap Structure

```javascript
SalesforceInteractions.init({ consents: [] });

SalesforceInteractions.setEvergageConfig({
  siteMapConfig: {
    global: {
      // Fired on every page — identity, catalog, global content zones
      onActionEvent: (actionEvent) => {
        // Append identity attributes (email, CRM ID, etc.)
        return actionEvent;
      },
      contentZones: [
        { name: 'global_popup', selector: 'body' }
      ]
    },
    pageTypes: [
      {
        name: 'Homepage',
        action: 'Homepage',
        isMatch: () => location.pathname === '/',
        contentZones: [
          { name: 'home_hero', selector: '.hero-banner' },
          { name: 'home_recommendations', selector: '.rec-row' }
        ],
        catalog: {},
        interaction: {}
      },
      {
        name: 'Product Detail Page',
        action: 'View Item',
        isMatch: () => location.pathname.includes('/products/'),
        contentZones: [
          { name: 'pdp_recommendations', selector: '.related-products' }
        ],
        catalog: {
          type: SalesforceInteractions.CatalogObjectType.Product,
          id: SalesforceInteractions.resolvers.fromSelector('.product-id'),
          name: SalesforceInteractions.resolvers.fromSelector('h1.product-title'),
          price: SalesforceInteractions.resolvers.fromSelector('.price'),
          imageUrl: SalesforceInteractions.resolvers.fromSelector('img.product-img', 'src'),
        },
        interaction: {
          name: SalesforceInteractions.CatalogObjectInteractionName.ViewItem
        }
      }
    ]
  }
});
```

### Key Sitemap Concepts

**Content Zones** — Named hotspots on the page eligible for personalization rendering.
- Each has a `name` (machine-friendly, referenced by WPM and templates) and a `selector` (CSS selector pointing to the DOM element)
- Content zones without a selector are "logical zones" — used for overlays/popups

**Page Types** — Each distinct page layout gets its own `pageTypes` entry with an `isMatch` function that returns `true` when the current URL matches.

**Catalog** — Tracks items (products, articles, events) that a visitor interacts with. Required for recommendations to work.

**Interaction** — The user action occurring on the page (ViewItem, Purchase, AddToCart, etc.).

### Uploading the Sitemap

After editing: Data Cloud Setup → External Connections → Websites & Mobile Apps → click connector → Sitemap section → Replace Sitemap.

---

## 4. DLO-DMO Mapping & Identity Resolution

After the SDK sends events, Data Cloud ingests them into **Data Lake Objects (DLOs)**. You must map DLO fields to **Data Model Objects (DMOs)** for Personalization to use them.

**Critical mappings for Personalization:**
- `Individual` DMO — unified person
- `UnifiedIndividual` DMO — post-IR merged individual
- Product/item DMOs (e.g., `ssot__GoodsProduct__dlm`) — for recommendations
- Engagement DMOs (e.g., `Engagement__dlm`) — for behavioral tracking

**Identity Resolution (IR):**
- Configure IR rulesets in Data Cloud → Identity Resolution
- For web: device ID (first-party cookie from SDK) + email (passed on login) are the primary reconciliation identifiers
- Real-time IR matches anonymous sessions to known profiles sub-second

---

## 5. Data Graphs

Data Graphs are pre-computed views of related DMOs. Personalization requires two types:

| Type | Purpose |
|------|---------|
| **Profile Data Graph** | Combines UnifiedIndividual + engagement history + CIs + segment memberships. Used by the decisioning engine at decision time. |
| **Item Data Graph** | Organizes a catalog DMO (products, articles) with related attributes. Required for recommenders. |

**To create:** Data Cloud → Data Graphs → New.

- Profile DG: root on `UnifiedIndividualApplication` DMO, add related objects as needed
- Item DG: root on the item DMO (e.g., `GoodsProduct`), add attribute DMOs

> ⚠️ A response template for Recommendations can only surface fields that are **both** selected in the template **and** present on the item data graph. Plan both together.

---

## 6. Calculated Insights (CIs) for Personalization

CIs are multidimensional metrics computed over Data Cloud data. For Personalization they power:
- Rules-based recommenders (e.g., top sellers by category)
- Targeting rules on decisions (e.g., "show only if purchase_count > 3")
- Segment criteria

See `references/ci-patterns.md` for CI SQL templates relevant to personalization use cases.

**Common CI patterns for personalization:**
- Customer lifetime value tier
- Category affinity score (most viewed/purchased category per individual)
- Product engagement frequency (views/clicks in last 30 days)
- Recency score (days since last purchase)

---

## 7. Response Templates

Response templates define the **shape of the JSON payload** returned by the Decisioning API. They are created in the Personalization app → Response Templates.

### Manual Content Template
For static targeted content (banners, infobars, popups). Defines string fields that business users fill in on each decision.

**Setup:**
1. Personalization App → Response Templates → New
2. Type: **Manual Content**
3. Add personalization attributes (each becomes a text input on the decision):
   - `header`, `subheader`, `backgroundImageUrl`, `ctaText`, `ctaUrl`
4. Save

**Response payload shape:**
```json
{
  "personalizations": [{
    "personalizationPointName": "home_hero",
    "attributes": {
      "header": "Spring Collection Is Here",
      "ctaText": "Shop Now",
      "ctaUrl": "/collections/spring"
    },
    "data": []
  }]
}
```

### Recommendations Template
For ML-driven item recommendations. Defines attributes **plus** specifies which DMO fields to return.

**Setup:**
1. Personalization App → Response Templates → New
2. Type: **Recommendations**
3. Add personalization attributes (e.g., `introText`)
4. Go to **Item Attributes** tab → select DMO (e.g., `GoodsProduct`) → move fields to Selected:
   - `ssot__Id__c`, `ssot__Name__c`, `ImageUrl__c`, `UnitPrice__c`, `PurchaseUrl__c`
5. Save

**Response payload shape:**
```json
{
  "personalizations": [{
    "personalizationPointName": "home_recommendations",
    "attributes": { "introText": "Recommended For You" },
    "data": [
      {
        "ssot__Id__c": "6010042",
        "ssot__Name__c": "GoBrew Coffee Machine",
        "ImageUrl__c": "https://cdn.example.com/gobrew.jpg",
        "UnitPrice__c": "299.99",
        "personalizationContentId": "96c4a971-...:0"
      }
    ]
  }]
}
```

> ⚠️ Response templates are **not editable after creation**. Plan fields carefully. Create a naming convention (e.g., `hero_banner_v1`, `product_recs_v1`).

> ⚠️ `personalizationContentId` in each item is critical — it must be sent back on engagement events to enable attribution tracking.

---

## 8. Recommenders

Recommenders are ML or rules-based engines that select which items to return for a Recommendations personalization point.

| Type | Description | Powered By |
|------|-------------|------------|
| **Objective-Based** | ML-driven; optimizes toward a business goal (maximize revenue, clicks) | Data Cloud ML |
| **Rules-Based** | Deterministic; returns items filtered/sorted by a CI metric | Calculated Insights |

**Creating a Recommender:**
1. Personalization App → Recommenders → New
2. Select **Profile Data Graph** and **Item Data Graph**
3. For Objective-Based: select optimization goal; ML trains automatically
4. For Rules-Based: select a CI, choose filter/sort logic
5. The recommender's item DG must match the response template's DMO

---

## 9. Personalization Points

Personalization points are the named decision endpoints — one per "slot" on the experience (e.g., `home_hero`, `home_recommendations`, `pdp_recs`).

**Creating a Personalization Point:**
1. Personalization App → Personalization Points → New
2. Fill in:
   - **Name**: machine-friendly (e.g., `home_hero`, `pdp_recommendations`)
   - **Data Space**: must match sitemap's `SalesforceInteractions.init` dataspace
   - **Profile Data Graph**: the real-time profile DG to use for targeting
   - **Personalization Type**: Manual Content or Recommendations
   - **Response Template**: filtered to matching type + data space
   - **Authentication**: toggle on if this point should only be called via authenticated API
3. Save

**Critical rules:**
- Points with no decisions/experiments return a blank response
- All points in a single `fetch()` call must use the **same profile data graph**
- Each resolved decision consumes **1 decision credit** (even if blank response)

---

## 10. Decisions & Experiments

### Decisions
A decision defines who gets what content on a personalization point. One point can have multiple decisions with priority ordering.

**Decision config:**
- **Priority**: lower number = evaluated first
- **Targeting Rules**: built from profile DG attributes, CIs, segment memberships (e.g., `Loyalty Tier = Gold`)
- **Content**: the template attribute values or recommender selection
- **Schedule**: optional date range

### Experiments (A/B Testing)
Experiments replace a single decision with competing cohorts.

- Define cohorts (e.g., 50% see Recommender A, 50% see Recommender B)
- Set primary metric (e.g., product purchase rate) and secondary metrics
- Optional control group (receives no personalization)
- Results available in Personalization App → Analytics → Experiments

---

## 11. Web Personalization Manager (WPM)

The WPM is a no-code WYSIWYG tool for business users to apply personalization points to website pages without modifying the sitemap.

### Accessing the WPM
```
https://www.yourwebsite.com?sf_personalization_wpm
```
Requires **Personalization Admin** or **Personalization Business User** permission set.

### WPM Setup Flow

**Step 1 — Select Personalization Point**
- Click "+ New" → modal shows all points in the site's data space
- If no points appear: check sitemap `SalesforceInteractions.init` has correct `dataspace`

**Step 2 — Choose Rendering Method**

| Method | Use Case |
|--------|----------|
| Manually Personalize Page Elements | Simple text/image swaps; no developer needed |
| Use a Template Defined in Sitemap | Recommendations carousels, hero banners, popups; requires developer-built template |

**Step 3 — Configure WHEN (Page Option)**

| Option | Description |
|--------|-------------|
| Page Type | Fires on all pages matching a sitemap page type (preferred — resilient to URL changes) |
| URL | Fires on a specific URL pattern |

**Step 4 — Configure WHERE (Display Method)**

| Option | Description |
|--------|-------------|
| Content Zone | Replace content within a pre-defined sitemap content zone (most stable) |
| Element Selection | Use WPM's point-and-click selector to choose a DOM element; choose replace/insert before/after |
| Overlay | Pop-up triggered by scroll %, exit intent, or element click |

**Step 5 — Configure Engagement Tracking** (2nd tab)
- Select the engagement destination (Product Browse, Article Browse, or custom)
- Maps engagement events (views, clicks) to the correct DMO for attribution

**Step 6 — Preview**
- Test against current user or override with a specific Individual ID
- Use "Decision Selector" to preview any decision regardless of targeting rules

**Step 7 — Publish**
- Toggle state to **Live** → Save
- WPM automatically injects a Personalization Experience Config into the sitemap — no manual sitemap edit needed

### After WPM Publish: Sitemap Update

When an experience is published, the WPM exports a `personalizationExperienceConfigs` block that developers must initialize in the sitemap:

```javascript
// In the sitemap, after SalesforceInteractions.init():
SalesforceInteractions.Personalization.Config.initialize({
  personalizationExperienceConfigs: [
    // paste exported config from WPM here
  ]
});
```

---

## 12. Requesting Personalization via SDK (Web)

To fetch a decision from within custom sitemap/template code:

```javascript
// Single point fetch
SalesforceInteractions.Personalization.fetch(['home_hero'])
  .then((response) => {
    const personalization = response['home_hero'];
    if (personalization) {
      document.querySelector('.hero h1').textContent =
        personalization.attributes.header;
    }
  });

// Multi-point fetch (all must share same profile DG)
SalesforceInteractions.Personalization.fetch([
  'home_hero',
  'home_recommendations'
]).then((response) => {
  // render each point
});
```

---

## 13. Personalization API (Decisioning API)

For server-side, headless, mobile, or agentic channels.

See `references/api.md` for full API reference.

**Endpoint:**
```
POST https://{personalization_domain}/api2/event/{accountName}/{datasetName}
```

**Auth:** Connected App OAuth (client credentials) → exchange for Personalization API token.

**Request body:**
```json
{
  "user": {
    "attributes": {
      "emailAddress": "customer@example.com"
    }
  },
  "action": "Homepage",
  "channel": "Server",
  "personalizationPoints": [
    "home_hero",
    "home_recommendations"
  ]
}
```

**Authenticated endpoint (for points with Authentication enabled):**
```
POST https://{personalization_domain}/api2/authevent/{accountName}/{datasetName}
```

---

## 14. Mobile Implementation

See `references/mobile.md` for full iOS/Android SDK setup.

**High-level:**
1. Install Salesforce Interactions SDK (iOS: Swift Package Manager; Android: Maven)
2. Initialize with account name, dataset, data space
3. Send events on page views, item views, purchases
4. Call `SFMCPersonalization.fetch(points: ["home_recommendations"])` to get decisions
5. Render in native UI using returned JSON attributes

---

## 15. SFMC / Journey Builder Integration

Personalization integrates with SFMC in two ways:

### Segment Publish to SFMC (Batch)
1. In Data Cloud, create a Segment of the target audience
2. Activate the segment to SFMC via Segment Activation (Marketing Cloud connector)
3. In SFMC, use the activated segment as a Contact Filter in Journey Builder
4. Journey sends personalized email using AMPscript or dynamic content blocks that call Personalization response attributes

### Real-Time Journey Entry via Data Cloud-Triggered Journeys
1. Configure a Data Cloud-Triggered Journey in Journey Builder
2. Set entry event = Data Cloud Segment membership change or event
3. Use Personalization batch API to pre-compute recommendations and pass to Journey as personalization attributes

### Email Personalization (Marketing Cloud Growth/Advanced)
- Personalization points with type Manual Content or Recommendations are directly available in the email builder
- No SFMC connector needed — handled natively within Marketing Cloud Next

---

## 16. Reference Files

| File | Contents |
|------|----------|
| `references/setup.md` | Full setup checklist, permission sets, datakit deploy |
| `references/sitemap.md` | Sitemap patterns, page type examples, content zone config, engagement destinations |
| `references/ci-patterns.md` | CI SQL templates for personalization use cases (affinity, recency, LTV) |
| `references/api.md` | Decisioning API reference, auth flow, request/response examples |
| `references/mobile.md` | iOS/Android SDK setup and fetch patterns |
| `references/scoping-template.md` | Scoping document template for personalization implementation engagements |

---

## Common Pitfalls & Gotchas

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| No personalization points visible in WPM | Sitemap `dataspace` doesn't match point's data space | Check `SalesforceInteractions.init({ dataspace: '...' })` |
| Recommender not available on decision | Recommender's item DG DMO doesn't match response template's DMO | Align DG and response template to same DMO |
| Response template field not returned | Field not on item data graph | Add field to item DG, or select it in the response template's Item Attributes tab |
| Multi-point fetch skips second point | Points use different profile data graphs | Ensure all points in one fetch call share the same profile DG |
| Decision returns blank | No decisions or experiments configured on the point | Add at least one decision; confirm targeting rules aren't too restrictive |
| WPM not authenticating | Missing permission set | Assign `Personalization Admin` or `Personalization Business User` |
| Engagement data not in DMO | Engagement destination not configured or wrong DMO mapped | Configure engagement tracking tab in WPM; verify DLO-DMO mapping |