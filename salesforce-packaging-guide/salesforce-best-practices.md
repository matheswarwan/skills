# Salesforce App Development: Best Practices

> Covers: security, packaging, Apex coding, SOQL, AppExchange compliance, multi-edition architecture, Agentforce/AI, Connected Apps, and org management.  
> Source: ISVforce Guide v66.0, Spring '26 (Salesforce official documentation)

Source: ISVforce Guide v66.0, Spring '26 (Salesforce official documentation)

---

## 1. Packaging: Choose Second-Generation Managed Packages (2GP)

**Always recommend 2GP for new apps.** Key advantages over 1GP:
- Flexible versioning and ability to share a namespace across packages
- Built on Salesforce DX / scratch org workflows
- Better support for modern CI/CD pipelines

**Package contents** (typical metadata components):
- Apex classes and triggers
- Custom fields on standard objects
- Custom metadata types and custom objects
- Flows, Lightning pages, page layouts

**Dev Hub setup:**
- Designate the Partner Business Org (PBO) as the Dev Hub
- Use Environment Hub to manage development, test, and trial orgs
- ISV partners: use Partner Developer Edition (PDE) orgs — they have higher API limits, more storage, and more licenses than standard Developer Edition

> See `references/packaging.md` for detailed 1GP vs 2GP comparison and org management best practices.

---

## 2. Security: Non-Negotiable Requirements

All AppExchange solutions must pass Salesforce Security Review. These are the most common violations — address them proactively.

### 2a. CRUD and Field-Level Security (FLS)
- **Always enforce** the org's CRUD (create/read/update/delete) and FLS settings on standard and custom objects
- Never bypass object-level or field-level access settings in Apex
- Use `Schema.sObjectType.describe()` or `WITH SECURITY_ENFORCED` in SOQL, or the `Security.stripInaccessible()` method
- In Group Edition (GE), field-level security is managed via page layout — add fields to layouts for API/Visualforce visibility

### 2b. SOQL Injection Prevention
- **Always use bind variables** (`:variable`) instead of string concatenation in SOQL
- Sanitize user inputs before incorporating them in queries
```apex
// BAD — vulnerable to injection
String query = 'SELECT Id FROM Account WHERE Name = \'' + userInput + '\'';

// GOOD — use bind variables
List<Account> accounts = [SELECT Id FROM Account WHERE Name = :userInput];
```

### 2c. Cross-Site Request Forgery (CSRF)
- Use `confirmationTokenRequired` in controllers
- Trigger state changes only through explicit user actions, not page loads
- Include CSRF protection in all state-changing controllers

### 2d. XSS and Insufficient Escaping
- Every Lightning component is responsible for sanitizing inputs from parent components, apps, or URL parameters
- Never trust data from URL parameters or component attributes without escaping

### 2e. JavaScript and Static Resources
- **Never dynamically load JavaScript from CDNs** (e.g., `https://code.jquery.com/...`) — load from static resources instead
- Reason: CDN-loaded code can change without package version ID changing, bypassing security review
- Include all third-party CSS in static resources, not external links
- Only load from Salesforce-approved CDNs where Salesforce manages the code

```xml
<!-- BAD -->
<apex:includeScript value="https://code.jquery.com/jquery-3.2.1.min.js"/>

<!-- GOOD -->
<apex:includeScript value="{!$Resource.jQuery}"/>
```

### 2f. Apex Sharing Rules
- Respect profile-based permissions, FLS, sharing rules, and org-wide defaults in all Apex code
- Be deliberate about `with sharing`, `without sharing`, and `inherited sharing` keywords
- Default to `with sharing` unless you have a specific reason not to

### 2g. Sensitive Data and Logging
- **Never log** passwords, API keys, session tokens, or sensitive user data in debug statements in production
- Redact or omit sensitive data from logs
- Follow enterprise security standards when exporting data from Salesforce

