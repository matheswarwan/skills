# Salesforce Personalization — Implementation Scoping Template

**Client:** [Client Name]
**Prepared by:** [Consultant Name] | Futurik Cloud Solutions
**Date:** [Date]
**Version:** 1.0

---

## 1. Executive Summary

[2–3 sentence overview of what is being implemented and the primary business objective.]

---

## 2. Business Objectives

| # | Objective | Success Metric |
|---|-----------|---------------|
| 1 | [e.g., Increase product recommendation click-through rate] | [e.g., CTR > 3%] |
| 2 | | |
| 3 | | |

---

## 3. Use Cases In Scope

| # | Use Case | Channel | Personalization Type | Priority |
|---|----------|---------|---------------------|---------|
| 1 | Homepage hero banner — segment-targeted | Web | Manual Content | P1 |
| 2 | Homepage product recommendations | Web | Recommendations (ML) | P1 |
| 3 | PDP cross-sell recommendations | Web | Recommendations (ML) | P2 |
| 4 | Exit-intent popup — win-back offer | Web | Manual Content | P2 |
| 5 | Email product recommendations | Email (MC Growth) | Recommendations | P3 |
| 6 | Mobile home recommendations | iOS / Android | Recommendations (ML) | P3 |

---

## 4. Technical Architecture

### 4.1 Data Flow Diagram

```
Website / Mobile App
  └── Salesforce Interactions SDK
        └── Data Cloud (DLOs → DMO mapping)
              └── Identity Resolution
                    └── Profile Data Graph  ←── Calculated Insights + Segments
                          └── Personalization Decisioning Engine
                                └── Decision Response → SDK renders on site
```

### 4.2 Data Spaces

| Data Space | Purpose |
|-----------|---------|
| default | All personalization objects (points, templates, decisions) |

### 4.3 Identity Strategy

| Identifier | Source | Known/Anonymous |
|-----------|--------|----------------|
| Device ID (first-party cookie) | SDK auto-capture | Anonymous |
| Email Address | Login event / checkout | Known |
| CRM Customer ID | Login event | Known |

---

## 5. Data Capture Plan (Sitemap)

### 5.1 Page Types

| Page Type | `action` Name | Catalog Type | Key Events |
|-----------|--------------|-------------|------------|
| Homepage | `Homepage` | None | Page view |
| Product Detail Page | `View Item` | Product | ViewItem |
| Category Page | `Category` | Category | ViewCategoryItem |
| Cart | `Cart` | None | Page view |
| Order Confirmation | `Purchase` | None | Purchase + LineItems |
| Login | `Login` | None | Login + identity pass |

### 5.2 Content Zones

| Zone Name | Page Type | CSS Selector | Personalization Use Case |
|-----------|----------|-------------|------------------------|
| `home_hero` | Homepage | `.hero-banner` | Hero banner — segment targeted |
| `home_recommendations` | Homepage | `#rec-row` | ML product recommendations |
| `pdp_recommendations` | PDP | `.related-products` | Cross-sell recommendations |
| `global_popup` | All | `body` | Exit-intent popup (no selector) |

### 5.3 Catalog Objects Tracked

| DMO | Object Type | Key Fields |
|-----|------------|-----------|
| `GoodsProduct` | Product | ID, Name, Category, Price, ImageUrl |
| [Add others as needed] | | |

---

## 6. Data Modeling Plan

### 6.1 Data Graph

| Data Graph | Type | Root DMO | Related DMOs |
|-----------|------|---------|-------------|
| `Personalization_Profile_DG` | Profile | UnifiedIndividualApplication | Engagements, CIs, Segments |
| `Product_Item_DG` | Item | GoodsProduct | Categories, Images |

### 6.2 Calculated Insights Required

| CI Name | Purpose | Used For |
|---------|---------|---------|
| `persnl_CategoryAffinity_v1` | Top category per individual | Decision targeting, recommender filter |
| `persnl_LTVTier_v1` | Platinum/Gold/Silver/Bronze | Decision targeting |
| `persnl_RecencySegment_v1` | Active/AtRisk/Lapsed/Churned | Win-back decision targeting |
| `persnl_TopSellersByCategory_v1` | Top products by category | Rules-based recommender |

---

## 7. Personalization Configuration Plan

### 7.1 Response Templates

