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


---

## V3 Agent Execution Pipeline

**Purpose:** Autonomous agent that accomplishes complex multi-step goals by orchestrating multiple tools (act, extract, goto, etc.) until task completion.

**Entry Point:** `V3AgentHandler.execute()` in `packages/core/lib/v3/handlers/v3AgentHandler.ts`

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    V3 AGENT EXECUTION PIPELINE                       │
└─────────────────────────────────────────────────────────────────────┘

User Request: agent.execute(instruction)
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Build System Prompt                                         │
│ Location: v3AgentHandler.ts:58-61 → buildSystemPrompt()             │
│                                                                      │
│ IF custom systemInstructions provided:                              │
│    "${systemInstructions}\n                                         │
│     Your current goal: ${executionInstruction}                      │
│     when the task is complete, use the 'close' tool with           │
│     taskComplete: true"                                              │
│                                                                      │
│ ELSE (Default System Prompt):                                       │
│    "You are a web automation assistant using browser automation    │
│     tools to accomplish the user's goal.                            │
│                                                                      │
│     Your task: ${executionInstruction}                              │
│                                                                      │
│     You have access to various browser automation tools. Use       │
│     them step by step to complete the task.                         │
│                                                                      │
│     IMPORTANT GUIDELINES:                                            │
│     1. Always start by understanding the current page state         │
│     2. Use the screenshot tool to verify page state when needed     │
│     3. Use appropriate tools for each action                         │
│     4. When the task is complete, use the 'close' tool with         │
│        taskComplete: true                                            │
│     5. If the task cannot be completed, use 'close' with            │
│        taskComplete: false                                           │
│                                                                      │
│     TOOLS OVERVIEW:                                                  │
│     - screenshot: Take a PNG screenshot for quick visual context    │
│       (use sparingly)                                                │
│     - ariaTree: Get an accessibility (ARIA) hybrid tree for full   │
│       page context                                                   │
│     - act: Perform a specific atomic action (click, type, etc.)     │
│     - extract: Extract structured data                               │
│     - goto: Navigate to a URL                                        │
│     - wait/navback/refresh: Control timing and navigation           │
│     - scroll: Scroll the page x pixels up or down                   │
│                                                                      │
│     STRATEGY:                                                        │
│     - Prefer ariaTree to understand the page before acting; use     │
│       screenshot for confirmation.                                   │
│     - Keep actions atomic and verify outcomes before proceeding."   │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Create Tools                                                │
│ Location: v3AgentHandler.ts:62 → createAgentTools()                 │
│                                                                      │
│ Available Tools (from lib/v3/agent/tools/):                         │
│                                                                      │
│ ┌─────────────────┬──────────────────────────────────────┐         │
│ │ Tool            │ Description                          │         │
│ ├─────────────────┼──────────────────────────────────────┤         │
│ │ act             │ Perform page action (click, type)    │         │
│ │ ariaTree        │ Get accessibility tree               │         │
│ │ close           │ Signal task completion               │         │
│ │ extract         │ Extract structured data              │         │
│ │ fillForm        │ Fill multiple form fields            │         │
│ │ goto            │ Navigate to URL                      │         │
│ │ navback         │ Go back to previous page             │         │
│ │ screenshot      │ Take PNG screenshot                  │         │
│ │ scroll          │ Scroll page up or down               │         │
│ │ wait            │ Wait for time period                 │         │
│ └─────────────────┴──────────────────────────────────────┘         │
│                                                                      │
│ + MCP Tools (if provided): Custom tools from Model Context Protocol │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Initialize Messages                                         │
│ Location: v3AgentHandler.ts:64-66                                   │
│                                                                      │
│ messages = [                                                         │
│   { role: "user", content: instruction }                            │
│ ]                                                                    │
│                                                                      │
│ → Processed through middleware for formatting                       │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: LLM Generate Text Loop (Tool Calling)                       │
│ Location: v3AgentHandler.ts:82-138                                  │
│                                                                      │
│ Configuration:                                                       │
│   - model: wrappedModel (with message processing middleware)        │
│   - system: systemPrompt (from Step 1)                              │
│   - messages: conversation history                                  │
│   - tools: all tools (from Step 2)                                  │
│   - stopWhen: stepCountIs(maxSteps) // default: 10                  │
│   - temperature: 1                                                   │
│   - toolChoice: "auto"                                               │
│                                                                      │
│ ┌───────────────────────────────────────────────────────┐          │
│ │ EXECUTION LOOP (up to maxSteps)                       │          │
│ │                                                        │          │
│ │ Loop Iteration:                                        │          │
│ │      │                                                 │          │
│ │      ├──► LLM Call with current conversation           │          │
│ │      │                                                 │          │
│ │      ├──► LLM Returns:                                 │          │
│ │      │    - Text (reasoning/thinking)                  │          │
│ │      │    - Tool calls (0 or more)                     │          │
│ │      │                                                 │          │
│ │      ├──► onStepFinish callback triggers:              │          │
│ │      │                                                 │          │
│ │      │    ┌─────────────────────────────────────┐     │          │
│ │      │    │ For each tool call:                 │     │          │
│ │      │    │                                      │     │          │
│ │      │    │ 1. Extract tool name & arguments    │     │          │
│ │      │    │                                      │     │          │
│ │      │    │ 2. IF reasoning text:                │     │          │
│ │      │    │    → Collect for final message      │     │          │
│ │      │    │                                      │     │          │
│ │      │    │ 3. IF tool is "close":               │     │          │
│ │      │    │    → Set completed = true            │     │          │
│ │      │    │    → Extract final message           │     │          │
│ │      │    │                                      │     │          │
│ │      │    │ 4. Execute tool:                     │     │          │
│ │      │    │                                      │     │          │
│ │      │    │    Example - ariaTree tool:          │     │          │
│ │      │    │      → Call v3.extract()             │     │          │
│ │      │    │      → Get pageText                  │     │          │
│ │      │    │      → Truncate if > 70k tokens      │     │          │
│ │      │    │      → Return tree text              │     │          │
│ │      │    │                                      │     │          │
│ │      │    │    Example - act tool:               │     │          │
│ │      │    │      → Call v3.act(action)           │     │          │
│ │      │    │      → Extract resulting actions     │     │          │
│ │      │    │      → Record for agent replay       │     │          │
│ │      │    │      → Return success/error          │     │          │
│ │      │    │                                      │     │          │
│ │      │    │    Example - screenshot tool:        │     │          │
│ │      │    │      → Take page screenshot          │     │          │
│ │      │    │      → Return base64 PNG             │     │          │
│ │      │    │                                      │     │          │
│ │      │    │ 5. Map tool result to AgentAction   │     │          │
│ │      │    │    → Add pageUrl, timestamp          │     │          │
│ │      │    │    → Append to actions array         │     │          │
│ │      │    │                                      │     │          │
│ │      │    │ 6. Tool result added to              │     │          │
│ │      │    │    conversation automatically        │     │          │
│ │      │    └─────────────────────────────────────┘     │          │
│ │      │                                                 │          │
│ │      ├──► IF completed OR maxSteps reached:            │          │
│ │      │    → Exit loop                                  │          │
│ │      │                                                 │          │
│ │      └──► ELSE: Continue to next iteration             │          │
│ │           (LLM sees tool results in next call)         │          │
│ └───────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Build Final Result                                          │
│ Location: v3AgentHandler.ts:140-168                                 │
│                                                                      │
│ IF no finalMessage yet:                                              │
│    → Combine all reasoning text from steps                          │
│    → OR use last LLM response text                                  │
│                                                                      │
│ Calculate inference time                                             │
│ Update metrics (tokens, time)                                        │
│                                                                      │
│ Return: {                                                            │
│   success: completed,        // true if close tool called           │
│   message: finalMessage,     // Combined reasoning                  │
│   actions: AgentAction[],    // All actions taken                   │
│   completed: boolean,         // Task completion status              │
│   usage: {                                                           │
│     input_tokens: number,                                            │
│     output_tokens: number,                                           │
│     inference_time_ms: number                                        │
│   }                                                                  │
│ }                                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Message Flow Example

