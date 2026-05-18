# Cost routing

> **Status:** outline draft — not yet written in full prose.

## What this system actually costs to run
- Order-of-magnitude monthly cost (representative figures, not exact)
- The expensive line items and the cheap ones
- What scales linearly with usage vs what's fixed

## The two cost buckets: production vs acquisition
- **Production cost** — what it costs to produce a unit of content (image, video, post, newsletter, blog).
- **Acquisition cost** — what it costs to land a quote request (Google Ads spend / GA4 conversions).
- These are routed separately and budgeted separately. The AI stack changes production cost dramatically; it does almost nothing for acquisition cost.

## Routing production by economics, not preference
- Image gen on Gemini (cheap, fast, good enough)
- Video gen on Veo (expensive per second, but no alternative at this quality bar)
- Text on a cloud model rotation; never on the local model for the primary path

## Routing acquisition: organic as volume, Google Ads as precision
- Organic content fills the funnel mid-stream and compounds over months. Marginal cost per piece is near zero.
- Google Ads catches high-intent searches at the bottom of the funnel — searches that organic will reach eventually, but not in time for this quarter's leads.
- The decision rule: spend on Google Ads up to the point where the incremental cost-per-acquired-lead equals the expected commission discounted by the close rate. Past that, more spend just burns margin.
- GA4 conversion events feed back into Google Ads for bid optimization, but Firestore `quote_events` is the source-of-truth attribution for board reporting.

## The free-tier ceilings that shape the system
- Upload-Post: 10 uploads / month, free tier
- Firebase: free tier covers current quote volume
- Gmail SMTP: free, with daily caps that are not a constraint at this volume
- GA4: free at this event volume; no concerns at current scale

## What gets cut first if the budget shrinks
- Google Ads first — it's the only variable cost tied to acquisition, and it's the most measurable to throttle.
- Veo next (drop to looped beds, reuse renders).
- Cloud-model rotation (collapse to one provider).
- Newsletter cadence (drop to fortnightly).
- The organic content engine is the last thing cut, because it's the only one that compounds.

---

[← AI stack](ai-stack.md) · [Next: Lessons learned →](lessons-learned.md)
