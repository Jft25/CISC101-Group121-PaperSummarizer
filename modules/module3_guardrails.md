> Change Log (2025-12-05)  
> - Added `evidence_mode` flag with `"strict"` behavior to enforce source-only claims.  
> - Introduced standardized warning messages for missing, empty, and very short sections (<50 words).  
> - Clarified how guardrails interact with section summaries, evidence checks, and final Checks & Warnings output.

## Purpose

Enforce safety, anti-hallucination behavior, and structural integrity across all section summaries.  
This module ensures that outputs obey PS2 constraints, respect evidence discipline, and surface clear warnings to the user.

## Inputs

- `section_summaries`, `evidence_notes`, and `flags` from Module 2.  
- `canonical_sections` and `section_order` from Module 1.  
- `config` object including:
  - Word limit settings.  
  - Shared acronym/terminology registry.  
  - **`evidence_mode` setting**, which may be:
    - `"strict"` (recommended default), or  
    - other modes (e.g., `"normal"`) if extended in future work.

## Outputs

- `validated_summaries`: section summaries cleaned and enforced by guardrails.  
- `consolidated_flags`: combined list of warnings and issues per section.  
- `updated_registry`: final terminology/acronym registry.  
- Standardized warning messages to be consumed by:
  - Section-by-Section Summary Table (Issues/Flags column).  
  - Checks & Warnings section in final output.

## Evidence Mode: `evidence_mode = "strict"`

When `evidence_mode` is set to `"strict"`, the module must:

1. **Only include claims, equations, and results that appear in the provided text.**
2. **If there is not enough information to summarize a section reliably**, replace or augment the summary with the message:

   > `"The source text does not provide enough detail to summarize this section in strict evidence mode."`

3. Remove or rephrase any content that cannot be clearly traced back to the source, and add a flag:
   - `strict_evidence_insufficient`.

If `evidence_mode` is less strict (e.g., `"normal"`), other modes may allow slightly broader paraphrases, but they must still avoid clear hallucinations.  
The default behavior for this project should assume `"strict"` unless otherwise specified.

## Standardized Section Warning Messages

This module defines standard messages for problematic sections. These must be used consistently in:

- The **Issues/Flags** column of the Section-by-Section Summary Table.  
- The **Checks & Warnings** section.

Warnings:

- **Missing section** (no text and not supplied):  
  - Message: `"Section skipped: no usable text was provided."`  
  - Flag: `missing_section`.

- **Empty section** (present in structure but effectively no content):  
  - Message: `"Section skipped: section was present but contained no usable text."`  
  - Flag: `empty_section`.

- **Very short section (< 50 words)**:  
  - Message: `"Section very short: summary may be incomplete in strict evidence mode."`  
  - Flag: `section_too_short`.

These warnings must be attached to both the per-section flags and the consolidated warnings list.

## Responsibilities

1. **Validate Sections**

   The function `ValidateSections(canonical_sections, section_summaries)` must:

   - Identify sections that are:
     - Missing entirely from provided text.
     - Present but empty.
     - Present and < 50 words of usable content.
   - Attach the appropriate standardized warning message and flag to each section.

2. **Prevent Hallucination**

   The function `PreventHallucination(summary, section_text, evidence_mode)` must:

   - For `evidence_mode = "strict"`:
     - Remove or rewrite any claim that cannot be traced to the source text.
     - Replace missing content with `"Not stated"` or the strict evidence message above when detail is lacking.
     - Add the flag `strict_evidence_insufficient` when summaries are weakened by missing detail.
   - Ensure that no new entities, datasets, or results are introduced that are not mentioned in the source.

3. **Apply Chunking Policy**

   The function `ApplyChunkingPolicy(section_text)` must:

   - Confirm that long sections were summarized by sub-chunk first, then aggregated.  
   - Ensure that aggregation does not introduce new claims; it may only compress or filter existing supported statements.

4. **Enforce Terminology Consistency**

   The function `EnforceTerminology(summary, registry)` must:

   - Ensure acronyms are expanded only on first use and reused thereafter.  
   - Keep terminology consistent with prior sections and the glossary.  
   - Update `registry` for downstream modules.

5. **Preserve Order**

   The function `PreserveOrder(section_order, outputs)` must:

   - Guarantee that all validated outputs (summaries, flags, warnings) are ordered exactly according to `section_order`.  
   - Prevent any re-sorting by score, length, or other criteria.

## Data Structures

Each validated section record should include:

```json
{
  "section_name": "<canonical name>",
  "summary_text": "<final summary after guardrails>",
  "warnings": [
    "Section skipped: no usable text was provided.",
    "Section very short: summary may be incomplete in strict evidence mode."
  ],
  "flags": [
    "missing_section",
    "section_too_short",
    "strict_evidence_insufficient"
  ]
}
consolidated_flags and the final Checks & Warnings output must be built by aggregating these per-section warnings and flags.

Key Functions
ValidateSections(canonical_sections, section_summaries)
Flags missing, empty, and very short sections and attaches standardized warning messages.

PreventHallucination(summary, section_text, evidence_mode)
Enforces strict evidence rules; strips unsupported claims; inserts "Not stated" or the strict evidence message when needed.

ApplyChunkingPolicy(section_text)
Confirms proper sub-chunk summarization and faithful aggregation.

EnforceTerminology(summary, registry)
Stabilizes acronyms and terminology; updates registry.

PreserveOrder(section_order, outputs)
Produces a final ordered list of validated summaries and flags.
