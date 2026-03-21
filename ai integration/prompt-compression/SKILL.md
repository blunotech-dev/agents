---
name: prompt-compression
description: Compress prompts to reduce token count without degrading output quality. Use this skill when the user wants to shorten a system prompt, user prompt, or instruction block; when they say a prompt is "too long", "hitting token limits", or "too expensive to run"; when they want to audit a prompt for bloat; or when they want to make a prompt leaner before scaling it to production. Trigger even if the user says "clean up my prompt" or "tighten this up" — compression is almost always part of what they mean.
category: "Ai Integration"
---

# Prompt Compression

Reduces prompt token count without losing effectiveness. Covers semantic deduplication, example pruning, instruction consolidation, and format directive compression. Critically: distinguishes between safe cuts and cuts that silently degrade output.

---

## Phase 1 — Audit Before Cutting

Read the full prompt. Map it into buckets before touching anything:

- **Role/persona block** — who the model is
- **Behavioral constraints** — what it must/must not do
- **Task instructions** — what to actually produce
- **Format directives** — how to structure output
- **Examples** — few-shot demonstrations
- **Context/background** — grounding information

Compression failures almost always come from cutting across buckets without noticing. A constraint buried in an example isn't redundant with the same constraint stated in instructions — it's reinforcement. Flag these before cutting.

---

## Phase 2 — Semantic Deduplication (Non-Obvious)

Surface-level repetition is easy to spot. The hard cases:

**Same instruction, different surface forms.** Look for instructions that produce identical model behavior even though they're worded differently. Example: "Be concise" + "Avoid unnecessary elaboration" + "Keep responses short" — these are one instruction. Keep the most precise version; the others are noise.

**Constraint restated as example preamble.** Many prompts say "Always cite sources" in instructions, then open every example with "Here I cite my sources because..." — the example preamble is redundant. Strip example preambles that just narrate compliance with explicit instructions.

**Role description that duplicates task description.** "You are a legal document reviewer who carefully reads contracts for risk" and "Your task is to read contracts and identify risk clauses" are one sentence. Merge them. The role line should add persona; if it's just restating the task, cut it.

**Hedging chains.** Prompts often stack modifiers: "Please carefully and thoroughly and completely analyze..." — this is one adverb. Pick the strongest one.

---

## Phase 3 — Example Pruning

Examples are the highest-token, highest-value part of most prompts. Wrong cuts here are the most damaging.

**Keep if:**
- It demonstrates an edge case not covered by instructions
- Input/output format is non-obvious and the example makes it concrete
- It shows a failure mode to avoid (negative examples are high-value per token)
- The task has high output variance and the example anchors style/tone

**Cut if:**
- It demonstrates something already unambiguously stated in instructions (adds zero information)
- Multiple examples cover the same scenario at different complexity levels — keep only the hardest one
- The example is perfectly "normal" with no edge case signal — these are the safest to cut first

**Compress rather than cut:**
- Trim example inputs to the minimum that makes the output meaningful — you don't need a full paragraph of input if two sentences carry the signal
- Remove example chain-of-thought unless the task explicitly requires CoT output — showing reasoning in examples inflates tokens without improving output quality on straightforward tasks

---

## Phase 4 — Instruction Consolidation

Scattered constraints bloat prompts. Patterns to collapse:

**Condition clusters → single rule.** If you see "If the user asks X, do Y. If the user asks X in context Z, still do Y. If X is implied, also do Y" — this is "Always do Y when X." Collapse it.

**Negative + positive restatement.** "Don't be vague. Be specific." is one instruction: "Be specific." The negative form adds nothing when the positive form is present. Exception: keep the negative form when the failure mode is common enough that the explicit prohibition matters (e.g., "Don't make up citations" is worth keeping even alongside "Only cite real sources" — the stakes justify the redundancy).

**Ordered lists that aren't actually ordered.** If a numbered list of instructions has no dependency between steps, it's a bulleted list using extra tokens. Convert to bullets or prose unless order genuinely matters to the task.

**Merge micro-constraints into governing rules.** A list of 8 specific dos/don'ts often collapses to 1-2 governing principles. Example: "Don't use passive voice. Don't use jargon. Don't use sentences over 20 words. Don't use hedging language." → "Write in plain, direct, active sentences under 20 words." This is both shorter and more generalizable.

---

## Phase 5 — Format Directive Compression

Format blocks are the most over-specified part of most prompts.

**Cut structural boilerplate.** "Begin your response with a brief introduction. Then provide the main content. End with a summary." — this is the default structure for any coherent response. It adds zero constraint. Cut it.

**Replace output templates with examples.** If the prompt contains a detailed template (field names, placeholder syntax, required sections), and there's also an example showing that template filled in, the template is redundant. Keep the example; cut the template.

**Collapse nested format instructions.** "Use markdown. Use headers for each section. Use bold for key terms. Use bullet points for lists." → "Use markdown with headers, bold key terms, and bullets for lists." One sentence.

**Length instructions that contradict task scope.** If the task is "summarize a 10-page document in 3 sentences" the length instruction is already in the task. A separate "Be brief" directive adds nothing.

---

## Phase 6 — What NOT to Compress

These are safe-looking cuts that silently break prompts:

**Repetition that spans a long context window.** If the prompt is long and a constraint appears both early and late, the late restatement is load-bearing for attention. Don't remove it just because it's technically redundant.

**Negative examples.** "Here's a bad response and why it's wrong" examples are dense signal. They feel cuttable because they're "just" showing failure — but they're often what separates correct behavior from the modal wrong behavior.

**Constraints that feel obvious but cover common model failures.** "Don't apologize excessively" seems obvious but models do this by default. "Don't start your response by restating the question" seems obvious but models do this constantly. If a constraint is in the prompt, assume it's there because the author observed the failure. Flag for the user before cutting, don't cut automatically.

**Persona/tone details.** These feel soft and cuttable but heavily influence output register. "Casual but precise" vs nothing can mean the difference between a response that fits the product and one that doesn't.

---

## Output Format

Return:

1. **Compression summary** — original token estimate, compressed token estimate, % reduction
2. **Cuts made** — brief log of what was removed and why (one line each)
3. **Flags** — anything that looked cuttable but was preserved because it's load-bearing; let the user decide
4. **Compressed prompt** — the full result, ready to use

If compression would drop below ~30% of original length with meaningful content still present, flag it: heavy compression past a certain point trades quality for tokens in ways that are hard to predict without testing.