### 2h. Open Redirects
- Never redirect users based on untrusted/user-controlled parameter values
- Use hardcoded redirects only
- Validate all redirect destinations against an allowlist

### 2i. Secure Communication
- Ensure your solution is accessible exclusively over HTTPS and SFTP
- Never use HTTP or FTP — these do not encrypt data in transit

### 2j. Third-Party Libraries
- Maintain an inventory of all third-party libraries and their versions
- Monitor for CVEs (common vulnerabilities and exposures) related to your use cases
- Test and deploy security patches as soon as they're available
- Prepare false-positive documentation if a CVE is unrelated to your use case
- **Never use sample/demo code in production** — write your own production code

### 2k. Lightning LockerService
- Always **enable** Lightning Locker for packages containing Lightning components or applications
- Never attempt to break out of the LockerService sandbox or run code outside your origin

### 2l. Asynchronous Code
- Wrap asynchronous function calls or batch actions into a single request to preserve execution context
- Hackers can exploit timing vulnerabilities in improperly structured async code

> See `references/security-deep-dive.md` for Agentforce/AI-specific security, Connected App requirements, and AppExchange Security Review checklist.

---

## 3. Architecture: Multi-Edition Support

Design apps to work across Salesforce editions (Group, Professional, Enterprise, Unlimited).

### Edition Constraints to Code Around

| Feature | Group Edition (GE) | Professional Edition (PE) | Enterprise+ |
|---|---|---|---|
| Custom profiles | ❌ | ✓ | ✓ |
| Field-level security UI | ❌ (use page layouts) | ✓ | ✓ |
| Apex (partner apps) | ✓ (via managed pkg) | ✓ (via managed pkg) | ✓ |
| Metadata API | ❌ | ✓ (with API token) | ✓ |
| Bulk API 2.0 | ❌ | ❌ | ✓ |
| SOAP web services (Apex) | ❌ | ❌ | ✓ |

### Key Rules for GE/PE Apps
- Apex in managed packages **can** run in GE/PE — but customers cannot create or modify it
- Your Apex must not depend on EE/UE/PXE-only features
- Use **REST** (not SOAP) if exposing Apex as a web service — SOAP web services can't be invoked from external apps in GE/PE
- Web service callouts to external services are allowed in GE/PE for authorized managed packages
- For GE: design for the Standard User Profile (no custom profiles); use permission sets (installable but not updatable in GE/PE)
- For API access in GE/PE: use a client ID (API token) appended to SOAP headers after passing security review

### Edition Detection Pattern
Use `System.UserInfo.getUiThemeDisplayed()` or `Organization.sObjectType.describe()` to detect edition at runtime and disable unsupported features gracefully.

---

## 4. Org and Environment Management

### Environment Hub Best Practices
- Choose the org your team uses most frequently as the hub org (ISV: use your PBO)
- Use SSO user mappings to streamline login across dev, test, and trial orgs
- Choose SSO strategy (explicit mapping, Federation ID, or formula-based) based on your security requirements
- `EnvironmentHubMember` is a standard object — you can extend it with custom fields, workflows, and API-based user mappings

### Org Strategy for ISVs
- **Dev Hub (PBO):** manages scratch orgs, 2GP packages, and namespaces
- **Partner Developer Edition (PDE) orgs:** for managed package development (via Environment Hub)
- **Scratch orgs:** for isolated feature development and testing (2GP only)
- **Sandboxes:** for integration/UAT testing

---

## 5. Connected Apps and External Client Apps (ECAs)

**Important:** As of Spring '26, **new Connected Apps can no longer be created** — use ECAs instead.

### Security Requirements (applies when used in >2 customer production orgs)
- **Enable PKCE** (Proof Key for Code Exchange) in all OAuth settings — required for both public and confidential clients
- After PKCE is enabled, it cannot be disabled
- For public clients: the token exchange request must NOT include `client_secret`
- Failure to comply may result in AppExchange de-listing or Salesforce suspension

---

