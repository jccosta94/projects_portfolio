# Configure a reliable platform, don't build a custom agent from scratch
**The consulting operating model that's worked for Salesforce delivery for two decades applies, almost unchanged, to AI cockpit delivery in 2026.**

*~1,800 words · drawn from the [Claude Cowork Vertical Workflows](../claude-cowork/) consulting practice and prior Salesforce delivery experience*

---

The default move in early-2026 AI consulting is to position every engagement as a "bespoke AI build." You hear it in the pitches: *we'll build you a custom AI agent for your business*. The promise is appealing — a tailored solution, fitted to the customer's specific workflows, built from the ground up to their requirements. The cost-of-delivery is what kills it. Every engagement starts from zero. Every workflow gets hand-authored. Every integration is debugged in production. The second customer's build can't reuse anything from the first because the first was a snowflake. The cost of building swallows the engagement fee, the consultancy runs out of margin by the third customer, and the customer is left holding a system nobody can maintain.

This essay is about the alternative operating model: configure a reliable platform for the customer's vertical, rather than build a bespoke alternative. It's the model I bring across from years in Salesforce delivery, and applying it to AI cockpit delivery is what makes a one-person consulting practice survive past the first three customers.

## What Salesforce consultants have known for a long time

Salesforce consultants do not write their own CRM. They configure Salesforce. The work — the actual billable, value-generating work — is in the configuration: which objects, which fields, which page layouts, which validation rules, which automation flows, which integration patterns. The platform vendor has invested billions in making the substrate reliable — the underlying database, the security model, the integration plane, the developer SDK, the auth model, the upgrade path, the global infrastructure. A consultant who tries to build their own CRM to compete with Salesforce ends with a worse Salesforce that took 10× the engineering effort and is still missing the things Salesforce ships for free.

This is a settled question in the SaaS consulting world. Nobody seriously argues that a Salesforce consultant should build their own customer database. The discipline of "configure the reliable platform, don't build the substrate" is so deeply baked into the practice that it's no longer named — it's just how the work happens.