**Task:** "Find the price of the first product on the page"

```
┌─────────────────────────────────────────────────────────────────────┐
│ ITERATION 1                                                          │
└─────────────────────────────────────────────────────────────────────┘

INPUT:
  System: [agent instructions with tools overview]
  User: "Find the price of the first product on the page"

LLM OUTPUT:
  Reasoning: "I need to first understand the page structure"
  Tool Call: ariaTree()

TOOL EXECUTION:
  → ariaTree executes → Returns accessibility tree

CONVERSATION UPDATE:
  + Assistant: [reasoning + ariaTree tool call]
  + User: [tool result with tree text]

┌─────────────────────────────────────────────────────────────────────┐
│ ITERATION 2                                                          │
└─────────────────────────────────────────────────────────────────────┘

INPUT:
  [Previous conversation + tool result]

LLM OUTPUT:
  Reasoning: "I can see products in the tree. I'll extract the first 
             product's price"
  Tool Call: extract({
    instruction: "extract first product price",
    schema: "z.object({ price: z.number() })"
  })

TOOL EXECUTION:
  → extract executes → Returns { price: 29.99 }

CONVERSATION UPDATE:
  + Assistant: [reasoning + extract tool call]
  + User: [tool result with extracted data]

┌─────────────────────────────────────────────────────────────────────┐
│ ITERATION 3                                                          │
└─────────────────────────────────────────────────────────────────────┘

INPUT:
  [Previous conversation + extraction result]

LLM OUTPUT:
  Reasoning: "I found the price: $29.99"
  Tool Call: close({
    reasoning: "Successfully found the first product price: $29.99",
    taskComplete: true
  })

TOOL EXECUTION:
  → close executes → Sets completed=true

RESULT:
  {
    success: true,
    completed: true,
    message: "I need to first understand the page structure. I can 
             see products in the tree. I'll extract the first product's
             price. I found the price: $29.99. Successfully found the 
             first product price: $29.99",
    actions: [
      { type: "extract", ... },
    ],
    usage: { input_tokens: 1523, output_tokens: 234, ... }
  }
```

### Tool Execution Details

#### ariaTree Tool
```typescript
Input: (none)
Process:
  1. Call v3.extract() without instruction
  2. Get pageText from accessibility tree
  3. Estimate tokens (length / 4)
  4. If > 70,000 tokens: truncate with warning
  5. Return { content: treeText, pageUrl }
Output: Full page structure as text
```