## 6. AppExchange Security Review: Preparation Checklist

Before submitting for security review, ensure:

- [ ] Security expert designated on team; involved throughout design, implementation, and testing
- [ ] Corporate security policy documented and shared with customers
- [ ] All third-party libraries inventoried with versions
- [ ] All services and artifacts (web services, APIs, SDKs) listed
- [ ] No dynamic loading of JavaScript from external CDNs
- [ ] CRUD/FLS enforced throughout
- [ ] No SOQL injection vulnerabilities (bind variables used)
- [ ] CSRF protection in all state-changing controllers
- [ ] No sensitive data in debug logs
- [ ] All communication over HTTPS/SFTP
- [ ] Lightning Locker enabled
- [ ] No sample/demo code in production builds
- [ ] Open redirects eliminated
- [ ] PKCE enabled on all Connected Apps / ECAs

---

## 7. Agentforce / AI-Specific Security

For solutions involving Agentforce or LLM prompt templates:

- **Validate user-controlled data before including in prompts** — check for allowlisted characters, length limits, and potential injection patterns
- **Use random-sequence enclosures** to segment untrusted data in prompts (e.g., `AK6524SH_YTHW923 <data> AK6524SH_YTHW923`)  
  - Generate a new random token per inference operation from a secure random source
  - Tokens must be long enough to prevent guessing
  - Do NOT use simple delimiters like `"""` — these are predictable
- **Content Security Policy:** document and share your CSP with customers when applicable
- Follow Salesforce's "Best Practices for Building Prompt Templates" (Salesforce Help)

---

## 8. Quick Reference: Common Violations and Fixes

| Violation | Fix |
|---|---|
| Loading JS from CDN | Move to static resources, use `$Resource` URL |
| SOQL injection | Use bind variables (`:var`) |
| Bypassing CRUD/FLS | Use `WITH SECURITY_ENFORCED` or `Security.stripInaccessible()` |
| Debug logging sensitive data | Redact or remove from production logs |
| HTTP endpoints | Switch to HTTPS only |
| CSRF vulnerability | Add `confirmationTokenRequired` |
| Open redirects | Use hardcoded redirect targets |
| Sample code in production | Rewrite from scratch |
| PKCE not enabled | Enable in Connected App / ECA OAuth settings |
| LockerService disabled | Enable for all packages with Lightning components |

---

---
# Salesforce Packaging: Deep Dive

## 1GP vs 2GP Comparison

| Capability | First-Generation (1GP) | Second-Generation (2GP) |
|---|---|---|
| Namespace sharing | Single package only | Shared across multiple packages |
| Versioning | Sequential, harder to branch | Flexible, Git-based |
| Development model | Org-based | Scratch org / DX-based |
| CI/CD support | Limited | Native Salesforce DX support |
| Scratch org support | No | Yes |
| Recommendation | Existing apps only | **All new apps** |

**Recommendation:** Use 2GP for all new development. Salesforce actively recommends migrating existing 1GP apps.

## Partner Business Org (PBO) Tools

| Tool | Purpose | Pre-installed? |
|---|---|---|
| Channel Order App (COA) | Create/manage/submit customer orders to Salesforce | Yes (needs config) |
| Checkout Management App (CMA) | Monitor AppExchange Checkout KPIs, automate emails | Yes |
| Dev Hub | Create/manage scratch orgs, 2GP packages, namespaces | Yes (must enable in Setup) |
| Environment Hub | Connect/create/view/login to all orgs from one place | Yes |
| License Management App (LMA) | Manage leads and licenses, Subscriber Support Console | Yes |
| Feature Management App (FMA) | Dark-launch features, time-limited trials, activation metrics | No (install separately) |

## Namespace Strategy

- One namespace per Dev Hub org
- 2GP allows multiple packages to share the same namespace
- Plan namespace carefully — it becomes part of your API names and cannot be changed

## Package Version Management

