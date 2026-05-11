---
name: mvp-geo-optimizer
description: Generative Engine Optimization (GEO) skill for optimizing landing pages, SaaS websites, product pages, and marketing copy for AI discoverability. Trigger for product-facing web content, schema markup, LLM visibility, or requests to optimize for ChatGPT, Perplexity, Gemini, and similar AI search tools. This skill is essential for MVP builders; trigger it even if the user doesn't explicitly mention GEO, schema, or structured data, as long as they're writing product-facing content.
---

# MVP GEO Optimizer

Generative Engine Optimization (GEO) ensures your MVP's pages are accurately
summarized and surfaced by AI-powered search engines: Perplexity, ChatGPT Search,
Gemini, Bing Copilot, and others. Unlike traditional SEO (optimizing for crawlers),
GEO optimizes for **LLM comprehension**: how well a language model can extract,
summarize, and confidently cite your product.

---

## Your Job When This Skill Triggers

When writing or reviewing any product/landing page content, you must:

1. **Embed GEO structures inline** as you write, not as a post-hoc audit
2. **Generate the JSON-LD schema block** appropriate for the product type
3. **Output a GEO Scorecard** at the end (see format below)
4. **Flag any GEO anti-patterns** found in existing content

Read `references/geo-patterns.md` for the full pattern library before writing.
Read `references/schema-templates.md` for ready-to-use JSON-LD blocks.

---

## Core GEO Principles

### 1. Entity Clarity (Most Important)
LLMs build a "knowledge graph" of what your product IS. Be explicit:

- State the product category in the first sentence: "X is a [category] that..."
- Name the problem solved in plain language, no metaphors in H1/H2
- Define any brand-specific terms the first time they appear
- Include the founding context: who built it, where, what for

**Bad**: "The future of team collaboration is here."
**Good**: "Nexus is a project management tool for remote software teams that replaces Jira with a no-code interface."

### 2. Semantic HTML Structure
LLMs weight content by HTML hierarchy. Use the right tags:

```html
<main>
  <article>
    <h1>[Product Name]: [Category] for [Target User]</h1>
    <p class="product-description">[One-sentence definition]</p>
    
    <section aria-label="features">
      <h2>Key Features</h2>
      <!-- Each feature as its own <article> or <section> -->
    </section>
    
    <section aria-label="use-cases">
      <h2>Who Uses [Product Name]</h2>
    </section>
    
    <section aria-label="faq">
      <h2>Frequently Asked Questions</h2>
      <!-- Use <details>/<summary> OR explicit Q&A pairs -->
    </section>
  </article>
</main>
```

### 3. JSON-LD Schema (Required on Every Page)
Always inject a `<script type="application/ld+json">` block. Minimum viable:

- `SoftwareApplication` or `Product` for the product itself
- `FAQPage` if FAQ section exists
- `Organization` for the company/brand
- `WebSite` with `SearchAction` if there's a search feature

See `references/schema-templates.md` for complete templates.

### 4. Citation-Friendly Prose
LLMs prefer content they can quote directly in answers. Write so that
individual sentences are self-contained facts:

- Use present tense, active voice
- Avoid pronouns without clear antecedents ("it", "this", "they")
- Each feature description should be a standalone, quotable claim
- Include concrete numbers where possible: "reduces setup time by 80%", "supports 14 integrations"

### 5. FAQ Section (High Impact)
FAQs are the highest-GEO section on any page. LLMs almost always pull from FAQ
content when answering "What is X?" or "How does X work?" queries.

Every MVP page needs at minimum 5 FAQs covering:
1. What is [Product]? (definition + category)
2. Who is [Product] for? (target user)
3. How does [Product] work? (mechanism, 2-3 sentences)
4. How is [Product] different from [top competitor]?
5. How much does [Product] cost? (even "free tier available" is better than silence)

### 6. Entity Disambiguation
If your product name is generic or shares a name with other entities, add
disambiguation signals:

```html
<meta name="description" content="[ProductName] by [CompanyName], [category] for [niche]">
```

And in the body: "**[ProductName]** (by [Company], founded [Year]) is a..."

---

## Output Format

When generating or reviewing a page, output in this order:

### A. The Content/Code
Write the full page content or component with all GEO structures embedded.
Include the JSON-LD block inside a `<head>` section comment or at the end
as a clearly labeled script block.

### B. GEO Scorecard
After the content, always append:

```
## GEO Scorecard

| Signal                        | Status | Notes                          |
|-------------------------------|--------|--------------------------------|
| Entity definition (H1)        | ✅/⚠️/❌ |                               |
| Semantic HTML structure       | ✅/⚠️/❌ |                               |
| JSON-LD: SoftwareApplication  | ✅/⚠️/❌ |                               |
| JSON-LD: FAQPage              | ✅/⚠️/❌ |                               |
| JSON-LD: Organization         | ✅/⚠️/❌ |                               |
| FAQ section (min 5 Q&As)      | ✅/⚠️/❌ |                               |
| Citation-friendly prose       | ✅/⚠️/❌ |                               |
| Competitor differentiation    | ✅/⚠️/❌ |                               |
| Entity disambiguation         | ✅/⚠️/❌ |                               |
| Concrete numbers/metrics      | ✅/⚠️/❌ |                               |

GEO Score: X/10
Top 3 improvements: ...
```

**Legend**: ✅ = implemented, ⚠️ = partial/weak, ❌ = missing

---

## Anti-Patterns to Flag

When reviewing existing content, call out:

- **Vague H1s**: Taglines that don't state product category ("Build something people love")
- **Pronoun soup**: "It helps them do this with their teams"; LLMs lose the referent
- **Feature-list-only pages**: Bullet lists with no prose context; LLMs can't build semantic chains from bullets alone
- **Missing schema**: Any product page without JSON-LD is invisible to structured AI search
- **No FAQ**: Single highest-impact missing element
- **Jargon without definition**: Internal brand terms used before being explained
- **Passive descriptions**: "Used by thousands of teams" vs "10,000 teams use [Product] to..."

---

## Workflow for New Pages

1. Collect: product name, category, target user, top 3 features, main competitor, pricing
2. Read `references/geo-patterns.md` for the content type (SaaS, marketplace, tool, API)
3. Read `references/schema-templates.md` and select the right JSON-LD template
4. Draft with all GEO signals embedded from the start
5. Output content + GEO Scorecard

## Workflow for Existing Pages

1. Read the existing content
2. Run through the GEO Scorecard checklist
3. Rewrite or patch weak/missing signals
4. Output the improved version + before/after GEO Scorecard comparison