#### act Tool
```typescript
Input: { action: "click the Login button" }
Process:
  1. Log tool invocation
  2. Call v3.act(action, { model: executionModel })
  3. Extract resulting Playwright actions
  4. Record step for agent replay
  5. Return { success, action, playwrightArguments }
Output: Action result
```

#### extract Tool
```typescript
Input: { 
  instruction: "extract product name",
  schema: "z.object({ name: z.string() })"
}
Process:
  1. Evaluate schema string (create Zod object)
  2. Call v3.extract(instruction, schema, { model })
  3. Return { success: true, result: extractedData }
Output: Structured data matching schema
```

#### close Tool
```typescript
Input: { 
  reasoning: "Task completed successfully",
  taskComplete: true
}
Process:
  1. Return immediately (signal only)
  2. Agent handler detects close call
  3. Sets completed flag
  4. Exits loop
Output: { success: true, reasoning, taskComplete }
```

### Key Features

1. **Autonomous Operation**
   - No human in the loop during execution
   - Agent decides which tools to use and when
   - Continues until task complete or max steps

2. **Tool Orchestration**
   - Multiple tools available simultaneously
   - LLM chooses appropriate tool for each step
   - Results from one tool inform next decision

3. **Reasoning Collection**
   - All reasoning/thinking text collected
   - Combined into final message for user
   - Provides transparency into agent's thought process

4. **State Management**
   - Conversation history maintained across steps
   - Each tool result added to context
   - LLM has full history for decision making

5. **Flexible Completion**
   - Agent signals completion with close tool
   - Can indicate success or failure
   - Provides reasoning for outcome

### Key Files

- **Handler:** `packages/core/lib/v3/handlers/v3AgentHandler.ts`
- **Tools:** `packages/core/lib/v3/agent/tools/` (all v3-*.ts files)
- **Tool Creation:** `packages/core/lib/v3/agent/tools/index.ts`
- **Message Processing:** `packages/core/lib/v3/agent/utils/messageProcessing.ts`
- **Action Mapping:** `packages/core/lib/v3/agent/utils/actionMapping.ts`

### Usage Example

```typescript
const agent = await stagehand.agent({
  systemInstructions: "You are a helpful shopping assistant",
  maxSteps: 10
});

const result = await agent.execute(
  "Go to example.com, find the first product, and extract its name and price"
);

console.log(result.message);    // Agent's reasoning
console.log(result.completed);  // true if task completed
console.log(result.actions);    // All actions taken
```


---

## Computer Use Agent Pipelines

Computer Use Agents (CUA) use provider-specific APIs (Anthropic, OpenAI, Google) that support direct browser control. Unlike V3 Agent which uses abstracted tools, CUA agents use provider-native computer use capabilities.

### Anthropic CUA Pipeline

**Entry Point:** `AnthropicCUAClient.execute()` in `packages/core/lib/v3/agent/AnthropicCUAClient.ts`

#### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ANTHROPIC CUA PIPELINE                           │
└─────────────────────────────────────────────────────────────────────┘

