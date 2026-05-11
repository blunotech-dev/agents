---
name: mvp-one-page-copywriter
description: Forces AI  to write conversion-focused copy using the PAS (Problem-Agitate-Solve) framework before generating frontend code for landing pages, MVPs, waitlist pages, launch pages, or product one-pagers. Trigger whenever the user wants to build a product-focused website or homepage, even if they directly ask for the code first.
---

# MVP One-Page Copywriter

This skill ensures that every MVP landing page or product one-pager has sharp, persuasive
copy BEFORE any code is written. Beautiful UI with weak messaging fails. This skill fixes that.

**The rule:** Copy first. Code second. Always.

---

## Phase 1: Extract the Product Context

Before writing a single word of copy or code, ask the user (or infer from context) the
following. If the user has already provided enough detail, skip the question and proceed:

1. **What does the product do?** One sentence, no jargon.
2. **Who is the target user?** Be specific (e.g., "freelance designers in Southeast Asia",
   not "everyone").
3. **What is the primary pain the product solves?** What is the user doing right now without
   this product, and why does that suck?
4. **What is the one action you want the visitor to take?** (Sign up, join waitlist, buy,
   book a call, etc.)

If context is thin, make reasonable assumptions and state them clearly so the user can
correct you. Never block on clarification if you can make a smart educated guess.

---

## Phase 2: Write the Copy Using the PAS Framework

Write the copy sections in order before producing any code. Format this as a clearly labeled
**Copy Brief** block so the user can review and request changes.

### Section 1: Hero

The hero is the first thing a visitor sees. It must answer "what is this and why should I
care" within three seconds.

Write:
- **Headline** (6-12 words): Outcome-focused, not feature-focused. What does the user
  *get* or *become*? Avoid generic openers like "Introducing..." or "Welcome to...".
- **Sub-headline** (1-2 sentences): Expand the headline. Name the target user, the problem,
  and the mechanism of the solution.
- **CTA Button Text** (2-5 words): Action verb + specific outcome. Not "Submit" or "Click
  here." Example: "Start for Free", "Join the Waitlist", "Get My Dashboard".

### Section 2: Pain Points (Agitate)

Make the problem real and visceral. Use 2-4 short pain statements that a target user would
read and feel "yes, that's exactly me." These can be formatted as bullet points or as a
short paragraph.

Rules:
- Use the user's language, not product language.
- Each point should describe a consequence, not just a problem. (Not "X is hard." But "X is
  hard, so you end up doing Y, which costs you Z.")
- This section should create emotional recognition, not just logical agreement.

### Section 3: Solution

Explain what the product does and why it works. Structure:
- **One-liner mechanism**: "ProductName does X by doing Y, so you get Z."
- **2-4 feature/benefit pairs**: Each listed as "[Feature] so you can [Benefit]." Never
  list a feature without its user benefit.
- **Social proof placeholder** (if applicable): Write a placeholder line like
  `[Testimonial from early user: outcome they achieved]` so the user knows to add it.

### Section 4: CTA Repeat

One final push at the bottom. Restate the core benefit and repeat the CTA. Can include a
micro-trust element (e.g., "No credit card required", "Free to start", "Cancel anytime").

---

## Phase 3: Copy Review Gate

After presenting the Copy Brief, explicitly pause and ask:

> "Does this copy capture your product well? Approve it or tell me what to change before I
> build the UI."

Do NOT proceed to code until the user approves, or explicitly says "looks good, go ahead"
or equivalent. If the user makes changes, revise the copy and confirm once more.

---

## Phase 4: Build the Frontend

Only after copy approval, proceed to build the landing page. Hand off to the
`frontend-design` skill conventions:

- **Populate the UI with the exact approved copy.** Never use lorem ipsum or placeholder
  text where real copy exists.
- Structure the page in the exact order the copy was written: Hero, Pain, Solution, CTA.
- Design choices should reinforce the tone implied by the copy. A B2B SaaS tool for
  accountants gets a different aesthetic than a Gen Z consumer app.
- All CTA buttons use the exact button text from the copy brief.
- If the user has not specified a visual style, make a confident aesthetic choice and name
  it briefly (e.g., "Going with a clean, high-contrast editorial style to match the
  professional audience.")

---

## Quality Checklist (Run Before Finalizing)

Before presenting the final output, verify:

- [ ] Hero headline is outcome-focused, not feature-focused
- [ ] Sub-headline names a specific user type
- [ ] Pain points use "you" language, not third-person
- [ ] Every feature listed has a corresponding benefit
- [ ] CTA button text is an action verb + outcome
- [ ] No lorem ipsum or placeholder text in the final build
- [ ] Page order is: Hero > Pain > Solution > CTA

---

## Example Copy Brief (for Reference)

**Product:** A tool that helps freelancers track unpaid invoices.
**Target user:** Freelance designers and developers who chase payments manually.

---

**Copy Brief**

**HERO**
- Headline: "Get Paid Faster, Without the Awkward Follow-Ups"
- Sub-headline: "InvoiceNow helps freelancers automatically track overdue payments and send
  professional reminders, so you spend your time building, not chasing."
- CTA: "Start Tracking Free"

**PAIN POINTS**
You finished the project. You sent the invoice. Now you're waiting.
- You check your bank account every morning hoping it magically appeared.
- You don't want to send a third reminder because it feels desperate, but the invoice is
  60 days overdue.
- Your accountant is asking you for numbers you don't have because half your invoices live
  in your email drafts.

**SOLUTION**
InvoiceNow automatically monitors your invoices and sends timed, professional follow-ups on
your behalf, so you get paid without the awkwardness.
- **Auto-reminders** so you never have to manually follow up again.
- **Payment status dashboard** so you always know exactly what you're owed and when.
- **Client portal** so clients can pay in one click, removing every excuse for delay.

`[Testimonial: "Got paid 3 weeks earlier on my last project just by turning on reminders." — freelance dev, Mumbai]`

**CTA REPEAT**
Stop chasing payments. Start getting paid.
[Start Tracking Free] — No credit card required.

---

*This brief is a reference. Always write fresh copy tailored to the user's actual product.*