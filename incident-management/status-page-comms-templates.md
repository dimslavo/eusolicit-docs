# Status Page Communications Templates

**Version**: 1.0 | **Story**: PE.06 (21-6) | **Last updated**: 2026-05-05

> **Usage**: These templates are paste-edit-ready for the on-call IC to use during SEV-1 incidents.
> Post manually to the status channel (Slack `#platform-incidents` + customer-visible status page).
>
> **Anti-pattern guard #7**: No Statuspage.io or automated status-page integration in scope of PE.06.
> Templates are copy-paste only. A Statuspage.io integration lands in a future epic if customer
> count justifies it.
>
> **No-blame rule**: All templates use explicit no-blame phrasing. No individual names,
> no vendor blame phrasing. If a vendor caused the outage, state: "a third-party service reported
> an incident" — do NOT name the vendor as the cause in customer-facing communications unless
> the vendor has publicly confirmed the incident on their own status page.
>
> **SLA-scope reminder**: EXEMPT incidents (KraftData, Stripe, Google/Microsoft OAuth, ClamAV)
> should use the EXEMPT template variant below rather than the SEV-1 production-outage template.

---

## Template 1: SEV-1 Initial

> **When to use**: Within 30 minutes of SEV-1 declaration.
> **Paste into**: Slack `#platform-incidents` + status page (if available).

---

**Subject**: Service Disruption — EU Solicit Platform [Investigating]

We are currently investigating reports of **\<symptom in plain language, e.g., "difficulties completing proposals and opportunity searches"\>**.

**Impact**: \<scope, e.g., "All users may be affected. The EU Solicit proposal submission, opportunity discovery, and billing features are currently unavailable."\>

**Started at**: \<time UTC\>

We have identified the issue and our team is actively working to restore service. We will provide an update within **1 hour**.

We apologise for the disruption to your workflow.

— EU Solicit Platform Team

---

## Template 2: SEV-1 Update

> **When to use**: Every 1 hour while a SEV-1 incident is ongoing.
> **Replace the previous update** with this template.

---

**Subject**: Service Disruption — EU Solicit Platform [Update \<N\>]

We are continuing to investigate the service disruption affecting **\<scope\>** that began at **\<start time UTC\>**.

**Current understanding**: \<one or two sentences on hypothesis or confirmed root cause. Use plain language. If root cause is unknown: "We are still working to identify the root cause. Our current investigation is focused on \<area\>."\>

**Actions taken**: \<brief, one-sentence summary of what has been tried. E.g., "We have applied a configuration change and are monitoring for recovery."\>

**Status**: \<Investigating | Root cause identified | Fix in progress | Monitoring recovery\>

Next update expected at: **\<time UTC\>** or sooner if the situation changes.

— EU Solicit Platform Team

---

## Template 3: SEV-1 Resolved

> **When to use**: After all §Verification checks pass and the incident is resolved in PagerDuty.
> **Post after** the SEV-1 Update with status "Monitoring recovery" is confirmed stable.

---

**Subject**: Service Disruption — EU Solicit Platform [Resolved]

The service disruption that began at **\<start time UTC\>** has been resolved as of **\<resolve time UTC\>** (duration: **\<X hours Y minutes\>**).

**Root cause**: \<one sentence in plain language. E.g., "A database storage capacity issue caused write operations to fail. The issue has been resolved by expanding storage capacity."\>

**Impact**: \<confirmation of what was affected and for how long\>

**What we are doing to prevent recurrence**: A post-mortem will be conducted within 5 business days. Findings and action items will be tracked internally. We will share relevant improvements in our next platform update.

We sincerely apologise for the disruption. If you have questions about data integrity or billing impact, please contact **support@eusolicit.eu**.

— EU Solicit Platform Team

---

## Template 4: EXEMPT Vendor Incident (SEV-2)

> **When to use**: For SLA-EXEMPT incidents (Stripe, KraftData, Google/Microsoft OAuth, ClamAV)
> where the platform is affected by an upstream vendor's outage.
> **Do not** use the SEV-1 templates for EXEMPT incidents — the platform SLA is not impacted.

---

**Subject**: Feature Disruption — \<Feature Name\> [Investigating Upstream Issue]

We are aware that **\<feature, e.g., "Google OAuth login / payment processing / opportunity data refresh"\>** is currently experiencing disruption.

**What we know**: \<vendor\> reported an incident affecting their service at **\<time UTC\>**. EU Solicit's \<feature\> relies on this third-party service.

**What is working**: \<list unaffected functionality. E.g., "Email-password login, existing sessions, proposal editing, and document management are all functioning normally."\>

**What to do**: \<alternative action for users. E.g., "Please use email-password login as an alternative to Google OAuth while this issue is being resolved."\>

We are monitoring \<vendor's\> status page and will restore the affected feature automatically once their service recovers.

— EU Solicit Platform Team

---

## No-Blame Phrasing Guidance

| Instead of | Use |
|------------|-----|
| "A team member accidentally deleted the configuration" | "A configuration change caused the storage threshold to be reset" |
| "The developer pushed a broken deploy" | "A recent application update introduced a regression" |
| "Stripe's outage broke our billing" | "A third-party payment service reported an incident affecting payment processing" |
| "Google's OAuth system failed" | "The third-party authentication provider reported an incident" |
| "We made a mistake" | "Our response process did not detect the issue before it impacted customers" |
| "Person X caused the incident" | (Never write this. If needed, redirect to the root-cause system/process.) |

> **Rule**: Customer-facing communications NEVER name individual engineers. Status-page messages
> are a public record. Any attribution of cause to an individual in a public communication causes
> lasting harm without any benefit to the customer.