User Request: cua.execute(instruction)
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Initialize Messages                                         │
│                                                                      │
│ messages = [                                                         │
│   {                                                                  │
│     role: "user",                                                    │
│     content: userProvidedInstructions || buildCuaDefaultSystemPrompt│
│   },                                                                 │
│   {                                                                  │
│     role: "user",                                                    │
│     content: instruction                                             │
│   }                                                                  │
│ ]                                                                    │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Execute Steps Loop (max 10 by default)                      │
│                                                                      │
│ ┌───────────────────────────────────────────────────────┐          │
│ │ Loop Iteration:                                        │          │
│ │                                                        │          │
│ │ ┌─────────────────────────────────────────────────┐  │          │
│ │ │ 2a. Call Anthropic Messages API                 │  │          │
│ │ │                                                  │  │          │
│ │ │ Parameters:                                      │  │          │
│ │ │   model: user-specified model                   │  │          │
│ │ │   max_tokens: 4096                               │  │          │
│ │ │   messages: conversation history                 │  │          │
│ │ │   betas: ["computer-use-2025-01-24"]            │  │          │
│ │ │   tools: [                                       │  │          │
│ │ │     {                                            │  │          │
│ │ │       type: "computer_20250124",                │  │          │
│ │ │       name: "computer",                          │  │          │
│ │ │       display_width_px: viewport.width,         │  │          │
│ │ │       display_height_px: viewport.height        │  │          │
│ │ │     },                                           │  │          │
│ │ │     ...mcpTools  // Custom MCP tools if any     │  │          │
│ │ │   ]                                              │  │          │
│ │ │   thinking: {                                    │  │          │
│ │ │     type: "enabled",                             │  │          │
│ │ │     budget_tokens: thinkingBudget  // if set    │  │          │
│ │ │   }                                              │  │          │
│ │ └─────────────────────────────────────────────────┘  │          │
│ │      │                                                 │          │
│ │      ▼                                                 │          │
│ │ ┌─────────────────────────────────────────────────┐  │          │
│ │ │ 2b. Process Response                             │  │          │
│ │ │                                                  │  │          │
│ │ │ Extract from content blocks:                     │  │          │
│ │ │   - tool_use blocks → Actions to execute        │  │          │
│ │ │   - text blocks → Reasoning/thinking            │  │          │
│ │ │   - thinking blocks → Extended thinking         │  │          │
│ │ │                                                  │  │          │
│ │ │ For each tool_use block:                         │  │          │
│ │ │   IF tool is "computer":                         │  │          │
│ │ │     → Extract action & parameters                │  │          │
│ │ │     → Convert Anthropic format to Stagehand     │  │          │
│ │ │                                                  │  │          │
│ │ │   Action Conversions:                            │  │          │
│ │ │     "left_click" [x, y] →                       │  │          │
│ │ │       click(x_scaled, y_scaled)                  │  │          │
│ │ │                                                  │  │          │
│ │ │     "type" text →                                │  │          │
│ │ │       type(text)                                 │  │          │
│ │ │                                                  │  │          │
│ │ │     "key" keyName →                              │  │          │
│ │ │       keypress([keyName])                        │  │          │
│ │ │                                                  │  │          │
│ │ │     "scroll" direction distance →                │  │          │
│ │ │       scroll_y: distance pixels                  │  │          │
│ │ │                                                  │  │          │
│ │ │     "cursor_position" [x, y] →                  │  │          │
│ │ │       hover(x_scaled, y_scaled)                  │  │          │
│ │ │                                                  │  │          │
│ │ │     "screenshot" →                               │  │          │
│ │ │       screenshot()                               │  │          │
│ │ │                                                  │  │          │
│ │ │   Coordinate Scaling:                            │  │          │
│ │ │     Anthropic uses 0-1000 range                  │  │          │
│ │ │     x_actual = (x / 1000) * viewport.width      │  │          │
│ │ │     y_actual = (y / 1000) * viewport.height     │  │          │
│ │ └─────────────────────────────────────────────────┘  │          │
│ │      │                                                 │          │
│ │      ▼                                                 │          │
│ │ ┌─────────────────────────────────────────────────┐  │          │
│ │ │ 2c. Execute Actions                              │  │          │
│ │ │                                                  │  │          │
│ │ │ For each converted action:                       │  │          │
│ │ │   → Call actionHandler(action)                   │  │          │
│ │ │   → Performs Playwright command                  │  │          │
│ │ │   → Records action in history                    │  │          │
│ │ └─────────────────────────────────────────────────┘  │          │
│ │      │                                                 │          │
│ │      ▼                                                 │          │
│ │ ┌─────────────────────────────────────────────────┐  │          │
│ │ │ 2d. Generate Tool Results                        │  │          │
│ │ │                                                  │  │          │
│ │ │ For each tool_use:                               │  │          │
│ │ │                                                  │  │          │
│ │ │   IF "computer" tool:                            │  │          │
│ │ │     → Take screenshot                            │  │          │
│ │ │     → Create tool_result:                        │  │          │
│ │ │       {                                          │  │          │
│ │ │         type: "tool_result",                     │  │          │
│ │ │         tool_use_id: toolUseId,                  │  │          │
│ │ │         content: [                               │  │          │
│ │ │           {                                      │  │          │
│ │ │             type: "image",                       │  │          │
│ │ │             source: {                            │  │          │
│ │ │               type: "base64",                    │  │          │
│ │ │               media_type: "image/png",          │  │          │
│ │ │               data: base64Screenshot            │  │          │
│ │ │             }                                    │  │          │
│ │ │           },                                     │  │          │
│ │ │           {                                      │  │          │
│ │ │             type: "text",                        │  │          │
│ │ │             text: "Current URL: https://..."    │  │          │
│ │ │           }                                      │  │          │
│ │ │         ]                                        │  │          │
│ │ │       }                                          │  │          │
│ │ │                                                  │  │          │
│ │ │   IF custom MCP tool:                            │  │          │
│ │ │     → Execute tool via MCP                       │  │          │
│ │ │     → Return result as tool_result               │  │          │
│ │ └─────────────────────────────────────────────────┘  │          │
│ │      │                                                 │          │
│ │      ▼                                                 │          │
│ │ ┌─────────────────────────────────────────────────┐  │          │
│ │ │ 2e. Add to Conversation                          │  │          │
│ │ │                                                  │  │          │
│ │ │ messages.push({                                  │  │          │
│ │ │   role: "assistant",                             │  │          │
│ │ │   content: [                                     │  │          │
│ │ │     ...thinkingBlocks,                           │  │          │
│ │ │     ...textBlocks,                               │  │          │
│ │ │     ...toolUseBlocks                             │  │          │
│ │ │   ]                                              │  │          │
│ │ │ });                                              │  │          │
│ │ │                                                  │  │          │
│ │ │ messages.push({                                  │  │          │
│ │ │   role: "user",                                  │  │          │
│ │ │   content: toolResults                           │  │          │
│ │ │ });                                              │  │          │
│ │ │                                                  │  │          │
│ │ │ Compress conversation images periodically        │  │          │
│ │ │ (keep only recent screenshots)                   │  │          │
│ │ └─────────────────────────────────────────────────┘  │          │
│ │      │                                                 │          │
│ │      ▼                                                 │          │
│ │ ┌─────────────────────────────────────────────────┐  │          │
│ │ │ 2f. Check Completion                             │  │          │
│ │ │                                                  │  │          │
│ │ │ IF no tool_use blocks in response:               │  │          │
│ │ │   → Task completed                               │  │          │
│ │ │   → Exit loop                                    │  │          │
│ │ │                                                  │  │          │
│ │ │ ELSE IF maxSteps reached:                        │  │          │
│ │ │   → Exit loop                                    │  │          │
│ │ │                                                  │  │          │
│ │ │ ELSE:                                            │  │          │
│ │ │   → Continue to next iteration                   │  │          │
│ │ └─────────────────────────────────────────────────┘  │          │
│ └───────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ RETURN TO USER                                                       │
│                                                                      │
│ {                                                                    │
│   completed: boolean,       // No tool_use in last response         │
│   message: string,           // Last text response                   │
│   actions: Action[],         // All actions executed                 │
│   usage: TokenUsage          // Total tokens used                    │
│ }                                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