The interesting thing, in 2026, is that the same discipline maps almost unchanged onto AI cockpit delivery. The platform vendor in this analogy is Anthropic; the platform is Claude Cowork (or Claude Code, or Claude in Chrome, or whatever cockpit the customer's role calls for). Anthropic has invested in the desktop cockpit, the MCP integration plane, the skill-loading runtime, the scheduled-tasks runner, the security model around tool execution, the global infrastructure for inference. *Configuring that platform for the customer's vertical* is the consulting work. *Building a competing AI agent from scratch* is the trap.

## The trap the first version of the practice fell into

The first version of my AI consulting practice, in early 2026, didn't have the discipline yet. Every prospect got a custom-scoped build. I'd take a discovery call, draft a custom architecture, scope a custom skill library, build custom MCP servers, hand-author custom memory files. Every engagement was a unique system. The deliverables looked good in isolation; the practice didn't survive past the third customer.

The cost structure was the killer. A bespoke build for a single clinic takes about three weeks of focused work — discovery, architecture, configuration, deployment, hand-holding through the first weeks of operation. At the price point a single clinic can pay (somewhere between €5,000 and €10,000 for the initial build, plus a managed-service retainer), the build phase consumes most of the engagement fee. The retainer pays for ongoing operations. There's no margin to compound across engagements because every engagement starts from zero — the second clinic's build doesn't reuse anything from the first.

I ran this version of the practice for about two months before the math caught up. The fix wasn't to charge more — clinics can't pay more, and charging more pushes the segment to "boutique custom AI builds" which is a totally different market from "small healthcare practices that need an AI cockpit". The fix was to change the operating model.

## The horizontal version that didn't work either

The second version was the inverse: a single horizontal "Command Center" package, sold the same way to every customer regardless of vertical. The marketing was clean — *we set you up with Claude Cowork plus a core skill library plus a curated MCP fleet*. The discovery work shrank from three weeks to one. The cost-of-delivery came down to something I could sell at a price point clinics and law firms and trades businesses could actually afford.

It didn't work either, but for a different reason. The horizontal package was *too generic to move the operator's bottleneck in any given vertical*. A clinic owner needs patient recall, no-show reduction, deposit collection, intake digitalization. A law firm needs matter intake with conflict checks, deadline diary management, document drafting with review gates, privilege-aware communication logging. A home services owner needs missed-call capture, job intake and dispatch, route briefs, on-site checklists. Selling the same Command Center to all three meant the clinic didn't get a recall loop, the firm didn't get conflict-check intake, and the home services owner didn't get dispatch — they all got inbox triage and a content calendar, and they all felt under-served.

The horizontal version solved the cost-of-delivery problem (one package, fast to ship), but it created a deliverable-thinness problem in its place. The third version had to solve both.

## What the third version actually is

The third version, the one that ships, is the model from Salesforce delivery applied wholesale. The platform stays fixed. The cockpit shape stays fixed. The 11-skill core library stays fixed. What varies per vertical is *the configuration* — the MCP fleet (which PMS, which DMS, which dispatch tool), the skill-library composition (which core skills are active, which vertical-specific extensions are added), the memory seeds (what counts as a VIP, what's special-category data, what's privileged), and the HITL gate model (owner-eyes-only on patient identifiers in healthcare, supervising-lawyer review on every drafted document in legal, dispatcher-confirms in field services).

None of this is code. All of it is configuration. Adding a new vertical to the practice means adding ~5-7 skills, ~2-3 MCP-server bindings, and a vertical-tuned HITL gate model. The base platform — Cowork, the integration plane, the runtime, the security model — does not change. The discovery for a new engagement is ~3 days, not 3 weeks. The deliverable is *specific to the vertical* because the per-vertical wiring is specific. The cost-of-delivery sits at a level where the practice can run a three-vertical catalog without going broke on the second customer.

This is exactly how Salesforce consulting works. A Salesforce consultant doesn't build a new CRM for the next dental practice; they configure Salesforce for dental practices. They don't build a new conflict-checking tool for the next law firm; they configure Salesforce's matter-management cloud and add vertical-specific automation. The substrate is fixed; the configuration is the consulting.

## Why this is hard to do well

The hard part isn't the platform choice. The hard part is the discipline to refuse custom code. When a customer asks for a feature that would require changing the underlying platform — adding a custom MCP server that doesn't fit the core integration plane, hand-rolling a workflow that bypasses the standard skill-loading runtime, writing bespoke Python that runs outside the Cowork-managed environment — the temptation is to say yes. It's a one-day win for the customer in front of you. It's a multi-year cost for the consulting practice, because the next customer in the same vertical doesn't have that custom code, the third customer doesn't either, and now you have one snowflake in a catalog that was supposed to be configured-not-built.

The Salesforce consulting world has a well-developed culture around this. "Configuration vs customisation" is a known distinction; "managed package vs custom code" is a known trade-off; "extend the platform vs replace the platform" is a settled question. The AI consulting world doesn't have this culture yet — the field is too new, the platforms are too new, the consultants are mostly coming from product or engineering backgrounds rather than from delivery consulting. The discipline has to be imported from somewhere; for me, it was Salesforce.

## When this doesn't apply

There are two cases where the "configure, don't build" model doesn't work:

**The platform doesn't exist for the customer's problem yet.** If you're a consultant working in a vertical where no reliable AI cockpit platform exists, the configure-don't-build model has nothing to configure. The honest answer in this case is to either wait for the platform to ship (most "we need an AI for X" problems will have a vendor in 12-24 months), or to position the engagement as platform-building work and price it accordingly — recognising that you're now the platform vendor for that customer, not a consultant on top of someone else's platform.

**The platform's roadmap doesn't survive the engagement.** The configure-don't-build model assumes the platform vendor will keep investing in the substrate. If you're configuring a platform whose vendor is winding down, getting acquired into a competitor, or running out of runway, the operating model has the wrong substrate. The hedge here is to pick platforms with credible commitment signals — venture-backed at scale, profitable at scale, or open-source with active maintenance.

Both of these are real risks. The first is solved by patience; the second is solved by platform selection. Neither argues against the model when it does apply.

## The fork to look for

The fork that tells you whether the configure-don't-build model fits your consulting practice is this: *would the same customer hire a SaaS consultant for this problem if the platform were Salesforce or Microsoft Dynamics or HubSpot?*

If yes — they'd hire a Salesforce consultant for "we need a customer database with custom workflows", they'd hire a Dynamics consultant for "we need an ERP with vertical-specific automation" — then the AI version of the same engagement follows the same operating model. Configure the platform, don't write a competing one.

If no — they wouldn't hire a SaaS consultant because their problem is genuinely new, or genuinely outside the substrate any existing platform provides — then you're not consulting, you're building. Price and pitch accordingly.

The portfolio's [Claude Cowork Vertical Workflows](../claude-cowork/) case study is the working example of the model end-to-end. The three vertical templates — Practice Command, Legal Efficiency Suite, Field Service Command — are configurations, not code. Two of them are live in pilot. The third is packaged and waiting. The cost-of-delivery per engagement is at a level where the practice can run more than three customers without the math collapsing. Which is the test.

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com)
