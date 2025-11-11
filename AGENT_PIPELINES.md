# Stagehand Agent Pipelines Documentation

This document provides comprehensive documentation of all agent execution pipelines and workflows in the Stagehand browser automation system. Each pipeline shows the step-by-step flow, prompt usage, data transformations, and how components interact.

## Table of Contents

1. [Extract Pipeline](#extract-pipeline)
2. [Observe Pipeline](#observe-pipeline)
3. [Act Pipeline](#act-pipeline)
4. [V3 Agent Execution Pipeline](#v3-agent-execution-pipeline)
5. [Computer Use Agent Pipelines](#computer-use-agent-pipelines)
   - [Anthropic CUA](#anthropic-cua-pipeline)
   - [OpenAI CUA](#openai-cua-pipeline)
   - [Google CUA](#google-cua-pipeline)
6. [Evaluator Pipeline](#evaluator-pipeline)

---

## Extract Pipeline

**Purpose:** Extract structured or unstructured data from web pages using AI to parse DOM elements.

**Entry Point:** `ExtractHandler.extract()` in `packages/core/lib/v3/handlers/extractHandler.ts`

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         EXTRACT PIPELINE                             │
└─────────────────────────────────────────────────────────────────────┘

User Request: extract(instruction, schema)
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Capture Hybrid Snapshot                                     │
│ Location: extractHandler.ts:98-123                                  │
│                                                                      │
│ Browser Page                                                         │
│      │                                                               │
│      ├──► Enable CDP Domains (DOM, Accessibility)                   │
│      │                                                               │
│      ├──► Build DOM Tree (depth: -1, pierce: true)                  │
│      │    → DOM.getDocument() → Full DOM structure                  │
│      │                                                               │
│      ├──► Build Accessibility Tree                                  │
│      │    → Accessibility.getFullAXTree() → A11y nodes              │
│      │                                                               │
│      ├──► Merge Trees (Hybrid Snapshot)                             │
│      │    → Combine DOM structure with A11y role annotations        │
│      │    → Extract XPath map: backendNodeId → XPath                │
│      │    → Extract URL map: backendNodeId → URL                    │
│      │                                                               │
│      └──► Output: {                                                 │
│              combinedTree: string,         // Human-readable tree   │
│              combinedXpathMap: Map,        // ID → XPath            │
│              combinedUrlMap: Map           // ID → URL              │
│           }                                                          │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Transform Schema (URL Handling)                             │
│ Location: extractHandler.ts:146-147                                 │
│                                                                      │
│ Schema (Zod)                                                         │
│      │                                                               │
│      ├──► Scan for z.string().url() fields                          │
│      │                                                               │
│      ├──► Replace with z.number() temporarily                        │
│      │    (URLs will be represented as numeric IDs in extraction)   │
│      │                                                               │
│      └──► Track paths to URL fields for later re-injection          │
│                                                                      │
│ Output: transformedSchema + urlFieldPaths                           │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: First LLM Call - Extract Data                               │
│ Location: extractHandler.ts:149-157 → lib/inference.ts:30-85        │
│                                                                      │
│ Prompt Construction (lib/prompt.ts:19-82):                          │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ SYSTEM PROMPT                            │                        │
│ │ buildExtractSystemPrompt()               │                        │
│ │                                           │                        │
│ │ "You are extracting content on behalf    │                        │
│ │  of a user. If a user asks you to        │                        │
│ │  extract a 'list' of information, or     │                        │
│ │  'all' information, YOU MUST EXTRACT     │                        │
│ │  ALL OF THE INFORMATION THAT THE USER    │                        │
│ │  REQUESTS.                                │                        │
│ │                                           │                        │
│ │  You will be given:                       │                        │
│ │  1. An instruction                        │                        │
│ │  2. A list of DOM elements to extract    │                        │
│ │      from.                                │                        │
│ │                                           │                        │
│ │  Print the exact text from the DOM       │                        │
│ │  elements with all symbols, characters,  │                        │
│ │  and endlines as is. Print null or an    │                        │
│ │  empty string if no new information is   │                        │
│ │  found.                                   │                        │
│ │                                           │                        │
│ │  [For Anthropic: Use print_extracted_    │                        │
│ │   data tool]                              │                        │
│ │                                           │                        │
│ │  If a user is attempting to extract      │                        │
│ │  links or URLs, you MUST respond with    │                        │
│ │  ONLY the IDs of the link elements."     │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ USER PROMPT                              │                        │
│ │ buildExtractUserPrompt()                 │                        │
│ │                                           │                        │
│ │ "Instruction: ${instruction}             │                        │
│ │  DOM: ${domElements}"                    │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ LLM Call: llmClient.createChatCompletion()                          │
│      │                                                               │
│      ├──► Model: user-specified (e.g., gpt-4o, claude-3.5)          │
│      ├──► Temperature: 0.1 (deterministic)                          │
│      ├──► Response Format: Structured output (Zod schema)           │
│      │                                                               │
│      └──► Output: Extracted data matching schema                    │
│                   (URLs as numeric IDs at this stage)               │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Second LLM Call - Evaluate Completion                       │
│ Location: inference.ts:130-174                                      │
│                                                                      │
│ Prompt Construction (lib/prompt.ts:84-108):                         │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ SYSTEM PROMPT                            │                        │
│ │ buildMetadataSystemPrompt()              │                        │
│ │                                           │                        │
│ │ "You are an AI assistant tasked with     │                        │
│ │  evaluating the progress and completion  │                        │
│ │  status of an extraction task.           │                        │
│ │  Analyze the extraction response and     │                        │
│ │  determine if the task is completed or   │                        │
│ │  if more information is needed.          │                        │
│ │                                           │                        │
│ │  Strictly abide by the following         │                        │
│ │  criteria:                                │                        │
│ │  1. Once the instruction has been        │                        │
│ │     satisfied by the current extraction  │                        │
│ │     response, ALWAYS set completion      │                        │
│ │     status to true and stop processing,  │                        │
│ │     regardless of remaining chunks.      │                        │
│ │  2. Only set completion status to false  │                        │
│ │     if BOTH of these conditions are      │                        │
│ │     true:                                 │                        │
│ │     - The instruction has not been       │                        │
│ │       satisfied yet                       │                        │
│ │     - There are still chunks left to     │                        │
│ │       process (chunksTotal >             │                        │
│ │       chunksSeen)"                        │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ USER PROMPT                              │                        │
│ │ buildMetadataPrompt()                    │                        │
│ │                                           │                        │
│ │ "Instruction: ${instruction}             │                        │
│ │  Extracted content:                      │                        │
│ │  ${JSON.stringify(                       │                        │
│ │    extractionResponse, null, 2           │                        │
│ │  )}"                                      │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ LLM Call: Uses metadata schema                                      │
│      │                                                               │
│      ├──► Schema: {                                                 │
│      │      progress: z.string(),                                   │
│      │      completed: z.boolean()                                  │
│      │    }                                                          │
│      │                                                               │
│      └──► Output: Completion metadata                               │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Re-inject URLs                                              │
│ Location: extractHandler.ts:198-204                                 │
│                                                                      │
│ For each URL field that was converted to numeric ID:                │
│      │                                                               │
│      ├──► Take numeric ID from extracted data                       │
│      │                                                               │
│      ├──► Look up ID in combinedUrlMap                              │
│      │    → combinedUrlMap[numericId] = "https://..."              │
│      │                                                               │
│      └──► Replace numeric ID with actual URL string                 │
│                                                                      │
│ Output: Final extracted data with URLs as strings                   │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ RETURN TO USER                                                       │
│                                                                      │
│ {                                                                    │
│   // Extracted data matching schema                                 │
│   // with URLs properly formatted                                   │
│   ...extractedFields                                                 │
│ }                                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Transformations

1. **Browser Page → Hybrid Snapshot**
   - DOM tree + Accessibility tree → Merged text representation
   - XPath mapping for element location
   - URL mapping for link extraction

2. **Schema → Transformed Schema**
   - `z.string().url()` fields → `z.number()` temporarily
   - Tracks transformation paths for reversal

3. **Hybrid Tree + Instruction → Extracted Data**
   - LLM parses tree structure using instruction
   - Returns data matching schema shape
   - URLs represented as numeric IDs

4. **Extracted Data + Metadata**
   - Second LLM call evaluates completeness
   - Determines if more chunks needed

5. **Numeric IDs → URLs**
   - Final transformation back to URL strings
   - Uses combinedUrlMap for lookup

### Key Files

- **Handler:** `packages/core/lib/v3/handlers/extractHandler.ts`
- **Prompts:** `packages/core/lib/prompt.ts` (lines 19-108)
- **Inference:** `packages/core/lib/inference.ts` (lines 30-174)
- **Snapshot:** `packages/core/lib/understudy/a11y/snapshot.ts`

### Usage Example

```typescript
const result = await stagehand.extract(
  "extract all product names and prices",
  z.object({
    products: z.array(z.object({
      name: z.string(),
      price: z.number()
    }))
  })
);

// Result: { products: [{ name: "...", price: 29.99 }, ...] }
```

