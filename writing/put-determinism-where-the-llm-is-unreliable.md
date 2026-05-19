# Put determinism where the LLM is unreliable
**Brand voice can live in a prompt. Compliance cannot. The library is not smart — it is right.**

*~1,400 words · drawn from the [Skoda AI Ops](../skoda-ai-ops/) Python compliance library and [Psinest's](../psinest/) planned PII-redaction proxy*

---

The first version of the Skoda content stack — early 2026, before the Python compliance library existed — leaned on prompt-level instructions for German regulatory compliance. The system prompt said things like *don't use the word "Festpreis"*, and *always hedge prices with "ab" or "bis zu"*, and *never imply a guarantee without a "Beispielrechnung" framing*. The list was about fifteen rules long. It worked roughly 95% of the time. It broke within a week.

The break wasn't a system bug. It was the LLM doing what LLMs do — generating plausible-looking content from the prompt, occasionally skipping a rule because the rule felt soft relative to the content goal. *Skoda Clever Kaufen guarantees a fixed price for your next service*, the system wrote, in a Facebook post the operator approved without noticing the regulatory exposure. The word "Festpreis" didn't appear, so the keyword filter passed; the *meaning* of "guaranteed fixed price" was there anyway, in a way the prompt-level rules didn't catch. Under German consumer-protection law and ERS-type advertising regulation, that's a real exposure for a small business that can't absorb regulatory exposure.

The fix wasn't to rewrite the prompt. The fix was to stop relying on the prompt for compliance.

## The pattern

When LLM output is going to be published *publicly* — to a website, a regulated advertising channel, a customer's social feed, an API call that can't be retracted — a 95%-reliable system is a 5%-regulatory-exposure system. The system has to *refuse to publish* content that violates the rules, not be asked nicely to comply.

The pattern that fixes this is to move the rules from the prompt to a deterministic library. In Skoda's case, that's about 300 lines of Python plus tests. The library runs *before* any publishing API call fires. Every payload is checked against the rules that constitute compliance for the domain: German-language signals (permissive heuristic — blocks only affirmative-English-with-zero-German, so legitimate German posts using cognates aren't false-positived), price hedging (every `€` value must be preceded by `ab` / `bis zu` / `ca.` / `etwa`, or the whole text framed as `Beispielrechnung`), a regex denylist with word-boundary enforcement (so `Festpreis` is blocked but `Beispielrechnung` passes), and channel-specific voice rules (LinkedIn requires `wir`, blocks `ich`). **If any check fails, the publishing API call does not fire.**

The library is the *only path* to the publishing surface. Interactive Cowork sessions call it before publishing. Cowork Automations call it before publishing. There is no path to the public-facing API that doesn't go through the library. This is the load-bearing piece of the pattern — if the LLM could bypass the library by calling a different endpoint, the library wouldn't be a compliance gate, it'd be a recommendation.

## Why the library is not smart

The library is intentionally not smart. It doesn't try to understand the *meaning* of the post. It doesn't run a second LLM to judge whether the content is compliant. It just runs a fixed set of regex checks and structural validations and returns *pass* or *fail*. If you ask the library why it failed, it returns the rule that fired and the part of the payload that triggered it. If you ask the library whether a post that's *kind of* close to violating a rule is acceptable, the library has no opinion — it answers within its rules and stops.

This is the point. The LLM is allowed to be wrong about anything; the library refuses to publish until it isn't wrong about the things that matter. The library doesn't have to be right about everything — just about the things that the regulatory regime cares about. The library is *not smart; it is right*. That's the distinction.

There's a temptation to make the library smarter — to run a second LLM as a "compliance reviewer" between the writer-LLM and the publishing API. Don't do this. The compliance reviewer-LLM has the same failure mode as the writer-LLM — it's plausibly correct on 95% of inputs and wrong on 5%, and the 5% it's wrong on is exactly the 5% the regulatory regime cares about. Two LLMs in series multiply the failure rate; they don't reduce it. The deterministic library has a 0% failure rate within its rules. It might have a false-positive rate (legitimate content rejected), and it might have false-negative blindness (rules that aren't in the library aren't checked) — but it doesn't have a *non-determinism* rate. That's the property that makes it the right tool.

## Where the LLM is still doing useful work

The pattern doesn't displace the LLM. The LLM is still doing the work the library can't do: generating the content, drafting the headline, picking the tone, deciding what to write about, brainstorming the campaign angle. *Brand voice can live in a prompt.* The LLM is still the system's writer; the library is the system's editor with a red pencil.

The split is along this line: *can the rule be enforced by a regex with positive context, or does it require judgment?* If regex, library. If judgment, prompt. "Don't use the word 'Festpreis'" is a regex rule, even if it's stated in prose. "Make this post sound enthusiastic without being pushy" is a judgment rule. The library handles the former; the prompt handles the latter.

The two compose. The library is the lower bound — content that fails the library never publishes. The prompt is the upper bound — content that passes the library is also shaped by the prompt's voice and tone guidance. Most posts go through both layers without intervention. A few posts get rejected by the library and re-drafted. The end-state is content that's both on-brand (because the LLM follows the prompt) and compliant (because the library refuses to publish anything that isn't).

## The same pattern, different domains

This pattern generalises far beyond Skoda's specific use-case. Any time LLM output crosses a regulated or trust-bearing boundary, the pattern applies:

- **[Psinest's planned PII redaction proxy](../psinest/README.md#ai-augmentation-roadmap-future-state)** uses the same architectural shape. Every external-LLM call against patient data passes through a redaction layer that strips names, addresses, phone numbers, and other PII before the data leaves the system. The redaction is always-on middleware, not opt-in per call — same as Skoda's library is the only path to the publishing surface.
- **A document drafting assistant in legal practice** would use the same pattern. The LLM drafts; a library checks for privilege-violating disclosures (client identifiers in opposing-counsel emails, settlement figures in non-NDA contexts, conflicts in unscreened intake forms); if any check fails, the draft doesn't send.
- **A financial advisor's note-taker** would use the same pattern. The LLM transcribes; a library checks for suitability-rule violations (recommendations to specific clients that don't match their risk profile, unauthorised projections of returns, missing disclosures); if any check fails, the note doesn't post to the CRM.

The pattern doesn't change. The rule library is the only path to the public-facing surface. The library is right within its rules. The LLM does the work the library can't do. The pattern composes.

## The fork to look for

If you're designing an AI system right now and want to know whether the pattern applies, the fork is this: *would a single bad output cost the business significantly more than the engineering effort to write the rule library?*

If yes — regulated content, public-facing advice, healthcare data, legal communication, financial recommendations — the library is mandatory. The engineering cost is real but bounded (Skoda's library is ~300 lines of Python plus tests). The cost of a single bad publish, in those domains, is order-of-magnitude larger.

If no — internal drafts, side-channel content the human reads before sending, content with cheap rollback — the prompt is enough. Don't over-engineer.

The portfolio's [Skoda AI Ops](../skoda-ai-ops/) case study has the implementation detail end-to-end: which rules are in the library, why each rule is there, what each rule's false-positive profile looks like, how the library evolves as new false-positives are discovered. It generalises directly. The pattern works.

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com)