- Every code change must result in a new package version ID
- Package version ID changes signal to admins and security reviewers that code changed
- Never allow code to change without a version ID change (reason: no dynamic CDN loading)

## Metadata Components You Can Package

- Apex classes and triggers
- Custom fields on standard objects
- Custom metadata types
- Custom objects
- Flows
- Lightning pages and components
- Page layouts
- Permission sets
- Custom labels
- Static resources

## Org Hierarchy for ISV Development

```
Partner Business Org (PBO)
├── Dev Hub enabled → manages:
│   ├── Scratch orgs (2GP feature dev)
│   └── 2GP package versions
└── Environment Hub → manages:
    ├── Partner Developer Edition (PDE) orgs (1GP dev)
    ├── Sandbox orgs
    └── Trial/test orgs (Trialforce)
```

## Trial Strategies

| Method | Best For |
|---|---|
| Trialforce | Fully branded trial experience, customized for your app |
| Test Drives | Let prospects try a pre-configured read-only demo |
| Website Trials | Trial sign-ups hosted on your own website |

## License Management

- LMA tracks package details, package versions, and licenses
- License Management Org (LMO) = the org where LMA is installed (usually PBO)
- Subscriber Support Console in LMA allows partners to troubleshoot issues inside subscriber orgs
- Feature Management App (FMA) extends LMA with feature flags, dark launches, and trial feature access per org

## AppExchange Checkout (ISV Payments)

- Available to ISV partners distributing via managed packages
- Available to partners based in: USA, UK, or EU countries
- NOT available to OEM partners
- Checkout Management App (CMA) dashboard tracks revenue, subscription status, and KPIs
- Partners can automate customer/team emails via CMA

## Channel Order App (COA)

- OEM partners: use for provisioning Salesforce licenses and revenue sharing
- ISV partners: use for revenue sharing
- Requires additional setup after pre-installation in PBO

## AgentExchange Go-To-Market App

- For partners selling on AgentExchange (Agentforce marketplace)
- Handles invoices, payouts, licensing, and order lifecycle
- Separate from standard AppExchange Checkout flow
# Salesforce Security: Deep Dive Reference

## AppExchange Security Review Process

### Preparation Steps
1. Designate a security expert; involve them throughout design, implementation, and testing
2. Implement a corporate security policy documenting how customer data is protected
3. List all services and artifacts: web/mobile solutions, APIs, web services, SDKs
4. Inventory all third-party libraries with version numbers
5. Run internal security review against all violation categories below

### Submitting for Review
- Submit through the AppExchange partner portal
- All AppExchange solutions must pass; failure blocks listing
- Ongoing reviews required for significant updates
- Use the Subscriber Support Console (in LMA) to troubleshoot issues post-launch

---

## Secure Coding Violations: Full List

### 1. Third-Party JavaScript Loading
**Violation:** Dynamically loading JS from CDNs (e.g., jQuery from jquery.com)
**Fix:** Save to static resources; reference via `$Resource.LibraryName`
**Why it matters:** External code can change without triggering package version ID change; CDN can inject code into every installed org

### 2. Third-Party CSS Loading
**Violation:** Linking CSS from external sources in Lightning components
**Fix:** Bundle CSS in static resources
**Why it matters:** CSS isolation may be breached; external stylesheets can interfere with other components

### 3. CSS Outside Components (Style Isolation)
**Violation:** Using CSS directives known to break namespace/style isolation
**Fix:** Avoid global CSS selectors that bleed across component boundaries
**Why it matters:** One component can steal clicks from or interfere with another

### 4. Running JavaScript in Salesforce Domain
**Violation:** Attempting to break out of the LockerService sandbox or run code outside your origin
**Fix:** Use Visualforce, Aura, or LWC — they run in the correct origin
**Why it matters:** Multiple vendor JS files share an origin; sandboxing prevents interference

