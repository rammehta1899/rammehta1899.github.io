---
title: "Inbound AI Agent Traffic: System Architecture Lessons"
description: "United Airlines banned third-party AI agents. Here is how platform leaders must architect M2M endpoints, rate limits, and zero-trust auth for automated bots."
categories: ["Engineering", "AI/ML"]
tags: ["platform-engineering", "ai-agents", "api-design", "security", "distributed-systems"]
labels: ["agent-workflows", "engineering-leadership", "security"]
---

When United Airlines recently updated its policies to prohibit third-party autonomous AI agents like Meta's Muse and OpenAI's dots from booking flights and completing check-ins, many observers framed it purely as a commercial battle over customer ownership. That narrative misses the core system engineering crisis. According to reporting on [United Airlines banning AI agents from booking flights](https://www.cbsnews.com/news/united-airlines-bans-ai-agents-booking-flights/), the carrier pointed to security risks, credential exposure, and federal compliance rules. Underneath the policy update lies a structural mismatch between how web platforms were designed and how autonomous software callers interact with backend services. We are watching the breakdown of assumptions built into application backends over the last two decades.

Consumer web interfaces were built for human pacing. They assume a person clicks a button, waits half a second for a response, reads search results, ponders, and perhaps selects a second link. When an autonomous agent acts on behalf of a traveler, those physical constraints vanish. An LLM agent running a task execution loop does not make three targeted searches. It fires dozens or hundreds of concurrent requests, mutating search parameters across dates, airports, and price windows to find an optimal itinerary.

## Infrastructure Costs and the Search-to-Book Exploded Ratio

Aviation analyst Henry Harteveldt noted that flight searches carry direct operational costs for carriers. Traditional travel metasearch engines like Kayak or Google Flights operate under established commercial contracts, predictable query volumes, and structured caching agreements. Autonomous agents scraping web pages or calling internal endpoints operate without those boundaries.

They create massive spikes in high-cost backend queries with almost zero guaranteed transaction conversion. While optimized specialized engines like [GPUWorks](https://github.com/flitai-io/gpuworks) manage high concurrency across thousands of model replicas on bare metal or cloud nodes with minimal overhead, legacy enterprise platforms carry middleware stacks, complex database joins, and third-party inventory booking fees for every look-up. A sudden surge of unauthenticated search loops consumes server compute, inflates infrastructure costs, and threatens to degrade availability for human users browsing the application.

## Credential Delegation and Regulatory Liability

The technical risks compound when an agent moves from read-only searching to write operations like booking or flight check-in. To complete a reservation, travelers frequently share raw credentials, including payment information and loyalty accounts such as MileagePlus. Passing raw passwords or auth tokens to an external personal assistant exposes sensitive personal data to interception, accidental account modifications, or unauthorized mileage redemptions.

Beyond security, regulatory boundaries break down. Federal regulations mandate that airlines confirm passengers have personally verified their luggage contains no hazardous materials prior to issuing a boarding pass. When an AI agent automatically answers that prompt during an automated check-in loop, the system logs a positive attestation that never actually occurred with the passenger. The legal liability shifts directly back to the enterprise. You cannot delegate statutory human attestations to an automated background script.

## Rethinking Inbound Boundaries: Web-Scraping vs. Native Partner APIs

Blocking traffic via Web Application Firewalls (WAF) or bot detection systems is a temporary patch. It degrades into an arms race where third-party agents route requests through residential proxy pools while platform teams continually tune IP throttling rules.

A different pattern comes from carriers like Southwest, which opted to expose flight inventory directly inside ChatGPT. Instead of forcing an autonomous caller to parse HTML elements or simulate browser actions on internal endpoints, Southwest established a controlled integration path. This architecture isolates automated queries inside structured, governed endpoints designed explicitly for machine callers.

If your platform team treats autonomous agent traffic purely as another strain of malicious scraping, you are missing the underlying architectural evolution. Systems require dedicated machine-to-machine (M2M) paths that enforce distinct service level agreements, explicit delegation scopes, and structural rate limits. Compact computational pipelines demonstrate how much work can be packed into tight compute footprints, much like specialized edge algorithms like [HaCI](https://github.com/RenderBear/haci) process dense image structures with low memory overhead. Enterprise backends must bring that same structural discipline to their ingress control planes.

## Practical Engineering Strategy for Autonomous Caller Traffic

To handle inbound autonomous agents without relying entirely on reactive blocking, platform leaders should establish three specific architectural primitives:

First, mandate tokenized delegation instead of raw credential sharing. End users should never supply main account passwords to external AI tools. Platforms must implement scoped OAuth 2.0 flows with granular authorization bounds. An agent should receive a short-lived token restricted strictly to search or itinerary construction, requiring step-up human authentication for payment processing or policy attestations.

Second, decouple public search backends from core transactional engines. Machine callers should be routed to aggressively cached read replicas or edge indexes. If an agent demands real-time seat availability, that request must pass through a dedicated machine gateway enforcing strict rate limiting, request concurrency caps, and cost attribution metrics tied to the requesting developer key.

Third, encode regulatory and compliance checks into API payload contracts. If an action requires explicit human confirmation, the API payload must demand a signed, non-repudiable user assertion token rather than a simple boolean flag. If an API call lacks that verifiable chain of custody, the backend must reject the transaction with a clear status code indicating missing human verification.

The decision by United Airlines is not an isolated policy stance. It is an early signal of what happens when client-side automation collides with backend infrastructure built for human interactions. Blocking non-compliant bots protects operational availability today, but the long-term solution requires building platform infrastructure that accepts automated callers on explicit, governed, and secure terms.

## References
- [United Airlines bans AI agents from booking flights on behalf of travelers](https://www.cbsnews.com/news/united-airlines-bans-ai-agents-booking-flights/)
- [GPUWorks lightweight cluster manager](https://github.com/flitai-io/gpuworks)
- [HaCI Harmonic Contour Integration engine](https://github.com/RenderBear/haci)