#### Key Features

1. **Native Computer Control**
   - Uses Anthropic's computer_20250124 tool
   - Direct coordinate-based interaction
   - Screenshot feedback after each action

2. **Coordinate System**
   - Anthropic uses 0-1000 normalized coordinates
   - Scaled to actual viewport dimensions
   - More flexible than selector-based approaches

3. **Extended Thinking**
   - Optional thinking budget for reasoning
   - Separate thinking blocks in response
   - Can use Claude 3.5 Sonnet thinking mode

4. **Image Compression**
   - Keeps only recent screenshots in conversation
   - Prevents context window overflow
   - Maintains enough history for decision-making

---

### OpenAI CUA Pipeline

**Entry Point:** `OpenAICUAClient.execute()` in `packages/core/lib/v3/agent/OpenAICUAClient.ts`

#### Flow Diagram (Differences from Anthropic)

```
Key Differences from Anthropic CUA:

1. API Format:
   - Uses Responses API (not Messages API)
   - Input items instead of messages
   - previous_response_id for continuity

2. Request Structure:
   {
     model: modelName,
     tools: [{
       type: "computer_use_preview",
       display_width: width,
       display_height: height,
       environment: "browser"  // or "mac", "windows", "ubuntu"
     }],
     input: inputItems,
     previous_response_id: previousId
   }

3. Response Processing:
   - Extract computer_call items (not tool_use)
   - Extract function_call items (custom tools)
   - Extract reasoning items (o1-style thinking)
   - Extract message items (text responses)

4. Tool Result Format:
   - computer_call_output (not tool_result)
   - function_call_output (for custom tools)

5. Conversation Tracking:
   - Uses previous_response_id
   - No separate user/assistant roles in input
   - All outputs appended for next request

Action conversions similar to Anthropic CUA
```

---

### Google CUA Pipeline

**Entry Point:** `GoogleCUAClient.execute()` in `packages/core/lib/v3/agent/GoogleCUAClient.ts`

#### Flow Diagram (Differences from Others)

```
Key Differences from Anthropic/OpenAI CUA:

1. Tool Definition:
   {
     computerUse: {
       environment: "ENVIRONMENT_BROWSER"
     },
     functionDeclarations: customTools  // MCP tools if any
   }

2. History Format:
   - Uses parts array for content
   - User/model roles (not assistant)
   - Compressed images (keeps only last 2)

3. Action Conversions:
   Function Name → Stagehand Action
   
   "click_at" [x, y] →
     click(x_actual, y_actual)
   
   "type_text_at" [x, y, text, opts] →
     1. click(x, y)
     2. IF clear_before_typing: select_all + backspace
     3. type(text)
     4. IF press_enter: keypress(["Enter"])
   
   "key_combination" keys →
     keypress(mappedKeys)
     Key mapping: "ctrl" → "Control", etc.
   
   "scroll_document" direction →
     direction == "down": keypress(["PageDown"])
     direction == "up": keypress(["PageUp"])
   
   "scroll_at" [x, y, magnitude] →
     scroll_y: magnitude pixels
   
   "navigate" url →
     goto(url)
   
   "go_back" / "go_forward" →
     navback() / navforward()
   
   "wait_5_seconds" →
     wait(5000)
   
   "drag_and_drop" [x1, y1, x2, y2] →
     drag with path

4. Coordinate Normalization:
   - Google uses 0-1000 range (like Anthropic)
   - Must cap coordinates at 999 to avoid errors
   - Special sanitization in history

5. Retry Logic:
   - Up to 5 attempts with exponential backoff
   - Handles rate limiting gracefully
   - 2s, 4s, 8s, 16s, 32s delays

6. Special Handling:
   - "open_web_browser": No-op (already in browser)
   - "type_text_at": Multi-step automation
   - Adds delays between actions (100ms)

7. Function Response Format:
   {
     name: functionCall.name,
     response: { url: currentUrl },
     parts: [{
       inlineData: {
         mimeType: "image/png",
         data: base64Screenshot
       }
     }]
   }
```

---

## CUA Comparison