| Template Name | Type | Attributes |
|--------------|------|-----------|
| `hero_banner_v1` | Manual Content | header, subheader, backgroundImageUrl, ctaText, ctaUrl |
| `product_recs_v1` | Recommendations (GoodsProduct) | introText + DMO fields: Id, Name, ImageUrl, Price, Url |
| `popup_v1` | Manual Content | headline, bodyText, ctaText, ctaUrl |

### 7.2 Personalization Points

| Point Name | Type | Response Template | Profile DG | Auth |
|-----------|------|------------------|-----------|------|
| `home_hero` | Manual Content | `hero_banner_v1` | `Personalization_Profile_DG` | No |
| `home_recommendations` | Recommendations | `product_recs_v1` | `Personalization_Profile_DG` | No |
| `pdp_recommendations` | Recommendations | `product_recs_v1` | `Personalization_Profile_DG` | No |
| `global_popup` | Manual Content | `popup_v1` | `Personalization_Profile_DG` | No |

### 7.3 Decisions (Initial)

| Decision | Point | Targeting Rule | Content |
|----------|-------|---------------|---------|
| VIP Hero | `home_hero` | LTVTier = Platinum | VIP exclusive offer banner |
| Outdoor Affinity Hero | `home_hero` | CategoryAffinity = Outdoor | Outdoor collection hero |
| Default Hero | `home_hero` | None (fallback) | Generic seasonal banner |
| ML Recs — Active | `home_recommendations` | RecencySegment = Active | Objective-based ML recommender |
| ML Recs — AtRisk | `home_recommendations` | RecencySegment = AtRisk | Rules-based: top sellers in affinity category |

---

## 8. Phased Delivery Plan

### Phase 1 — Foundation (Weeks 1–4)
- [ ] SDK beacon deployed
- [ ] Sitemap built and validated (Homepage, PDP, Cart, Purchase pages)
- [ ] DLO → DMO mapping complete
- [ ] Identity Resolution configured and validated
- [ ] Profile and Item Data Graphs built

### Phase 2 — Web Personalization (Weeks 5–8)
- [ ] CIs created and validated
- [ ] Response templates created
- [ ] Personalization points configured
- [ ] Initial decisions configured (hero banner, homepage recs)
- [ ] WPM experiences published
- [ ] Engagement tracking validated (view/click DMO writes)

### Phase 3 — ML & Experiments (Weeks 9–12)
- [ ] Recommenders configured (rules-based + objective-based)
- [ ] PDP cross-sell point and decisions live
- [ ] A/B experiment configured on homepage recommendations
- [ ] Experiment metrics baseline established

### Phase 4 — Mobile & Email (Weeks 13–16)
- [ ] Mobile SDK integrated (iOS and/or Android)
- [ ] Mobile personalization points and decisions
- [ ] Email recommendations via MC Growth/Advanced or batch API
- [ ] Attribution reporting review

---

## 9. Out of Scope

- [List anything explicitly excluded]
- Legacy Interaction Studio / Evergage configuration
- Marketing Cloud Engagement template development (separate workstream)

---

## 10. Assumptions & Dependencies

| # | Assumption / Dependency |
|---|------------------------|
| 1 | Client has active Salesforce Data Cloud license with Personalization add-on |
| 2 | Client website team can deploy the SDK beacon script within 1 week of engagement start |
| 3 | Product catalog data (IDs, names, prices, image URLs) is available via website DOM or API |
| 4 | Client CRM sends customer identity (email) to website on login |
| 5 | Data Cloud base configuration (data spaces, connected sources) is already in place |

---

## 11. Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| SDK beacon deployment delayed | Medium | High | Engage client DevOps early; use GTM as fallback |
| Identity resolution merge rate low | Medium | High | Validate email pass on login before Phase 2 |
| Response template fields change after creation | Low | Medium | Plan template fields with client before creation (immutable) |
| Decision credit consumption over projection | Low | Medium | Monitor via Digital Wallet; cap per-session fetches |

---

## 12. Open Questions

| # | Question | Owner | Due |
|---|----------|-------|-----|
| 1 | Which product catalog fields are available in DOM vs. require API enrichment? | Client Dev | [Date] |
| 2 | Are there multiple data spaces in use, or just default? | Client Arch | [Date] |
| 3 | Is Marketing Cloud Growth or Advanced Edition licensed for email personalization? | Client | [Date] |