# Calculated Insights SQL Patterns — Salesforce Personalization

All CIs for Personalization are authored in Data Cloud → Calculated Insights. They are available
as targeting dimensions on decisions and as inputs to rules-based recommenders.

---

## 1. Category Affinity Score (Most Viewed Category per Individual)

Identifies which product category each individual views most — drives category-based banner targeting.

```sql
SELECT
    ui.Id__c                                   AS individualId,
    pe.category__c                             AS preferredCategory,
    COUNT(pe.Id__c)                            AS categoryViewCount,
    RANK() OVER (
        PARTITION BY ui.Id__c
        ORDER BY COUNT(pe.Id__c) DESC
    )                                          AS categoryRank
FROM UnifiedIndividual__dlm ui
JOIN ProductEngagement__dlm pe
    ON pe.UnifiedIndividualId__c = ui.Id__c
WHERE pe.engagementType__c = 'ViewItem'
  AND pe.engagementDate__c >= DATEADD(day, -90, CURRENT_DATE)
GROUP BY ui.Id__c, pe.category__c
```

**Usage on decision targeting:**
`preferredCategory = 'Outdoor Gear'` → show Outdoor Gear hero banner

---

## 2. Recency Score (Days Since Last Purchase)

Segments customers by how recently they purchased. Powers re-engagement targeting.

```sql
SELECT
    ui.Id__c                                   AS individualId,
    MAX(o.orderDate__c)                        AS lastPurchaseDate,
    DATEDIFF(day, MAX(o.orderDate__c), CURRENT_DATE) AS daysSinceLastPurchase,
    CASE
        WHEN DATEDIFF(day, MAX(o.orderDate__c), CURRENT_DATE) <= 30  THEN 'Active'
        WHEN DATEDIFF(day, MAX(o.orderDate__c), CURRENT_DATE) <= 90  THEN 'AtRisk'
        WHEN DATEDIFF(day, MAX(o.orderDate__c), CURRENT_DATE) <= 180 THEN 'Lapsed'
        ELSE 'Churned'
    END                                        AS recencySegment
FROM UnifiedIndividual__dlm ui
LEFT JOIN Order__dlm o
    ON o.UnifiedIndividualId__c = ui.Id__c
GROUP BY ui.Id__c
```

**Usage on decision targeting:**
`recencySegment = 'AtRisk'` → show win-back offer banner

---

## 3. Purchase Frequency & Monetary (for LTV Tier)

Classic RFM — Frequency and Monetary components for LTV segmentation.

```sql
SELECT
    ui.Id__c                                   AS individualId,
    COUNT(DISTINCT o.orderId__c)               AS purchaseCount,
    SUM(o.orderTotal__c)                       AS lifetimeRevenue,
    AVG(o.orderTotal__c)                       AS avgOrderValue,
    CASE
        WHEN SUM(o.orderTotal__c) >= 5000      THEN 'Platinum'
        WHEN SUM(o.orderTotal__c) >= 1000      THEN 'Gold'
        WHEN SUM(o.orderTotal__c) >= 250       THEN 'Silver'
        ELSE 'Bronze'
    END                                        AS ltv_tier
FROM UnifiedIndividual__dlm ui
LEFT JOIN Order__dlm o
    ON o.UnifiedIndividualId__c = ui.Id__c
GROUP BY ui.Id__c
```

**Usage on decision targeting:**
`ltv_tier = 'Platinum'` → show VIP early access banner

---

## 4. Product Browse Affinity (Top Viewed Product per Individual)

Identifies the specific product each individual has viewed most — for PDP cross-sell targeting.

```sql
SELECT
    ui.Id__c                                   AS individualId,
    pe.productId__c                            AS topProductId,
    COUNT(pe.Id__c)                            AS viewCount
FROM UnifiedIndividual__dlm ui
JOIN ProductEngagement__dlm pe
    ON pe.UnifiedIndividualId__c = ui.Id__c
WHERE pe.engagementType__c = 'ViewItem'
  AND pe.engagementDate__c >= DATEADD(day, -30, CURRENT_DATE)
GROUP BY ui.Id__c, pe.productId__c
QUALIFY RANK() OVER (PARTITION BY ui.Id__c ORDER BY COUNT(pe.Id__c) DESC) = 1
```

---

## 5. Top Sellers by Category (for Rules-Based Recommender)

Item-level CI — identifies best-selling products per category over last 30 days.
Used as input to a rules-based recommender to return top sellers in affinity category.

```sql
SELECT
    p.productId__c                             AS productId,
    p.category__c                              AS category,
    SUM(ol.quantity__c)                        AS unitsSold,
    COUNT(DISTINCT o.orderId__c)               AS orderCount,
    SUM(ol.lineTotal__c)                       AS revenue
FROM GoodsProduct__dlm p
JOIN OrderLineItem__dlm ol ON ol.productId__c = p.productId__c
JOIN Order__dlm o          ON o.orderId__c = ol.orderId__c
WHERE o.orderDate__c >= DATEADD(day, -30, CURRENT_DATE)
GROUP BY p.productId__c, p.category__c
```

**Recommender config:** Sort by `unitsSold` DESC, filter by `category = individual's preferredCategory`

---

## 6. Email Engagement Recency (for Suppress-if-Recent-Email Targeting)

Prevents over-messaging — suppresses web popup if individual was emailed recently.

```sql
SELECT
    ui.Id__c                                       AS individualId,
    MAX(e.emailSentDate__c)                        AS lastEmailDate,
    DATEDIFF(day, MAX(e.emailSentDate__c), CURRENT_DATE) AS daysSinceLastEmail
FROM UnifiedIndividual__dlm ui
LEFT JOIN EmailEngagement__dlm e
    ON e.UnifiedIndividualId__c = ui.Id__c
  AND e.engagementType__c = 'Send'
GROUP BY ui.Id__c
```

**Usage on decision targeting:**
`daysSinceLastEmail > 7` → only show popup if not emailed in last 7 days

---

## 7. Anonymous Session Depth (for First-Visit vs. Returning Visitor)

Works on anonymous profiles. Counts sessions to distinguish new vs. returning visitors.

```sql
SELECT
    ui.Id__c                                   AS individualId,
    COUNT(DISTINCT s.sessionId__c)             AS sessionCount,
    CASE
        WHEN COUNT(DISTINCT s.sessionId__c) = 1 THEN 'New'
        WHEN COUNT(DISTINCT s.sessionId__c) <= 5 THEN 'Returning'
        ELSE 'Loyal'
    END                                        AS visitorType
FROM UnifiedIndividual__dlm ui
JOIN WebSession__dlm s
    ON s.UnifiedIndividualId__c = ui.Id__c
GROUP BY ui.Id__c
```

**Usage:** Show "Welcome! Here's what's popular" for new visitors; show personalized recs for Loyal

---

## Notes on CI Design for Personalization

- CIs must be added to the **Profile Data Graph** to be available as targeting dimensions on decisions
- Scalar CIs (single value per individual) are best for decision targeting rules
- Array/multi-row CIs (like top N products) are better suited as recommender inputs
- Real-time CIs update on each SDK event; batch CIs update on scheduled refresh — plan accordingly
- Name CIs descriptively: `persnl_CategoryAffinity_v1`, `persnl_LTVTier_v1` (use a prefix for easy filtering)