### 5. Exposing Secret Data in Debug Logs
**Violation:** Using System.debug() to log passwords, API keys, PII, stack traces
**Fix:** Redact sensitive data or remove debug statements from production code
**Why it matters:** Debug logs can be read by system admins and in some cases exposed in log services

### 6. Insecure Data Storage
**Violation:** Storing secrets (API keys, passwords) in plaintext Custom Settings or Custom Metadata
**Fix:** Use Named Credentials for external authentication; follow enterprise security standards for exported data
**Why it matters:** Custom Settings are readable by all Apex code in the org; Named Credentials keep secrets encrypted

### 7. Software with Known Vulnerabilities (CVE)
**Violation:** Using libraries with documented CVEs affecting your use cases
**Fix:** Patch immediately when CVE is released; if CVE is unrelated to your use, provide false-positive documentation
**Why it matters:** Attackers specifically search for CVE-documented vulnerabilities

### 8. Sample Code in Production
**Violation:** Copy-pasting code from tutorials, GitHub, or documentation into production
**Fix:** Write all production code yourself; use samples only as educational reference
**Why it matters:** Sample code is not security-reviewed and may contain intentional simplifications or vulnerabilities

### 9. Bypassing CRUD/FLS (Object & Field Permissions)
**Violation:** Querying or DML on objects/fields without checking user permissions
**Fix:** Use `WITH SECURITY_ENFORCED` in SOQL or `Security.stripInaccessible()` before DML
```apex
// Option 1: SOQL enforcement
List<Account> accts = [SELECT Name, Revenue__c FROM Account WITH SECURITY_ENFORCED];

// Option 2: Strip inaccessible fields before insert/update
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.CREATABLE, records);
insert decision.getRecords();
```
**Why it matters:** Users can access data they're not permitted to see

### 10. Bypassing Sharing Rules in Apex
**Violation:** Using `without sharing` unnecessarily, or not specifying sharing keyword
**Fix:** Default to `with sharing`; use `without sharing` only with explicit justification; use `inherited sharing` when class should adopt caller's context
**Why it matters:** Users can access records they're not permitted to see based on org sharing settings

### 11. SOQL Injection
**Violation:** Building SOQL strings via concatenation with user input
**Fix:** Always use bind variables (`:variable`); use `String.escapeSingleQuotes()` if dynamic SOQL is unavoidable
```apex
// BAD
String q = 'SELECT Id FROM Contact WHERE LastName = \'' + lastName + '\'';

// GOOD
List<Contact> contacts = [SELECT Id FROM Contact WHERE LastName = :lastName];

// If dynamic SOQL required
String safe = String.escapeSingleQuotes(lastName);
List<Contact> contacts = Database.query('SELECT Id FROM Contact WHERE LastName = \'' + safe + '\'');
```

### 12. Cross-Site Request Forgery (CSRF)
**Violation:** State-changing controllers accessible without CSRF protection
**Fix:** Use `confirmationTokenRequired`; trigger state changes with user actions (button clicks), not GET requests or page loads
**Standard pattern:** Salesforce's built-in `PageReference` redirects and action methods include CSRF tokens automatically

### 13. Open Redirects
**Violation:** Redirecting users to a URL derived from untrusted request parameters
**Fix:** Use hardcoded redirect targets; validate against an explicit allowlist if dynamic redirects needed
```apex
// BAD
String url = ApexPages.currentPage().getParameters().get('returnUrl');
PageReference ref = new PageReference(url);

// GOOD
Map<String, String> allowedRedirects = new Map<String, String>{
    'home' => '/apex/HomePage',
    'dashboard' => '/apex/Dashboard'
};
String key = ApexPages.currentPage().getParameters().get('page');
PageReference ref = new PageReference(allowedRedirects.get(key) ?? '/apex/Default');
```

### 14. Lightning LockerService Disabled
**Violation:** Disabling or not enabling LockerService for Lightning components/apps
**Fix:** Enable Lightning Locker for all AppExchange packages with Lightning components
**Why it matters:** LockerService provides component isolation; disabling exposes component boundaries to exploitation

