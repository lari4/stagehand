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


---

## Observe Pipeline

**Purpose:** Find multiple elements on a page based on observation intent without performing actions on them.

**Entry Point:** `ObserveHandler.observe()` in `packages/core/lib/v3/handlers/observeHandler.ts`

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OBSERVE PIPELINE                              │
└─────────────────────────────────────────────────────────────────────┘

User Request: observe(instruction)
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Capture Hybrid Snapshot                                     │
│ Location: observeHandler.ts:81-84                                   │
│                                                                      │
│ Same process as Extract Pipeline:                                   │
│      │                                                               │
│      ├──► Enable CDP Domains (DOM, Accessibility)                   │
│      ├──► Build DOM Tree + Accessibility Tree                       │
│      ├──► Merge Trees into Hybrid Snapshot                          │
│      │                                                               │
│      └──► Output: {                                                 │
│              combinedTree: string,         // Human-readable tree   │
│              combinedXpathMap: Map         // ID → XPath            │
│           }                                                          │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: LLM Call - Find Elements                                    │
│ Location: observeHandler.ts:96-103 → lib/inference.ts:223-356       │
│                                                                      │
│ Prompt Construction (lib/prompt.ts:111-141):                        │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ SYSTEM PROMPT                            │                        │
│ │ buildObserveSystemPrompt()               │                        │
│ │                                           │                        │
│ │ "You are helping the user automate the   │                        │
│ │  browser by finding elements based on    │                        │
│ │  what the user wants to observe in the   │                        │
│ │  page.                                    │                        │
│ │                                           │                        │
│ │  You will be given:                       │                        │
│ │  1. a instruction of elements to         │                        │
│ │     observe                               │                        │
│ │  2. a hierarchical accessibility tree    │                        │
│ │     showing the semantic structure of    │                        │
│ │     the page. The tree is a hybrid of    │                        │
│ │     the DOM and the accessibility tree.  │                        │
│ │                                           │                        │
│ │  Return an array of elements that match  │                        │
│ │  the instruction if they exist,          │                        │
│ │  otherwise return an empty array."       │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ USER PROMPT                              │                        │
│ │ buildObserveUserMessage()                │                        │
│ │                                           │                        │
│ │ "instruction: ${instruction}             │                        │
│ │  Accessibility Tree: \n${domElements}"  │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ LLM Call: llmClient.createChatCompletion()                          │
│      │                                                               │
│      ├──► Temperature: 0.1 (deterministic)                          │
│      ├──► Response Format: Structured output (observe schema)       │
│      │                                                               │
│      │    Schema: {                                                 │
│      │      elements: z.array(z.object({                            │
│      │        elementId: z.string(),      // "ordinal-nodeId"       │
│      │        description: z.string(),                              │
│      │        method: z.string(),         // Playwright method      │
│      │        arguments: z.array(z.string())                        │
│      │      }))                                                      │
│      │    }                                                          │
│      │                                                               │
│      └──► Output: Array of element objects with IDs                 │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Map Element IDs to Selectors                                │
│ Location: observeHandler.ts:120-149                                 │
│                                                                      │
│ For each element in LLM response:                                   │
│      │                                                               │
│      ├──► Take elementId (format: "ordinal-backendNodeId")          │
│      │                                                               │
│      ├──► Look up in combinedXpathMap[elementId]                    │
│      │    → combinedXpathMap["0-42"] = "/html/body/div[1]/button"  │
│      │                                                               │
│      ├──► Trim trailing text nodes from XPath                       │
│      │    (Remove ::text()[1] suffixes for cleaner selectors)       │
│      │                                                               │
│      ├──► Format as Action object:                                  │
│      │    {                                                          │
│      │      description: "Login button",                            │
│      │      method: "click",                                         │
│      │      arguments: [],                                           │
│      │      selector: "xpath=/html/body/div[1]/button"             │
│      │    }                                                          │
│      │                                                               │
│      └──► Add to results array                                      │
│                                                                      │
│ Output: Array of Action objects ready for execution                 │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ RETURN TO USER                                                       │
│                                                                      │
│ {                                                                    │
│   elements: [                                                        │
│     {                                                                │
│       description: "Login button",                                  │
│       method: "click",                                               │
│       arguments: [],                                                 │
│       selector: "xpath=/html/body/div[1]/button"                   │
│     },                                                               │
│     {                                                                │
│       description: "Username input",                                │
│       method: "fill",                                                │
│       arguments: [],                                                 │
│       selector: "xpath=/html/body/div[1]/input[1]"                 │
│     },                                                               │
│     ...                                                              │
│   ]                                                                  │
│ }                                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Transformations