| Feature | Anthropic | OpenAI | Google |
|---------|-----------|--------|--------|
| **API Type** | Messages API | Responses API | Generative AI API |
| **Tool Name** | computer_20250124 | computer_use_preview | computerUse |
| **Coordinates** | 0-1000 | 0-1000 | 0-1000 |
| **Environment** | N/A | Configurable | ENVIRONMENT_BROWSER |
| **Thinking** | thinking blocks | reasoning items | N/A (in model) |
| **Continuity** | Conversation | previous_response_id | History |
| **Image Compression** | Yes | Yes | Yes (last 2 only) |
| **Retry Logic** | No | No | Yes (5 attempts) |
| **Action Delays** | No | No | Yes (100ms) |
| **Type Text** | Simple | Simple | Multi-step (click+type+enter) |
| **Key Mapping** | Direct | Direct | Custom mapping |

### When to Use Each CUA

- **Anthropic CUA**: Best for extended thinking, complex reasoning tasks
- **OpenAI CUA**: Good for tasks needing o1-style reasoning, diverse environments
- **Google CUA**: Best for cost-effective automation, built-in retry logic


---

## Evaluator Pipeline

**Purpose:** Evaluate whether a task goal was achieved based on screenshots, agent reasoning, or both. Returns YES/NO evaluation with detailed reasoning.

**Entry Point:** `V3Evaluator.ask()`, `batchAsk()`, or `_evaluateWithMultipleScreenshots()` in `packages/core/lib/v3Evaluator.ts`

### Flow Diagram - Single Evaluation

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EVALUATOR PIPELINE (SINGLE)                       │
└─────────────────────────────────────────────────────────────────────┘

User Request: evaluator.ask({ question, screenshot: true })
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Wait and Capture Screenshot                                 │
│ Location: v3Evaluator.ts:87-92                                      │
│                                                                      │
│      ├──► Wait screenshotDelayMs (default: 250ms)                   │
│      │    → Ensures page rendering is complete                      │
│      │                                                               │
│      └──► IF screenshot enabled:                                    │
│             → page.screenshot({ fullPage: false })                   │
│             → Capture as PNG buffer                                 │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Build Evaluation Prompt                                     │
│ Location: v3Evaluator.ts:85, 98-130                                 │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ SYSTEM PROMPT                            │                        │
│ │                                           │                        │
│ │ "You are an expert evaluator that        │                        │
│ │  confidently returns YES or NO based on  │                        │
│ │  if the original goal was achieved.      │                        │
│ │                                           │                        │
│ │  You have access to                      │                        │
│ │  [a screenshot / the agents reasoning    │                        │
│ │   and actions throughout the task]       │                        │
│ │  that you can use to evaluate the tasks  │                        │
│ │  completion.                              │                        │
│ │                                           │                        │
│ │  Provide detailed reasoning for your     │                        │
│ │  answer.                                  │                        │
│ │                                           │                        │
│ │  Today's date is ${currentDate}"         │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ USER MESSAGE                             │                        │
│ │                                           │                        │
│ │ Content Parts:                            │                        │
│ │                                           │                        │
│ │ 1. Text Part:                             │                        │
│ │    IF agentReasoning:                     │                        │
│ │      "Question: ${question}              │                        │
│ │                                           │                        │
│ │       Agent's reasoning and actions      │                        │
│ │       taken:                              │                        │
│ │       ${agentReasoning}"                  │                        │
│ │    ELSE:                                  │                        │
│ │      "${question}"                        │                        │
│ │                                           │                        │
│ │ 2. Image Part (if screenshot):            │                        │
│ │    {                                      │                        │
│ │      type: "image_url",                   │                        │
│ │      image_url: {                         │                        │
│ │        url: "data:image/jpeg;base64,..." │                        │
│ │      }                                    │                        │
│ │    }                                      │                        │
│ │                                           │                        │
│ │ 3. Answer Part (if provided):             │                        │
│ │    {                                      │                        │
│ │      type: "text",                        │                        │
│ │      text: "the answer is ${answer}"     │                        │
│ │    }                                      │                        │
│ └──────────────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: LLM Call                                                     │
│ Location: v3Evaluator.ts:96-130                                     │
│                                                                      │
│ llmClient.createChatCompletion({                                     │
│   logger: silentLogger,                                              │
│   options: {                                                         │
│     messages: [systemPrompt, userMessage],                           │
│     response_model: {                                                │
│       name: "EvaluationResult",                                      │
│       schema: EvaluationSchema                                       │
│     }                                                                 │
│   }                                                                  │
│ })                                                                   │
│                                                                      │
│ EvaluationSchema = z.object({                                        │
│   evaluation: z.enum(["YES", "NO"]),                                │
│   reasoning: z.string()                                              │
│ })                                                                   │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ RETURN TO USER                                                       │
│                                                                      │
│ {                                                                    │
│   evaluation: "YES" | "NO" | "INVALID",                             │
│   reasoning: "Detailed explanation of why..."                       │
│ }                                                                    │
│                                                                      │
│ Note: "INVALID" returned if parsing fails                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Flow Diagram - Batch Evaluation

