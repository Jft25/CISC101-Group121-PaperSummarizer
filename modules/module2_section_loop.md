# Module 2: Section Loop

> Change Log (2025-12-05)  
> - Added `summary_level` variable (`"short"` or `"detailed"`) to control per-section summary detail.  
> - Defined conditional behavior: short mode → 1–2 sentences; detailed mode → paragraph + 3–5 key-point bullets.  
> - Clarified how the section loop formats output for each summary level while keeping PS2 word limits and evidence rules.

## Purpose

Summarize each section of the paper in canonical order, respecting PS2 constraints and applying evidence checks.  
This module now supports configurable **summary levels** for each section via the `summary_level` variable.

- `summary_level = "short"` → 1–2 sentence compact summary per section.  
- `summary_level = "detailed"` → a short paragraph **plus** a bullet list of 3–5 key points per section.

## Inputs

- `canonical_sections` and associated section text (from Module 1).  
- `section_order` (array of canonical section names).  
- `config` object, including:
  - Per-section word limit (≤120 words).  
  - Global limit reference.  
  - Audience flag (expert/lay).  
  - Acronym/terminology registry.  
  - **`summary_level` setting**: `"short"` or `"detailed"` (default: `"short"` if unspecified).

## Outputs

- `section_summaries`: structured summaries per section, formatted according to `summary_level`.  
- `evidence_notes`: per-section notes on traceability and source alignment.  
- `flags`: per-section issues/flags for Guardrails (e.g., unsupported claims, insufficiency).

## Responsibilities

For each section in `section_order`, the loop must:

1. **Extract and chunk text**
   - Use `ChunkSectionText(section_text)` to:
     - Split long sections into sub-chunks by heading, paragraph, or token window.
     - Preserve references so evidence can be traced back.

2. **Summarize with word limits and summary level**

   - Call `SummarizeSectionWithLevel(chunks, summary_level, limit=120)` which:
     - Always respects the ≤120-word limit for the **entire section summary** (paragraph + bullets combined).
     - Behavior by mode:
       
       - If `summary_level = "short"`:
         - Output: **1–2 sentences** (compact paragraph) describing the main idea of the section.
         - No bullet list.
       
       - If `summary_level = "detailed"`:
         - Output:
           - A **short paragraph** (2–4 sentences) capturing the main idea.
           - A **bullet list of 3–5 key points** highlighting datasets, methods, results, or other core facts.
         - Combined paragraph + bullets must remain within 120 words total.

   - Summaries must:
     - Be grounded strictly in section text.
     - Use terminology consistent with the acronym registry.
     - Avoid speculation.

3. **Run evidence checks**

   - Call `EvidenceCheck(summary, section_text)` to:
     - Verify that each claim in the summary can be traced back to the original section text.
     - Mark ambiguous or unsupported content as `"Not stated"` rather than inventing details.
     - Generate `evidence_notes` describing:
       - Whether all key claims are supported.
       - Any areas where detail is missing or incomplete.
     - Add flags such as:
       - `unsupported_claims`
       - `insufficient_detail`
       - `section_too_short_for_full_summary` (for <50-word sections, to be consumed by Module 3).

4. **Manage acronyms and terminology**

   - Call `AcronymManager(summary, registry)` to:
     - Ensure each acronym is expanded only on first use (e.g., “Transformer Neural Network (TNN)”).
     - Enforce reuse of the acronym thereafter.
     - Update the shared acronym/terminology registry for use by Rendering and Glossary modules.

5. **Produce structured section output**

   For each section, produce a structured record like:

   ```json
   {
     "section_name": "<canonical name>",
     "summary_level": "<short|detailed>",
     "summary_paragraph": "<main paragraph text>",
     "summary_bullets": ["<bullet 1>", "<bullet 2>", "..."],  // empty list if short mode
     "evidence_notes": "<short note on traceability>",
     "flags": ["unsupported_claims", "section_too_short_for_full_summary"]
   }
   In summary_level = "short" mode:

summary_bullets MUST be an empty list.

In summary_level = "detailed" mode:

summary_bullets MUST contain 3–5 items when enough evidence is available.

Functions
ChunkSectionText(section_text)

Segments long sections into manageable parts.

Preserves ordering and references for evidence mapping.

SummarizeSectionWithLevel(chunks, summary_level, limit=120)

Generates either:

1–2 sentence compact summary ("short"), or

A paragraph plus 3–5 bullets ("detailed"),
while staying within limit words overall.

EvidenceCheck(summary, section_text)

Verifies claims against source text.

Adds "Not stated" where the source is silent.

Produces evidence notes and flags.

AcronymManager(summary, registry)

Handles first-use expansion and subsequent reuse.

Updates shared registry.