1. **Browser Page → Hybrid Snapshot**
   - DOM tree + Accessibility tree → Merged text representation
   - XPath mapping for element location

2. **Hybrid Tree + Instruction → Element Array**
   - LLM identifies all matching elements from tree
   - Returns array with element IDs and descriptions

3. **EncodedId → XPath**
   - Maps element IDs to XPath selectors
   - Format: `"ordinal-backendNodeId"` → `"/html/body/..."`

4. **XPath → Action Objects**
   - Formats as executable action objects
   - Adds `xpath=` prefix for Playwright
   - Includes method and arguments

### Key Differences from Extract

- **Multiple Results:** Returns array of elements vs single extraction
- **No Schema Required:** Just an instruction, no structured schema
- **Selector Generation:** Focuses on generating Playwright selectors
- **No Metadata Call:** Single LLM call (no completion evaluation)

### Key Files

- **Handler:** `packages/core/lib/v3/handlers/observeHandler.ts`
- **Prompts:** `packages/core/lib/prompt.ts` (lines 111-141)
- **Inference:** `packages/core/lib/inference.ts` (lines 223-356)

### Usage Example

```typescript
const result = await stagehand.observe("find all buttons on the page");

// Result: {
//   elements: [
//     { 
//       description: "Submit button", 
//       method: "click", 
//       selector: "xpath=..." 
//     },
//     { 
//       description: "Cancel button", 
//       method: "click", 
//       selector: "xpath=..." 
//     },
//     ...
//   ]
// }
```


---

## Act Pipeline

**Purpose:** Find a single element and perform an action on it (click, type, etc.). Supports complex scenarios like two-step dropdowns and self-healing when actions fail.

**Entry Point:** `ActHandler.act()` in `packages/core/lib/v3/handlers/actHandler.ts`

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          ACT PIPELINE                                │
└─────────────────────────────────────────────────────────────────────┘