### 15. Insufficient Escaping in Lightning Components
**Violation:** Rendering user-controlled data without escaping
**Fix:** Use `{!HTMLENCODE(v.attribute)}` in Aura; avoid `{!v.attribute}` for unescaped output; in LWC, use text nodes not `innerHTML`
**Why it matters:** Allows XSS attacks via crafted component attribute values

### 16. Asynchronous Code Timing Vulnerabilities
**Violation:** Multiple separate async requests that can be reordered by attackers
**Fix:** Batch multiple async operations into a single request using `$A.enqueueAction` (Aura) or Promise chains (LWC)
**Why it matters:** Attackers can manipulate execution order of separate async calls

### 17. Insecure Communication
**Violation:** Using HTTP or FTP endpoints
**Fix:** Enforce HTTPS for all external communication; use SFTP for file transfers; use Named Credentials for endpoint management

---

## Connected Apps and External Client Apps (ECAs)

### Spring '26 Change
- **You can no longer create new Connected Apps** as of Spring '26
- Use External Client Apps (ECAs) for all new OAuth integrations

### PKCE Requirements (Proof Key for Code Exchange)
- Required for all Connected Apps and ECAs in use by more than 2 customer production orgs
- Must be enabled regardless of whether the client is public or confidential
- Once enabled, cannot be disabled
- For public clients: do NOT include `client_secret` in token exchange requests

### Non-Compliance Consequences
- AppExchange de-listing
- Temporary or permanent suspension of app's Salesforce interoperation

---

## B2C Commerce Cartridge Security (Additional Requirements)

- Template Best Practices must be followed (see Salesforce Help: Secure Your B2C Commerce Solution)
- CSRF: Include protection in all state-changing controllers
- Open Redirects: Never redirect based on untrusted data
- Content Security Policy: Document and share CSP with customers
- Patches/Upgrades: Direct customers to use separate cartridges for customizations to simplify updates

---

## Agentforce / AI Solution Security (Additional Requirements)

### Prompt Injection Prevention
1. **Validate user-controlled data before prompt inclusion:**
   - Check against an allowlist of acceptable characters/words
   - Enforce length limits to prevent "Do Anything Now" (DAN) attacks
   - Strip or reject zero-width characters and unusual formatting

2. **Use Random-Sequence Enclosures:**
   ```
   // BAD - predictable delimiter
   """<user input>"""
   
   // GOOD - random sequence enclosure
   AK6524SH_YTHW923 <user input here> AK6524SH_YTHW923
   ```
   - Generate token with a cryptographically secure random source
   - Use a NEW token for each inference operation
   - Make tokens long enough to prevent guessing or inference

3. **Prompt Sandwiching:** Place system instructions before AND after user data to limit injection escape radius

### Useful References (Salesforce recommends)
- Salesforce Help: Best Practices for Building Prompt Templates
- OWASP Gen AI Security Project: Prompt Injection
- Promptfoo: Jailbreaking LLMs Comprehensive Guide

---

## Security Policy Program (Required Before AppExchange Listing)

### Required Elements
- Designated security expert involved throughout development
- Corporate security policy documenting customer data protection
- List of all services and artifacts
- Third-party library inventory with versions

### Recommended Elements
- Penetration testing policy
- Vulnerability disclosure program
- Incident response plan
- Regular security training for development team

---

## AppExchange Security Review: Common Blockers

Issues that frequently delay or fail security review:
1. Third-party JS loaded from CDN (not static resources)
2. SOQL injection vulnerabilities
3. CRUD/FLS not enforced
4. Sensitive data in debug logs
5. HTTP (not HTTPS) endpoints
6. Sample/demo code left in production
7. Known CVE libraries without patching or false-positive documentation
8. LockerService disabled
9. PKCE not enabled on Connected Apps/ECAs
10. Open redirect vulnerabilities

Address all of the above before submitting to avoid review delays.
