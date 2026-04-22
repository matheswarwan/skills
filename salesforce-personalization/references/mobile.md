# Mobile SDK — Salesforce Personalization

## Overview

The Salesforce Interactions SDK for mobile mirrors the web Interactions SDK — it sends behavioral events to Personalization and fetches real-time decisions for native iOS and Android apps.

**Connector setup prerequisite:** Data Cloud Setup → External Connections → Websites & Mobile Apps → New → Mobile App (see `setup.md` for full steps).

---

## iOS (Swift)

### Installation — Swift Package Manager

1. In Xcode: File → Add Packages
2. Enter the Salesforce Interactions SDK package URL (provided in the Mobile App connector in Data Cloud Setup)
3. Add `SalesforcePersonalization` to your app target

Or add to `Package.swift`:
```swift
dependencies: [
    .package(url: "https://github.com/salesforce-marketingcloud/sfmc-sdk-ios", from: "1.0.0")
]
```

> Get the exact URL and version from Data Cloud Setup → External Connections → Websites & Mobile Apps → your connector → SDK Instructions.

---

### Initialization

Initialize once in `AppDelegate.application(_:didFinishLaunchingWithOptions:)` or your app entry point:

```swift
import SalesforcePersonalization

SFMCPersonalization.shared.initialize(
    accountName: "YOUR_ACCOUNT_NAME",
    datasetId: "YOUR_DATASET_ID",
    dataspace: "default"          // match the data space of your personalization points
)
```

| Parameter | Where to find |
|-----------|--------------|
| `accountName` | Data Cloud Setup → Websites & Mobile Apps → connector → Account Name |
| `datasetId` | Data Cloud Setup → Websites & Mobile Apps → connector → Dataset ID |
| `dataspace` | The data space your personalization points live in (usually `default`) |

---

### Sending Events

Send events to power behavioral data ingestion and profile building.

**Page / Screen View:**
```swift
SFMCPersonalization.shared.track(
    action: "Home",
    attributes: [
        "category": "home"
    ]
)
```

**Item / Product View:**
```swift
SFMCPersonalization.shared.track(
    action: "View Item",
    attributes: [
        "itemType": "Product",
        "itemId": "6010042"
    ]
)
```

**Purchase:**
```swift
SFMCPersonalization.shared.track(
    action: "Purchase",
    attributes: [
        "orderId": "ORD-98765",
        "totalValue": "299.99"
    ],
    items: [
        ["itemType": "Product", "itemId": "6010042", "price": "299.99", "quantity": "1"]
    ]
)
```

---

### Fetching Decisions

Call `fetch` with one or more personalization point names. Always call after `track` so the current screen context is available.

```swift
SFMCPersonalization.shared.fetch(
    points: ["home_recommendations", "home_hero"]
) { result in
    switch result {
    case .success(let response):
        for personalization in response.personalizations {
            let pointName = personalization.personalizationPointName
            let attributes = personalization.attributes   // [String: String]
            let items = personalization.data              // [[String: String]]
            // render in native UI
        }
    case .failure(let error):
        // handle gracefully — show default/fallback content
        print("Personalization fetch failed: \(error)")
    }
}
```

**Multi-point constraint:** All points in a single `fetch` call must share the same Profile Data Graph. Split into separate calls if points use different profile DGs.

---

### Sending Engagement Events

Report impressions and clicks for attribution and ML training:

```swift
// Impression
SFMCPersonalization.shared.track(
    action: "Impression",
    attributes: [
        "personalizationId": personalization.personalizationId,
        "personalizationContentId": item["personalizationContentId"] ?? ""
    ]
)

// Click / Tap
SFMCPersonalization.shared.track(
    action: "Clickthrough",
    attributes: [
        "personalizationId": personalization.personalizationId,
        "personalizationContentId": item["personalizationContentId"] ?? ""
    ]
)
```

---

## Android (Kotlin / Java)

### Installation — Maven (Gradle)

Add to `build.gradle` (app module):

```groovy
dependencies {
    implementation 'com.salesforce.mobilesdk:SalesforcePersonalization:1.+'
}
```

