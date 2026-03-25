# Sitemap Reference — Salesforce Personalization

## Full Sitemap Template (SalesforceInteractions namespace)

```javascript
// ─── 1. Initialize SDK ──────────────────────────────────────────────
SalesforceInteractions.init({
  consents: [],            // Populate with user's consent object before sending events
  dataspace: 'default',   // Must match data space used in Personalization Points
});

// ─── 2. Engagement Destinations (custom, if needed) ─────────────────
// Run BEFORE setEvergageConfig. Register custom engagement destinations
// so WPM can reference them in the Engagement Tracking tab.
const engagementConfig = SalesforceInteractions.Personalization.Config.Engagement.get();
engagementConfig.destinations.push({
  label: 'Product Browse',
  description: 'Tracks product recommendation interactions',
  isRecommendations: true,
  engagementDmoApiName: 'Product_Engagement__dlm', // your mapped engagement DMO
});
// Repeat for Article Browse or other item types

// ─── 3. Personalization Experience Config (exported from WPM) ────────
// After business users publish experiences in WPM, export the config
// and paste it here. WPM generates this automatically on publish.
SalesforceInteractions.Personalization.Config.initialize({
  personalizationExperienceConfigs: [
    // { /* exported WPM config block */ }
  ]
});

// ─── 4. Site Map Config ──────────────────────────────────────────────
SalesforceInteractions.setEvergageConfig({
  siteMapConfig: {

    // ── GLOBAL: runs on every page ──────────────────────────────────
    global: {
      contentZones: [
        { name: 'global_popup', selector: 'body' }
      ],
      onActionEvent: (actionEvent) => {
        // Append known identity when user is logged in
        const email = document.querySelector('meta[name="user-email"]')?.content;
        if (email) {
          actionEvent.user = actionEvent.user || {};
          actionEvent.user.attributes = actionEvent.user.attributes || {};
          actionEvent.user.attributes.emailAddress = email;
        }
        return actionEvent;
      }
    },

    // ── PAGE TYPES ───────────────────────────────────────────────────
    pageTypes: [

      // Home Page
      {
        name: 'Homepage',
        action: 'Homepage',
        isMatch: () => location.pathname === '/' || location.pathname === '/home',
        contentZones: [
          { name: 'home_hero', selector: '.hero-banner' },
          { name: 'home_recommendations', selector: '.rec-row' }
        ],
        interaction: {},
        onActionEvent: (actionEvent) => actionEvent
      },

      // Product Detail Page
      {
        name: 'Product Detail Page',
        action: 'View Item',
        isMatch: () =>
          /\/products\//.test(location.pathname) ||
          /\/pdp\//.test(location.pathname),
        contentZones: [
          { name: 'pdp_recommendations', selector: '.related-products' }
        ],
        catalog: {
          type: SalesforceInteractions.CatalogObjectType.Product,
          id: SalesforceInteractions.resolvers.fromSelector('[data-product-id]', 'data-product-id'),
          name: SalesforceInteractions.resolvers.fromSelector('h1.product-title'),
          price: SalesforceInteractions.resolvers.fromSelector('.price-display'),
          imageUrl: SalesforceInteractions.resolvers.fromSelector('img.product-hero', 'src'),
          categories: SalesforceInteractions.resolvers.fromMeta('product:category'),
        },
        interaction: {
          name: SalesforceInteractions.CatalogObjectInteractionName.ViewItem
        },
        onActionEvent: (actionEvent) => actionEvent
      },

      // Category / PLP Page
      {
        name: 'Category Page',
        action: 'Category',
        isMatch: () => /\/category\/|\/collections\//.test(location.pathname),
        contentZones: [
          { name: 'category_banner', selector: '.category-header' }
        ],
        catalog: {
          type: SalesforceInteractions.CatalogObjectType.Category,
          id: SalesforceInteractions.resolvers.fromMeta('page:category-id'),
          name: SalesforceInteractions.resolvers.fromSelector('h1.category-title'),
        },
        interaction: {
          name: SalesforceInteractions.CatalogObjectInteractionName.ViewCategoryItem
        },
        onActionEvent: (actionEvent) => actionEvent
      },

      // Cart Page
      {
        name: 'Cart Page',
        action: 'Cart',
        isMatch: () => location.pathname.includes('/cart'),
        contentZones: [
          { name: 'cart_upsell', selector: '.cart-recommendations' }
        ],
        interaction: {},
        onActionEvent: (actionEvent) => {
          // Capture cart line items
          const cartItems = [];
          document.querySelectorAll('.cart-item').forEach((item) => {
            cartItems.push({
              id: item.dataset.productId,
              price: parseFloat(item.querySelector('.item-price')?.textContent),
              quantity: parseInt(item.querySelector('.item-qty')?.textContent),
            });
          });
          if (cartItems.length > 0) {
            actionEvent.order = { lineItems: cartItems };
          }
          return actionEvent;
        }
      },

      // Order Confirmation Page
      {
        name: 'Order Confirmation',
        action: 'Purchase',
        isMatch: () =>
          location.pathname.includes('/order-confirmation') ||
          location.pathname.includes('/checkout/success'),
        interaction: {},
        onActionEvent: (actionEvent) => {
          const orderId = document.querySelector('[data-order-id]')?.dataset.orderId;
          const lineItems = [];
          document.querySelectorAll('.order-item').forEach((item) => {
            lineItems.push({
              id: item.dataset.productId,
              price: parseFloat(item.dataset.price),
              quantity: parseInt(item.dataset.qty),
            });
          });
          actionEvent.order = { orderId, lineItems };
          return actionEvent;
        }
      },

      // Login Page
      {
        name: 'Login',
        action: 'Login',
        isMatch: () => location.pathname.includes('/login') || location.pathname.includes('/sign-in'),
        interaction: {},
        onActionEvent: (actionEvent) => actionEvent
      },

      // Default / fallback
    ],

    // ── DEFAULT (fallback for unmatched pages) ───────────────────────
    pageTypeDefault: {
      name: 'Default',
      interaction: {
        name: 'Default Page'
      }
    }
  }
});
```

