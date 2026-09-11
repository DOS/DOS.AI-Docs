# CRM & Marketing Connectors

Connect customer relationship management (CRM) systems, social media schedulers, and ad platforms to DOSClaw AI agents. Enable automated lead capture, pipeline updates, blog creation, and multi-channel social campaign execution with review-gated approval workflows.

## Supported Platforms

| Platform | Type | Key Capabilities |
| --- | --- | --- |
| [HubSpot CRM](hubspot.md) | Inbound CRM | Contact lookup, company data, deal pipelines |
| [Salesforce Enterprise CRM](salesforce.md) | Enterprise CRM | Leads, Accounts, Opportunities, Service Cloud cases |
| [Crove Post](crove-post.md) | Social Scheduler | Multi-channel publishing, analytics, campaign review |
| [NueLink](nuelink.md) | Social Automation | Cross-platform scheduling queues (10+ networks) |
| [WordPress Direct CMS](wordpress.md) | Content Management | SEO blog post drafts, category tagging, media upload |
| [Meta Ads & Lead Ads](meta.md) | Ad Automation | Real-time lead form webhook, instant follow-up, CAPI |

---

## Marketing Governance & Review Gates

To safeguard brand voice and ad spend, DOSClaw enforces governance modes:
- **Review Gated (`draft_only`)**: The agent drafts articles, social posts, or ad actions and submits them for human review.
- **Approval Required (`approval_required`)**: Mutating actions require explicit confirmation from a manager before dispatch.
- **Autonomous (`autonomous`)**: Routine tasks (lead ingestion, contact lookups) execute automatically.