Sync Gradle. The exact artifact coordinates and version are in your Data Cloud Setup → Websites & Mobile Apps → connector → SDK Instructions.

---

### Initialization

Initialize in `Application.onCreate()`:

```kotlin
import com.salesforce.mobilesdk.personalization.SFMCPersonalization

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        SFMCPersonalization.initialize(
            context = this,
            accountName = "YOUR_ACCOUNT_NAME",
            datasetId = "YOUR_DATASET_ID",
            dataspace = "default"
        )
    }
}
```

---

### Sending Events

```kotlin
// Screen view
SFMCPersonalization.shared().track(
    action = "Home",
    attributes = mapOf("category" to "home")
)

// Item view
SFMCPersonalization.shared().track(
    action = "View Item",
    attributes = mapOf(
        "itemType" to "Product",
        "itemId" to "6010042"
    )
)

// Purchase
SFMCPersonalization.shared().track(
    action = "Purchase",
    attributes = mapOf(
        "orderId" to "ORD-98765",
        "totalValue" to "299.99"
    ),
    items = listOf(
        mapOf("itemType" to "Product", "itemId" to "6010042", "price" to "299.99", "quantity" to "1")
    )
)
```

---

### Fetching Decisions

```kotlin
SFMCPersonalization.shared().fetch(
    points = listOf("home_recommendations", "home_hero"),
    callback = object : PersonalizationCallback {
        override fun onSuccess(response: PersonalizationResponse) {
            for (personalization in response.personalizations) {
                val pointName = personalization.personalizationPointName
                val attributes = personalization.attributes   // Map<String, String>
                val items = personalization.data              // List<Map<String, String>>
                // render in native UI
            }
        }

        override fun onFailure(error: PersonalizationError) {
            // show default/fallback content
        }
    }
)
```

---

### Sending Engagement Events

```kotlin
// Impression
SFMCPersonalization.shared().track(
    action = "Impression",
    attributes = mapOf(
        "personalizationId" to personalization.personalizationId,
        "personalizationContentId" to item["personalizationContentId"].orEmpty()
    )
)

// Click
SFMCPersonalization.shared().track(
    action = "Clickthrough",
    attributes = mapOf(
        "personalizationId" to personalization.personalizationId,
        "personalizationContentId" to item["personalizationContentId"].orEmpty()
    )
)
```

---

## Identity & Profile Matching

The mobile SDK matches the device to a unified Data Cloud profile using:

| Attribute | How to pass |
|-----------|------------|
| `emailAddress` | Pass in `track()` attributes when known (e.g., after login) |
| `userId` | Pass your CRM/SF Contact ID in `track()` attributes |
| Device ID | SDK sends automatically — used for anonymous profile until identity is resolved |

```swift
// iOS — associate identity after login
SFMCPersonalization.shared.track(
    action: "Login",
    attributes: [
        "emailAddress": "customer@example.com",
        "userId": "CRM_12345"
    ]
)
```

```kotlin
// Android — associate identity after login
SFMCPersonalization.shared().track(
    action = "Login",
    attributes = mapOf(
        "emailAddress" to "customer@example.com",
        "userId" to "CRM_12345"
    )
)
```

Identity Resolution in Data Cloud will merge the anonymous device profile with the known profile on the next run.

---

## Common Pitfalls

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Fetch returns empty `attributes` | No qualifying decision on the point | Check decision targeting rules in WPM; ensure audience isn't too restrictive |
| Events not appearing in Data Explorer | `accountName` or `datasetId` mismatch | Copy values directly from the connector in Data Cloud Setup |
| Multi-point fetch returns only one point | Points use different Profile Data Graphs | Ensure all fetched points share the same profile DG, or use separate fetch calls |
| Anonymous and known profiles not merging | Identity attribute not sent on `track` after login | Call `track` with `emailAddress`/`userId` on Login action |
| SDK not initializing on Android | `Application` subclass not registered in `AndroidManifest.xml` | Add `android:name=".MyApp"` to `<application>` tag |
