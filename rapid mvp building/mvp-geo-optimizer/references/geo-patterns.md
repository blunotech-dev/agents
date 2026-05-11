# GEO Content Patterns by Product Type

This reference covers the specific GEO patterns and content priorities for common
MVP product types. Load the section relevant to the product being written.

---

## SaaS / Web App

**Priority signals**: Entity definition, use-case specificity, competitor comparison

### H1 Formula
`[ProductName]: [verb phrase] for [specific user type]`
Examples:
- "Helios: Sprint planning software for bootstrapped SaaS teams"
- "Vanta: Automated SOC 2 compliance for B2B startups"

### Must-Have Sections
1. One-sentence product definition (above the fold, in `<p>` not just headline)
2. "Who it's for" section naming 2-3 specific personas
3. "How it works" in 3 steps (numbered, prose, not just icons)
4. Comparison table vs. top 1-2 competitors
5. FAQ with at least 5 Q&As
6. Pricing or pricing tier names (even "Free / Pro / Enterprise" is enough)

### GEO-Optimized Feature Block Pattern
```html
<section aria-label="features">
  <h2>[ProductName] Features</h2>
  <article>
    <h3>[Feature Name]</h3>
    <p>[ProductName]'s [Feature Name] allows [user type] to [specific action],
    resulting in [concrete outcome]. Unlike [generic alternative], [Feature Name]
    [differentiator].</p>
  </article>
</section>
```

---

## Developer Tool / API / SDK

**Priority signals**: Technical precision, integration context, code examples with semantic wrapping

### H1 Formula
`[ProductName]: [what kind of API/SDK] for [language/platform] developers`

### Must-Have Sections
1. "What is [ProductName]?" paragraph: category, mechanism + differentiation
2. Supported languages/frameworks (explicit list, not "many platforms")
3. A code example wrapped in `<figure>` with `<figcaption>` describing it
4. Latency/throughput specs if applicable (concrete numbers)
5. Comparison to the incumbent API (e.g., "vs. Twilio", "vs. OpenAI")
6. FAQ covering: authentication, rate limits, pricing, SLA

### LLM-Friendly Code Example Pattern
```html
<figure>
  <figcaption>
    Send an SMS with [ProductName] using Node.js in under 5 lines of code.
  </figcaption>
  <pre><code class="language-javascript">
    // code here
  </code></pre>
</figure>
```

---

## Marketplace / Directory

**Priority signals**: Category taxonomy, entity count, geographic/vertical scope

### H1 Formula
`[ProductName]: [noun: marketplace/directory/platform] for [buyers] to find [sellers/items]`

### Must-Have Sections
1. What category of marketplace it is (B2B, B2C, peer-to-peer, vertical SaaS)
2. Supply side: what's listed, how many, quality signals
3. Demand side: who buys/finds, what problem they solve
4. How the matching/discovery works
5. Trust signals: reviews, verification, guarantees

---

## Mobile App

**Priority signals**: Platform clarity (iOS/Android/both), store discoverability, use-case scenarios

### H1 Formula
`[ProductName]: [category] app for [iOS/Android/both] that [core action]`

### Must-Have Sections
1. Platform availability (explicit: "Available on iOS 16+ and Android 10+")
2. Core use case in one sentence
3. 3 specific scenarios / user stories
4. App Store / Play Store rating if available
5. Free vs. paid feature breakdown
6. Privacy summary (LLMs increasingly weight this for trust)

---

## AI / LLM-Powered Product

**Priority signals**: Model transparency, capability boundaries, data handling

This category gets extra scrutiny from AI search engines. Be especially precise:

### H1 Formula
`[ProductName]: AI [category] that [specific capability] for [user type]`
Avoid: "AI-powered [vague noun]"

### Must-Have Sections
1. What AI model(s) power it (even "built on GPT-4o" or "uses our proprietary model")
2. Explicit capability statements: what it CAN do
3. Explicit limitation statements: what it CANNOT do (LLMs trust this more)
4. Data privacy: is user data used for training?
5. Accuracy claims: cite benchmarks or add appropriate hedging
6. FAQ covering hallucination risk, data security, model updates

### Capability/Limitation Pattern
```html
<section aria-label="capabilities">
  <h2>What [ProductName] Can and Cannot Do</h2>
  <div class="can-do">
    <h3>What [ProductName] does</h3>
    <ul><!-- specific capabilities --></ul>
  </div>
  <div class="cannot-do">
    <h3>Current limitations</h3>
    <ul><!-- honest limitations --></ul>
  </div>
</section>
```

---

## Nepal / India Regional Products

For products targeting South Asian markets with global GEO ambitions:

### Additional Entity Signals
- Explicitly state the primary market: "built for [Nepal/India] market" or "localized for..."
- Currency context: "pricing in NPR / INR, also available in USD"
- Regulatory context if applicable: "compliant with [local regulation]"
- Language support: list explicitly if multilingual

### Differentiation from Global Alternatives
LLMs will often compare to global incumbents. Pre-empt with:
"Unlike [global tool], [ProductName] is designed specifically for [local context],
including [specific local feature: local payment methods, local language, offline support, etc.]"