User Request: act(instruction)
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Wait for Stability                                          │
│ Location: actHandler.ts:75-78                                       │
│                                                                      │
│      ├──► Wait for DOM mutations to stop                            │
│      └──► Wait for network requests to complete                     │
│                                                                      │
│ Ensures page is in stable state before capturing                    │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Capture Hybrid Snapshot                                     │
│ Location: actHandler.ts:79-86                                       │
│                                                                      │
│ Same as Extract/Observe pipelines                                   │
│      └──► Output: { combinedTree, combinedXpathMap }                │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: First LLM Call - Find Action Element                        │
│ Location: actHandler.ts:88-102 → lib/inference.ts:358-481           │
│                                                                      │
│ Prompt Construction (lib/prompt.ts:143-205):                        │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ SYSTEM PROMPT                            │                        │
│ │ buildActSystemPrompt()                   │                        │
│ │                                           │                        │
│ │ "You are helping the user automate the   │                        │
│ │  browser by finding elements based on    │                        │
│ │  what action the user wants to take on   │                        │
│ │  the page.                                │                        │
│ │                                           │                        │
│ │  You will be given:                       │                        │
│ │  1. a user defined instruction about     │                        │
│ │     what action to take                   │                        │
│ │  2. a hierarchical accessibility tree    │                        │
│ │     showing the semantic structure of    │                        │
│ │     the page.                             │                        │
│ │                                           │                        │
│ │  Return the element that matches the     │                        │
│ │  instruction if it exists. Otherwise,    │                        │
│ │  return an empty object."                 │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ USER PROMPT                              │                        │
│ │ buildActPrompt()                         │                        │
│ │                                           │                        │
│ │ "Find the most relevant element to      │                        │
│ │  perform an action on given the          │                        │
│ │  following action: ${action}.            │                        │
│ │                                           │                        │
│ │  IF AND ONLY IF the action EXPLICITLY    │                        │
│ │  includes the word 'dropdown' and        │                        │
│ │  implies choosing/selecting an option    │                        │
│ │  from a dropdown, ignore the 'General    │                        │
│ │  Instructions' section, and follow the   │                        │
│ │  'Dropdown Specific Instructions'        │                        │
│ │  section carefully.                       │                        │
│ │                                           │                        │
│ │  General Instructions:                    │                        │
│ │  - Provide an action for this element    │                        │
│ │    such as ${supportedActions}           │                        │
│ │  - Remember that to users, buttons and   │                        │
│ │    links look the same                    │                        │
│ │  - If action is unrelated, return empty  │                        │
│ │    object                                 │                        │
│ │  - ONLY return one action                 │                        │
│ │  - For scroll positions: format as       │                        │
│ │    percentage ('50%', '75%')             │                        │
│ │  - For key press: use press method       │                        │
│ │    ('Enter', 'Space', 'a')               │                        │
│ │                                           │                        │
│ │  Dropdown Specific Instructions:          │                        │
│ │                                           │                        │
│ │  CASE 1: element is a 'select' element   │                        │
│ │    - use selectOptionFromDropdown        │                        │
│ │    - set argument to exact option text   │                        │
│ │    - set twoStep to false                 │                        │
│ │                                           │                        │
│ │  CASE 2: element is NOT a 'select':      │                        │
│ │    - choose node that most closely       │                        │
│ │      corresponds to instruction EVEN if  │                        │
│ │      it's a 'StaticText' element         │                        │
│ │    - choose 'click' method                │                        │
│ │    - set twoStep to true                  │                        │
│ │                                           │                        │
│ │  [Variable substitution instructions if  │                        │
│ │   variables provided]"                    │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ LLM Call:                                                            │
│      │                                                               │
│      ├──► Schema: {                                                 │
│      │      elementId: z.string(),                                  │
│      │      description: z.string(),                                │
│      │      method: z.string(),                                     │
│      │      arguments: z.array(z.string()),                         │
│      │      twoStep: z.boolean()    // For non-select dropdowns     │
│      │    }                                                          │
│      │                                                               │
│      └──► Output: Single element with action details                │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Execute First Action (with Self-Heal)                       │
│ Location: actHandler.ts:174-179, 375-478                            │
│                                                                      │
│ Map element ID to XPath → Execute action                            │
│      │                                                               │
│      ├──► Convert elementId to XPath using combinedXpathMap         │
│      │                                                               │
│      ├──► Call performUnderstudyMethod(method, xpath, args)         │
│      │                                                               │
│      ├──► IF ACTION SUCCEEDS:                                       │
│      │      └──► Continue to Step 5 (check if twoStep)              │
│      │                                                               │
│      └──► IF ACTION FAILS & selfHeal=true:                          │
│            ┌───────────────────────────────────┐                    │
│            │ SELF-HEALING MECHANISM            │                    │
│            │                                    │                    │
│            │ 1. Log error and start reprocess  │                    │
│            │                                    │                    │
│            │ 2. Take FRESH snapshot            │                    │
│            │    → New combinedTree & XpathMap  │                    │
│            │                                    │                    │
│            │ 3. Build instruction from         │                    │
│            │    method + description            │                    │
│            │    e.g., "click the Login button" │                    │
│            │                                    │                    │
│            │ 4. Call actInference() AGAIN      │                    │
│            │    with fresh snapshot             │                    │
│            │                                    │                    │
│            │ 5. Extract new selector from      │                    │
│            │    LLM response                    │                    │
│            │                                    │                    │
│            │ 6. RETRY                           │                    │
│            │    performUnderstudyMethod()       │                    │
│            │    with new selector               │                    │
│            │                                    │                    │
│            │ 7. Update metrics for healing     │                    │
│            │    attempt                         │                    │
│            └───────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Two-Step Handling (if twoStep=true)                         │
│ Location: actHandler.ts:182-301                                     │
│                                                                      │
│ Used for non-select dropdowns that need to be expanded first        │
│                                                                      │
│ ┌─────────────────────────────────────────────────────┐            │
│ │ 5a. Capture Second Snapshot                         │            │
│ │     After first action completes                     │            │
│ │     → New combinedTree2 & XpathMap2                 │            │
│ └─────────────────────────────────────────────────────┘            │
│      │                                                               │
│      ▼                                                               │
│ ┌─────────────────────────────────────────────────────┐            │
│ │ 5b. Diff Trees                                       │            │
│ │     diffCombinedTrees(tree1, tree2)                 │            │
│ │                                                       │            │
│ │     → Finds newly appeared elements                  │            │
│ │       (e.g., dropdown options that are now visible) │            │
│ │     → If no diff found, uses full second tree       │            │
│ └─────────────────────────────────────────────────────┘            │
│      │                                                               │
│      ▼                                                               │
│ ┌─────────────────────────────────────────────────────┐            │
│ │ 5c. Second LLM Call                                  │            │
│ │                                                       │            │
│ │     SYSTEM PROMPT: Same as first call                │            │
│ │                                                       │            │
│ │     USER PROMPT: buildStepTwoPrompt()                │            │
│ │                                                       │            │
│ │     "The original user action was:                   │            │
│ │      ${originalUserAction}.                          │            │
│ │                                                       │            │
│ │      You have just taken the following action       │            │
│ │      which completed step 1 of 2:                    │            │
│ │      ${previousAction}.                              │            │
│ │                                                       │            │
│ │      Now, you must find the most relevant           │            │
│ │      element to perform an action on in order       │            │
│ │      to complete step 2 of 2.                        │            │
│ │                                                       │            │
│ │      [Same general instructions as buildActPrompt]" │            │
│ │                                                       │            │
│ │     Available actions: Excludes                      │            │
│ │                        SELECT_OPTION_FROM_DROPDOWN   │            │
│ │                        (already used in step 1)      │            │
│ └─────────────────────────────────────────────────────┘            │
│      │                                                               │
│      ▼                                                               │
│ ┌─────────────────────────────────────────────────────┐            │
│ │ 5d. Execute Second Action                            │            │
│ │     With new element from opened dropdown            │            │
│ │     → Map to XPath → Execute action                  │            │
│ └─────────────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ RETURN TO USER                                                       │
│                                                                      │
│ {                                                                    │
│   success: true,                                                     │
│   message: "Action completed successfully",                         │
│   actionDescription: "clicked the Login button",                    │
│   actions: [                                                         │
│     {                                                                │
│       description: "Login button",                                  │
│       method: "click",                                               │
│       arguments: [],                                                 │
│       selector: "xpath=/html/body/div[1]/button"                   │
│     },                                                               │
│     // If two-step, second action here                              │
│   ]                                                                  │
│ }                                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Two-Step Action Example

