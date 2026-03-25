# Personalization Decisioning API Reference

## Overview

The Personalization Decisioning API provides real-time personalization decisions to any channel that can make HTTP requests — server-side apps, headless commerce, mobile backends, kiosks, Agentforce flows.

**Base URL:**
```
https://{personalization_domain}/api2/event/{accountName}/{datasetName}
```

Find your `personalization_domain`, `accountName`, and `datasetName` in:
Data Cloud Setup → External Connections → Websites & Mobile Apps → your connector

---

## Authentication

### Getting an API Token

1. In Data Cloud, create a Connected App with OAuth client credentials
2. Exchange for a Data Cloud token:

```bash
POST https://login.salesforce.com/services/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={connected_app_client_id}
&client_secret={connected_app_client_secret}
```

3. Use the returned `access_token` as a Bearer token in Personalization API calls

---

## Endpoints

### Unauthenticated (Public) Points
```
POST /api2/event/{accountName}/{datasetName}
Authorization: Bearer {token}
Content-Type: application/json
```

### Authenticated (Protected) Points — channel: Server
```
POST /api2/authevent/{accountName}/{datasetName}
Authorization: Bearer {token}
Content-Type: application/json
```
Use `/authevent` for personalization points with **Authentication** enabled.

---

## Request Body

```json
{
  "user": {
    "attributes": {
      "emailAddress": "customer@example.com",
      "userId": "CRM_12345"
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

| Field | Required | Description |
|-------|----------|-------------|
| `user.attributes` | Yes | Identity attributes for profile matching. At minimum one identifier (email, userId, deviceId) |
| `action` | Yes | Page type / action name — maps to sitemap page types or can be any string |
| `channel` | Yes for authevent | Set to `"Server"` for server-side calls via `/authevent` |
| `personalizationPoints` | Yes | Array of personalization point names to evaluate |

---

## Response

```json
{
  "personalizations": [
    {
      "personalizationId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "personalizationPointName": "home_hero",
      "personalizationPointId": "0jDXX000000xxxxx",
      "decisionId": "0jEXX000000xxxxx",
      "attributes": {
        "header": "Spring Collection Is Here",
        "subheader": "New arrivals just dropped",
        "ctaText": "Shop Now",
        "ctaUrl": "/collections/spring",
        "backgroundImageUrl": "https://cdn.example.com/spring-hero.jpg"
      },
      "data": []
    },
    {
      "personalizationId": "96c4a971-1234-5678-abcd-000000000001",
      "personalizationPointName": "home_recommendations",
      "personalizationPointId": "0jDXX000000yyyyy",
      "decisionId": "0jEXX000000yyyyy",
      "attributes": {
        "introText": "Recommended For You"
      },
      "data": [
        {
          "ssot__Id__c": "6010042",
          "ssot__Name__c": "GoBrew Connected Coffee Machine",
          "ImageUrl__c": "https://cdn.example.com/gobrew.jpg",
          "UnitPrice__c": "299.99",
          "PurchaseUrl__c": "/products/gobrew-6010042",
          "personalizationContentId": "96c4a971-1234-5678-abcd-000000000001:0"
        },
        {
          "ssot__Id__c": "6010089",
          "ssot__Name__c": "Alpine Pro Hiking Boots",
          "ImageUrl__c": "https://cdn.example.com/alpine-boots.jpg",
          "UnitPrice__c": "189.99",
          "PurchaseUrl__c": "/products/alpine-boots-6010089",
          "personalizationContentId": "96c4a971-1234-5678-abcd-000000000001:1"
        }
      ]
    }
  ]
}
```

### Key Response Fields

| Field | Description |
|-------|-------------|
| `personalizationId` | Unique decision ID — include in engagement events for attribution |
| `personalizationPointName` | Matches the requested point name |
| `attributes` | String fields from the response template — filled in by marketer on the decision |
| `data[]` | Array of recommended items (empty for Manual Content points) |
| `personalizationContentId` | Per-item ID in format `{personalizationId}:{index}` — send back on click/view events |

### Blank Response (No Decision Qualifies)

If the individual doesn't qualify for any decision, the point returns an empty personalization:
```json
{
  "personalizations": [
    {
      "personalizationPointName": "home_hero",
      "attributes": {},
      "data": []
    }
  ]
}
```
Always check that `attributes` is non-empty before rendering.

---

## Sending Engagement Events Back

After rendering personalized content, send engagement events to power attribution and ML training:

```bash
POST /api2/event/{accountName}/{datasetName}
Authorization: Bearer {token}
Content-Type: application/json

{
  "user": {
    "attributes": { "emailAddress": "customer@example.com" }
  },
  "action": "Clickthrough",
  "personalizationId": "96c4a971-1234-5678-abcd-000000000001",
  "personalizationContentId": "96c4a971-1234-5678-abcd-000000000001:0",
  "catalog": {
    "type": "Product",
    "id": "6010042"
  }
}
```

---

## Batch Personalization (Scheduled)

For email and offline channels — pre-compute recommendations for large audiences:

```bash
POST /api2/batch/event/{accountName}/{datasetName}
Authorization: Bearer {token}
Content-Type: application/json

{
  "users": [
    { "attributes": { "emailAddress": "user1@example.com" } },
    { "attributes": { "emailAddress": "user2@example.com" } }
  ],
  "action": "Email",
  "personalizationPoints": ["email_product_recs"]
}
```

---

## Agentforce Integration (Invocable Actions)

Two Invocable Actions are available in Flows for Agentforce:

### Get Recommendations
Returns item recommendations for an individual — inject into agent conversation context.

Flow action: `Personalization - Get Recommendations`
- Input: Individual ID, Personalization Point Name
- Output: Array of recommended items with attributes

### Get Context
Returns profile data for an individual — gives agent access to segment membership, CIs, preferences.

Flow action: `Personalization - Get Context`
- Input: Individual ID
- Output: Profile attributes, segments, CI values

---

## Error Codes

| HTTP Code | Meaning |
|-----------|---------|
| 200 | Success (check response body for blank personalizations) |
| 400 | Bad request — check request body format |
| 401 | Auth failure — check Bearer token |
| 403 | Forbidden — authenticated point called via unauthenticated endpoint |
| 404 | Account/dataset not found — check URL |
| 429 | Rate limit exceeded |
| 500 | Personalization server error — retry with backoff |