---

## Identity Mapping Patterns

```javascript
// Pass email on login event
SalesforceInteractions.sendEvent({
  interaction: { name: 'Login' },
  user: {
    attributes: {
      emailAddress: 'user@example.com',   // maps to Individual email DLO
      userId: 'CRM_ID_12345',             // maps to Individual CRM ID
    }
  }
});
```

---

## Manually Sending Engagement Events (Custom Templates)

When using custom-built sitemap templates (not WPM), send engagement events manually:

```javascript
// On recommendation click
SalesforceInteractions.sendEvent({
  interaction: {
    name: 'Clickthrough',                       // engagement type
    catalog: {
      type: SalesforceInteractions.CatalogObjectType.Product,
      id: clickedProductId
    }
  },
  user: {},
  personalizationId: response.personalizationId,             // from decision response
  personalizationContentId: item.personalizationContentId    // from each item in data[]
});
```

---

## Content Zone Naming Conventions

| Zone Name | Location | Selector Example |
|-----------|----------|-----------------|
| `home_hero` | Homepage hero banner | `.hero-banner` |
| `home_recommendations` | Homepage rec row | `#home-rec-row` |
| `pdp_recommendations` | Product detail related products | `.related-products` |
| `category_banner` | Category page header | `.category-hero` |
| `cart_upsell` | Cart page upsell row | `.cart-rec-strip` |
| `global_popup` | Global popup overlay | `body` (no selector) |
| `nav_personalized` | Personalized nav element | `nav .personalized-slot` |

---

## SPA (Single-Page Application) Handling

For React, Next.js, Vue, etc.:

```javascript
// Re-fire page event on route change
// Call this from your router's onRouteChange callback
SalesforceInteractions.reinit();

// Or send a manual page event
SalesforceInteractions.sendEvent({
  interaction: { name: 'Homepage' }
});
```