**Scenario:** Select "Large" from a custom dropdown (not a `<select>` element)

```
User: act("select Large from the size dropdown")

Step 1:
  → LLM identifies dropdown trigger element
  → Sets twoStep=true
  → Clicks dropdown to expand it

Step 2:
  → Capture new snapshot (dropdown is now open)
  → Diff trees to find newly visible options
  → LLM identifies "Large" option in diff
  → Clicks "Large" option
  
Result: Both actions recorded in actions array
```

### Self-Healing Example

**Scenario:** Element moves or changes between snapshot and execution

```
Initial:
  → Snapshot captured at time T0
  → LLM identifies element at xpath1
  → Page dynamically updates (animation, lazy load, etc.)
  → Action FAILS at xpath1 (element not found/stale)

Self-Heal (if enabled):
  → Take FRESH snapshot at time T1
  → LLM re-identifies element with new state
  → New element at xpath2
  → RETRY action at xpath2
  → SUCCESS
```

### Data Transformations

1. **Browser Page → Hybrid Snapshot**
   - Waits for stability first
   - DOM + A11y → Text tree + XPath map

2. **Hybrid Tree + Instruction → Action**
   - LLM identifies single most relevant element
   - Returns method, arguments, and twoStep flag

3. **EncodedId → XPath → Playwright Action**
   - Maps to executable selector
   - Performs Playwright command

4. **(Two-Step) Tree Diff → New Elements**
   - Captures changes after first action
   - Identifies newly visible options

5. **(Self-Heal) Error → Fresh Detection**
   - Retries with updated page state
   - New snapshot → New selector

### Key Files

- **Handler:** `packages/core/lib/v3/handlers/actHandler.ts`
- **Prompts:** `packages/core/lib/prompt.ts` (lines 143-239)
- **Inference:** `packages/core/lib/inference.ts` (lines 358-481)
- **Tree Diff:** `packages/core/lib/v3/handlers/actHandler.ts` (diffCombinedTrees)

### Usage Examples

```typescript
// Simple action
await stagehand.act("click the Login button");

// Action with typing
await stagehand.act('type "john@example.com" into the email field');

// Dropdown (two-step automatically detected)
await stagehand.act("select Large from the size dropdown");

// Key press
await stagehand.act("press Enter");

// Scroll
await stagehand.act("scroll halfway down the page");

// With self-healing enabled
await stagehand.act("click the dynamic button", { selfHeal: true });
```

