# Business problem

## What the business is

Skoda Clever Kaufen is an independent referral business operating in Germany. The model is straightforward: private customers configure a new Škoda on a clean landing page, request a personalized quote, and a partner dealer follows up to close the sale. The operator is paid a referral commission on completed deals. The business is not affiliated with Škoda Auto.

This is a price-sensitive, comparison-heavy purchase. Customers who land on the site are typically two-to-six weeks from buying. They are not looking for inspiration; they are looking for a credible quote, a fast response, and someone who knows the configuration trade-offs. The competitive set is dealer websites, comparison portals like mydealz and AutoScout24, and direct manufacturer pages.

## What the constraint is

The operator is one person. There is no marketing team, no in-house content writer, no video producer, no social media manager, and no engineering team. The business was built to be operated solo, on consumer-grade tooling, with a small and disciplined paid acquisition budget on Google Ads sitting alongside the organic content engine.

The reason paid is in the mix at all — despite the obvious appeal of running a "zero ad spend" story — is that high-intent automotive search has a discoverability problem for new brands. A buyer in the final two weeks before purchase is searching with very specific intent ("Octavia Combi Selection 2.0 TDI Konfigurator Rabatt") and that surface is dominated by manufacturer pages and aggregator portals. Organic SEO will eventually get there; Google Ads can be there tomorrow. The right answer is both: organic content as the volume play, Google Ads as the precision channel that catches in-market buyers while the organic library compounds.

The decision to add paid changes the system's observability requirements but not its safety model. GA4 with Consent Mode v2 (default-denied until the cookie banner gets consent) feeds conversion events to both Google Ads and to a Firestore `quote_events` collection. The Firestore copy is the source of truth for attribution back to the Airtable lead — Google Ads' attribution is treated as directional, not gospel.

Three constraints fall out of the one-person setup:

- **Content output must match category convention** without a content team. Buyers expect a steady flow of model comparisons, configuration guides, financing explainers, and short video walkthroughs. Without that, the site reads as inactive and loses to the larger players in search results and social distribution.
- **Lead handling must be fast and predictable.** A quote request that sits for 24 hours is already cold; the buyer has moved on to two other portals. The auto-responder with a personalized price proposal is the first impression of the brand and the highest-leverage piece of automation in the entire system.
- **Compliance cannot be soft.** This is a real commercial business operating under German consumer protection and GDPR rules. The wrong word in a published price claim ("Festpreis", "garantierter Preis", "Rechnung") is not a writing error; it's a legal exposure. The system has to refuse to publish content that crosses those lines, regardless of what an LLM was happy to draft.

## Why a traditional content stack does not solve this

The obvious answer is a CMS plus a freelance content writer plus a social media scheduler. That stack costs roughly €2,500–€4,000 per month at the production cadence expected in this category (13 blog posts a quarter, weekly newsletters, weekly short-form video, daily social presence). At a referral business with low commission per unit and uncertain conversion, that cost structure is the difference between a marginal business and an unprofitable one.

The second-order problem is editorial velocity. Even with a writer on retainer, the loop from "we should publish something about the new model variant" to a live post is days, not hours. By the time the content is out, the variant has already been discussed everywhere else.

## Why an AI-orchestrated stack is the right answer here, and where it isn't

For a one-person operation in a content-heavy category, generative AI changes the unit economics. A blog post that took €150 and three days now takes the operator an hour of supervision and effectively zero marginal cost. A YouTube Short that required a videographer and an editor is a Veo render with synchronized audio, produced in minutes. A personalized price proposal that previously required an inside sales rep is a Cloud Function plus a PDF builder plus an SMTP send.

This works *for content production and distribution*. It does not work for the parts of the business that are inherently relational: the dealer relationship, the actual sale conversation with the customer, the negotiation. The system is deliberately not trying to automate those. The lead hand-off is the boundary — once the lead is in the CRM, a human picks up the phone.

The rest of this case study is the design of the layer between "operator has an idea" and "post is live on the right channel with the right guardrails." That layer is the work.

---

[← Back to README](../README.md) · [Next: Architecture evolution →](architecture-evolution.md)