```
┌─────────────────────────────────────────────────────────────────────┐
│                   EVALUATOR PIPELINE (BATCH)                         │
└─────────────────────────────────────────────────────────────────────┘

User Request: evaluator.batchAsk({
  questions: [
    { question: "Is user logged in?", answer: "john@example.com" },
    { question: "Is cart showing items?" },
    ...
  ],
  screenshot: true
})
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Capture Single Screenshot                                   │
│                                                                      │
│ One screenshot for all questions                                     │
│      └──► page.screenshot({ fullPage: false })                      │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Format Questions                                            │
│ Location: v3Evaluator.ts:168-173                                    │
│                                                                      │
│ Formatted Text:                                                      │
│ "1. Is user logged in?                                              │
│     Answer: john@example.com                                         │
│                                                                      │
│  2. Is cart showing items?                                           │
│                                                                      │
│  3. Is checkout button visible?                                      │
│     Answer: Yes"                                                     │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Build Batch Prompt                                          │
│ Location: v3Evaluator.ts:180-183                                    │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ SYSTEM PROMPT                            │                        │
│ │                                           │                        │
│ │ "${baseSystemPrompt}                     │                        │
│ │                                           │                        │
│ │  You will be given multiple questions    │                        │
│ │  [with a screenshot].                     │                        │
│ │  [Some questions include answers to      │                        │
│ │   evaluate.]                              │                        │
│ │                                           │                        │
│ │  Answer each question by returning an    │                        │
│ │  object in the specified JSON format.    │                        │
│ │  Return a single JSON array containing   │                        │
│ │  one object for each question in the     │                        │
│ │  order they were asked."                  │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ USER MESSAGE                             │                        │
│ │                                           │                        │
│ │ Content:                                  │                        │
│ │   [formatted questions text]              │                        │
│ │   [screenshot if enabled]                 │                        │
│ └──────────────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: LLM Call with Batch Schema                                  │
│ Location: v3Evaluator.ts:175-207                                    │
│                                                                      │
│ BatchEvaluationSchema = z.array(EvaluationSchema)                   │
│                                                                      │
│ Returns array of evaluations, one per question                      │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ RETURN TO USER                                                       │
│                                                                      │
│ [                                                                    │
│   {                                                                  │
│     evaluation: "YES",                                               │
│     reasoning: "User email john@example.com is visible..."          │
│   },                                                                 │
│   {                                                                  │
│     evaluation: "NO",                                                │
│     reasoning: "Cart appears empty in the screenshot..."            │
│   },                                                                 │
│   ...                                                                │
│ ]                                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Flow Diagram - Multi-Screenshot Evaluation

```
┌─────────────────────────────────────────────────────────────────────┐
│              EVALUATOR PIPELINE (MULTI-SCREENSHOT)                   │
└─────────────────────────────────────────────────────────────────────┘

User Request: evaluator.ask({
  question: "Did the user successfully add item to cart?",
  screenshot: [screenshot1, screenshot2, screenshot3],
  agentReasoning: "Step 1: Clicked product... Step 2: ..."
})
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Build Multi-Screenshot Prompt                               │
│ Location: v3Evaluator.ts:227-243                                    │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ SYSTEM PROMPT                            │                        │
│ │                                           │                        │
│ │ "You are an expert evaluator that        │                        │
│ │  confidently returns YES or NO given a   │                        │
│ │  question and multiple screenshots       │                        │
│ │  showing the progression of a task.      │                        │
│ │                                           │                        │
│ │  [You also have access to the agent's    │                        │
│ │   detailed reasoning and thought process │                        │
│ │   throughout the task.]                   │                        │
│ │                                           │                        │
│ │  Analyze ALL screenshots to understand   │                        │
│ │  the complete journey. Look for evidence │                        │
│ │  of task completion across all           │                        │
│ │  screenshots, not just the last one.     │                        │
│ │                                           │                        │
│ │  Success criteria may appear at          │                        │
│ │  different points in the sequence        │                        │
│ │  (confirmation messages, intermediate    │                        │
│ │   states, etc).                           │                        │
│ │                                           │                        │
│ │  [The agent's reasoning provides crucial │                        │
│ │   context about what actions were        │                        │
│ │   attempted, what was observed, and the  │                        │
│ │   decision-making process. Use this      │                        │
│ │   alongside the visual evidence to make  │                        │
│ │   a comprehensive evaluation.]           │                        │
│ │                                           │                        │
│ │  Today's date is ${currentDate}"         │                        │
│ └──────────────────────────────────────────┘                        │
│                                                                      │
│ ┌──────────────────────────────────────────┐                        │
│ │ USER MESSAGE                             │                        │
│ │                                           │                        │
│ │ IF agentReasoning:                        │                        │
│ │   "Question: ${question}                 │                        │
│ │                                           │                        │
│ │    Agent's reasoning and actions         │                        │
│ │    throughout the task:                   │                        │
│ │    ${agentReasoning}                      │                        │
│ │                                           │                        │
│ │    I'm providing ${screenshots.length}   │                        │
│ │    screenshots showing the progression   │                        │
│ │    of the task. Please analyze both the  │                        │
│ │    agent's reasoning and all screenshots │                        │
│ │    to determine if the task was          │                        │
│ │    completed successfully."               │                        │
│ │                                           │                        │
│ │ ELSE:                                     │                        │
│ │   "${question}                            │                        │
│ │                                           │                        │
│ │    I'm providing ${screenshots.length}   │                        │
│ │    screenshots showing the progression   │                        │
│ │    of the task. Please analyze all of    │                        │
│ │    them to determine if the task was     │                        │
│ │    completed successfully."               │                        │
│ │                                           │                        │
│ │ Content Parts:                            │                        │
│ │   [text part above]                       │                        │
│ │   [image 1: base64 PNG]                   │                        │
│ │   [image 2: base64 PNG]                   │                        │
│ │   [image 3: base64 PNG]                   │                        │
│ │   ...                                     │                        │
│ └──────────────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: LLM Call                                                     │
│ Location: v3Evaluator.ts:261-283                                    │
│                                                                      │
│ Same schema as single evaluation                                    │
│ LLM analyzes all images together                                    │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ RETURN TO USER                                                       │
│                                                                      │
│ {                                                                    │
│   evaluation: "YES",                                                 │
│   reasoning: "Looking across all 3 screenshots, I can see the       │
│              progression: Screenshot 1 shows the product page,      │
│              Screenshot 2 shows the 'Add to Cart' button being      │
│              clicked with a loading state, and Screenshot 3 shows   │
│              the cart icon updated with '1 item' and a confirmation │
│              message. The agent's reasoning confirms these steps.   │
│              Task completed successfully."                           │
│ }                                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Use Cases

