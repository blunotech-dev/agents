# JSON-LD Schema Templates for GEO

Select and adapt the templates below based on the product type.
Always include at minimum: SoftwareApplication (or Product) + Organization.
Add FAQPage whenever a FAQ section exists.

---

## 1. SoftwareApplication (Primary: use for any web app or SaaS)

```json
{
  "@context": "https://schema.org",
  "@type": "SoftwareApplication",
  "name": "[ProductName]",
  "applicationCategory": "[e.g. BusinessApplication, DeveloperApplication, UtilitiesApplication]",
  "applicationSubCategory": "[e.g. Project Management, Analytics, CRM]",
  "operatingSystem": "Web, iOS, Android",
  "description": "[One to two sentence product description. Should match your H1 formula.]",
  "url": "https://[yourproduct.com]",
  "offers": {
    "@type": "Offer",
    "price": "0",
    "priceCurrency": "USD",
    "description": "Free tier available. Pro plan from $X/month."
  },
  "featureList": [
    "[Feature 1]",
    "[Feature 2]",
    "[Feature 3]"
  ],
  "author": {
    "@type": "Organization",
    "name": "[Company/Founder Name]",
    "url": "https://[company.com]"
  },
  "datePublished": "[YYYY-MM-DD]",
  "softwareVersion": "[version or 'Beta']",
  "screenshot": "https://[yourproduct.com/screenshot.png]",
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.8",
    "reviewCount": "120"
  }
}
```

**Notes**:
- `applicationCategory` must be a schema.org approved value. Common ones: `BusinessApplication`, `DeveloperApplication`, `EducationalApplication`, `FinanceApplication`, `HealthApplication`, `ProductivityApplication`, `UtilitiesApplication`
- Remove `aggregateRating` if you don't have real ratings yet
- Remove `screenshot` if not available; add when you do

---

## 2. Organization (Always include alongside the product schema)

```json
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "[Company or Brand Name]",
  "url": "https://[yourproduct.com]",
  "logo": "https://[yourproduct.com/logo.png]",
  "description": "[One sentence: what your company/project builds and for whom]",
  "foundingDate": "[YYYY]",
  "foundingLocation": {
    "@type": "Place",
    "name": "[City, Country]"
  },
  "sameAs": [
    "https://twitter.com/[handle]",
    "https://linkedin.com/company/[handle]",
    "https://github.com/[handle]"
  ]
}
```

**Note**: `sameAs` links are high-value for entity disambiguation. Always include
Twitter/X, LinkedIn, and GitHub where they exist.

---

## 3. FAQPage (Add whenever a FAQ section is present)

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "What is [ProductName]?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "[ProductName] is a [category] for [target user] that [core value prop]. It [mechanism in one sentence]."
      }
    },
    {
      "@type": "Question",
      "name": "Who is [ProductName] for?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "[ProductName] is designed for [persona 1], [persona 2], and [persona 3] who need to [core job-to-be-done]."
      }
    },
    {
      "@type": "Question",
      "name": "How does [ProductName] work?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "[ProductName] works by [mechanism step 1]. Users then [step 2]. The result is [outcome]."
      }
    },
    {
      "@type": "Question",
      "name": "How is [ProductName] different from [Competitor]?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Unlike [Competitor], [ProductName] [key differentiator]. [ProductName] also [secondary differentiator], which [Competitor] does not offer."
      }
    },
    {
      "@type": "Question",
      "name": "How much does [ProductName] cost?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "[ProductName] offers a free tier with [limits]. Paid plans start at $[X]/month, which includes [features]."
      }
    }
  ]
}
```

**Note**: Each FAQ answer should be a complete standalone paragraph. LLMs pull
these answers verbatim when responding to "What is X?" queries.

---

## 4. WebSite (Add to homepage only)

```json
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "[ProductName]",
  "url": "https://[yourproduct.com]",
  "description": "[Product description]",
  "potentialAction": {
    "@type": "SearchAction",
    "target": {
      "@type": "EntryPoint",
      "urlTemplate": "https://[yourproduct.com]/search?q={search_term_string}"
    },
    "query-input": "required name=search_term_string"
  }
}
```

Remove `potentialAction` if the product has no search feature.

---

## 5. Product (Use for physical/digital products, not SaaS apps)

```json
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "[ProductName]",
  "description": "[Product description]",
  "brand": {
    "@type": "Brand",
    "name": "[Brand Name]"
  },
  "offers": {
    "@type": "Offer",
    "price": "[X.XX]",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock",
    "url": "https://[yourproduct.com/buy]"
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.7",
    "reviewCount": "85"
  }
}
```

---

## Combining Multiple Schemas

Embed all applicable schemas in one `<script>` block as a JSON-LD `@graph`:

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "SoftwareApplication",
      "name": "[ProductName]",
      ...
    },
    {
      "@type": "Organization",
      "name": "[Company]",
      ...
    },
    {
      "@type": "FAQPage",
      "mainEntity": [ ... ]
    }
  ]
}
</script>
```

Place this block inside `<head>` or at the very end of `<body>` before `</body>`.

---

## Meta Tags to Pair with JSON-LD

These `<meta>` tags reinforce entity signals for LLM crawlers:

```html
<!-- Required -->
<meta name="description" content="[ProductName] by [Company], [category] for [user]. [One-line value prop].">

<!-- Open Graph (used by AI crawlers too) -->
<meta property="og:title" content="[ProductName]: [Short tagline with category]">
<meta property="og:description" content="[Same as meta description]">
<meta property="og:type" content="website">
<meta property="og:url" content="https://[yourproduct.com]">
<meta property="og:image" content="https://[yourproduct.com/og-image.png]">

<!-- Product-specific -->
<meta name="application-name" content="[ProductName]">
<meta name="keywords" content="[category], [use case], [target user], [top feature]">
```

**Note on keywords**: While traditional search engines de-weight keywords, several AI crawlers (including Perplexity's) still use them for topic classification.
Include 4-8 specific, non-spammy keywords.