#### Single Evaluation
```typescript
// Evaluate based on screenshot only
const result = await evaluator.ask({
  question: "Is the user logged in?",
  screenshot: true
});

// Evaluate based on agent reasoning only
const result = await evaluator.ask({
  question: "Did the agent complete the checkout?",
  screenshot: false,
  agentReasoning: agent.collectedReasoning
});

// Evaluate with both screenshot and reasoning
const result = await evaluator.ask({
  question: "Was the form submitted successfully?",
  screenshot: true,
  agentReasoning: agent.collectedReasoning
});
```

#### Batch Evaluation
```typescript
// Evaluate multiple assertions at once
const results = await evaluator.batchAsk({
  questions: [
    { question: "Is user logged in?" },
    { question: "Is cart showing 3 items?" },
    { question: "Is total price correct?", answer: "$149.99" }
  ],
  screenshot: true
});

// Results: Array of YES/NO evaluations
```

#### Multi-Screenshot Evaluation
```typescript
// Capture screenshots during agent execution
const screenshots = [];
agent.onStep((step) => {
  screenshots.push(step.screenshot);
});

// Evaluate based on entire journey
const result = await evaluator.ask({
  question: "Did the user successfully complete the purchase?",
  screenshot: screenshots,
  agentReasoning: agent.collectedReasoning
});
```

### Key Features

1. **Flexible Input**
   - Screenshot only
   - Agent reasoning only
   - Both screenshot and reasoning
   - Multiple screenshots for progression analysis

2. **Structured Output**
   - Always returns YES/NO/INVALID
   - Detailed reasoning for decision
   - INVALID only on parsing errors

3. **Batch Processing**
   - Multiple questions with one screenshot
   - Efficient for assertion testing
   - Maintains question order

4. **Journey Analysis**
   - Multi-screenshot support
   - Understands task progression
   - Looks for success criteria at any point
   - Combines visual and reasoning evidence

5. **Customizable**
   - Custom system prompt
   - Screenshot delay adjustment
   - Flexible model selection

### Key Files

- **Main Class:** `packages/core/lib/v3Evaluator.ts`
- **Prompts:** Inline in v3Evaluator.ts (lines 85, 151-183, 237-242)
- **Schema:** Zod schemas defined in file (lines 24-29)

### Usage Example

```typescript
// Create evaluator
const evaluator = new V3Evaluator(
  v3Instance,
  "google/gemini-2.5-flash",  // Fast, cheap model
  { apiKey: process.env.GEMINI_API_KEY }
);

// Single evaluation with screenshot
const loginCheck = await evaluator.ask({
  question: "Is the user successfully logged in as john@example.com?",
  screenshot: true,
  screenshotDelayMs: 500  // Wait longer for animations
});

if (loginCheck.evaluation === "YES") {
  console.log("Login successful!");
  console.log("Reasoning:", loginCheck.reasoning);
}

// Batch evaluation
const checks = await evaluator.batchAsk({
  questions: [
    { question: "Is user logged in?" },
    { question: "Is dashboard showing?" },
    { question: "Are notifications visible?" }
  ],
  screenshot: true
});

const allPassed = checks.every(c => c.evaluation === "YES");
```

---

## Summary

This documentation provides complete coverage of all Stagehand agent execution pipelines:

1. **Extract Pipeline** - Structured data extraction from pages
2. **Observe Pipeline** - Finding multiple elements without action
3. **Act Pipeline** - Single action execution with two-step and self-healing
4. **V3 Agent Pipeline** - Autonomous multi-step goal accomplishment
5. **Computer Use Agent Pipelines** - Provider-specific direct browser control
   - Anthropic CUA
   - OpenAI CUA
   - Google CUA
6. **Evaluator Pipeline** - Task completion evaluation with screenshots

Each pipeline is documented with:
- Clear purpose statement
- Entry point location
- ASCII flow diagrams
- Step-by-step execution details
- Prompt construction and usage
- Data transformations
- Key features and differences
- Code examples
- Related files

This documentation enables developers to:
- Understand how AI prompts flow through the system
- See how data transforms at each step
- Choose the right pipeline for their use case
- Debug issues by following the execution path
- Extend the system with custom pipelines

