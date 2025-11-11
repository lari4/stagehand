# Stagehand AI Prompts Documentation

This document provides comprehensive documentation of all AI prompts used in the Stagehand browser automation system. Prompts are organized by their functional category and purpose.

## Table of Contents

1. [Extract Operation Prompts](#extract-operation-prompts)
2. [Observe Operation Prompts](#observe-operation-prompts)
3. [Act Operation Prompts](#act-operation-prompts)
4. [Agent/Operator Prompts](#agentoperator-prompts)
5. [Computer Use Agent Prompts](#computer-use-agent-prompts)
6. [V3 Agent Handler Prompts](#v3-agent-handler-prompts)
7. [Tool Description Prompts](#tool-description-prompts)
8. [Evaluator Prompts](#evaluator-prompts)

---

## Extract Operation Prompts

### 1. Extract System Prompt (`buildExtractSystemPrompt`)

**Purpose:** This system prompt instructs the AI on how to extract content from web pages. It's used when users need to get structured or unstructured data from DOM elements.

**Location:** `packages/core/lib/prompt.ts:19-62`

**Parameters:**
- `isUsingPrintExtractedDataTool` (boolean): Whether to use the print_extracted_data tool
- `userProvidedInstructions` (optional string): Custom instructions from the user

**Key Features:**
- Emphasizes extracting ALL information when user requests a 'list' or 'all'
- Preserves exact text with all symbols, characters, and endlines
- Special handling for links/URLs extraction (responds with element IDs)
- Can enforce tool-based extraction using print_extracted_data tool

**Code:**
```typescript
export function buildExtractSystemPrompt(
  isUsingPrintExtractedDataTool: boolean = false,
  userProvidedInstructions?: string,
): ChatMessage {
  const baseContent = `You are extracting content on behalf of a user.
  If a user asks you to extract a 'list' of information, or 'all' information,
  YOU MUST EXTRACT ALL OF THE INFORMATION THAT THE USER REQUESTS.

  You will be given:
1. An instruction
2. `;

  const contentDetail = `A list of DOM elements to extract from.`;

  const instructions = `
Print the exact text from the DOM elements with all symbols, characters, and endlines as is.
Print null or an empty string if no new information is found.
  `.trim();

  const toolInstructions = isUsingPrintExtractedDataTool
    ? `
ONLY print the content using the print_extracted_data tool provided.
ONLY print the content using the print_extracted_data tool provided.
  `.trim()
    : "";

  const additionalInstructions =
    "If a user is attempting to extract links or URLs, you MUST respond with ONLY the IDs of the link elements. \n" +
    "Do not attempt to extract links directly from the text unless absolutely necessary. ";

  const userInstructions = buildUserInstructionsString(
    userProvidedInstructions,
  );

  const content =
    `${baseContent}${contentDetail}\n\n${instructions}\n${toolInstructions}${
      additionalInstructions ? `\n\n${additionalInstructions}` : ""
    }${userInstructions ? `\n\n${userInstructions}` : ""}`.replace(/\s+/g, " ");

  return {
    role: "system",
    content,
  };
}
```

### 2. Extract User Prompt (`buildExtractUserPrompt`)

**Purpose:** Formats the user's extraction instruction and DOM elements into a user message for the LLM.

**Location:** `packages/core/lib/prompt.ts:64-82`

**Parameters:**
- `instruction` (string): The extraction instruction from the user
- `domElements` (string): Serialized DOM elements to extract from
- `isUsingPrintExtractedDataTool` (boolean): Whether to use the print_extracted_data tool

**Code:**
```typescript
export function buildExtractUserPrompt(
  instruction: string,
  domElements: string,
  isUsingPrintExtractedDataTool: boolean = false,
): ChatMessage {
  let content = `Instruction: ${instruction}
DOM: ${domElements}`;

  if (isUsingPrintExtractedDataTool) {
    content += `
ONLY print the content using the print_extracted_data tool provided.
ONLY print the content using the print_extracted_data tool provided.`;
  }

  return {
    role: "user",
    content,
  };
}
```

### 3. Metadata System Prompt (`buildMetadataSystemPrompt`)

**Purpose:** Evaluates whether an extraction task is complete or if more information is needed. Used for chunked extraction processing.

**Location:** `packages/core/lib/prompt.ts:84-97`

**Key Criteria:**
1. Sets completion to `true` once instruction is satisfied, regardless of remaining chunks
2. Sets completion to `false` only if BOTH conditions are true:
   - The instruction has not been satisfied yet
   - There are still chunks left to process (chunksTotal > chunksSeen)

**Code:**
```typescript
const metadataSystemPrompt = `You are an AI assistant tasked with evaluating the progress and completion status of an extraction task.
Analyze the extraction response and determine if the task is completed or if more information is needed.
Strictly abide by the following criteria:
1. Once the instruction has been satisfied by the current extraction response, ALWAYS set completion status to true and stop processing, regardless of remaining chunks.
2. Only set completion status to false if BOTH of these conditions are true:
   - The instruction has not been satisfied yet
   - There are still chunks left to process (chunksTotal > chunksSeen)`;

export function buildMetadataSystemPrompt(): ChatMessage {
  return {
    role: "system",
    content: metadataSystemPrompt,
  };
}
```

### 4. Metadata Prompt (`buildMetadataPrompt`)

**Purpose:** Formats the extraction instruction and response for metadata evaluation.

**Location:** `packages/core/lib/prompt.ts:99-108`

**Parameters:**
- `instruction` (string): Original extraction instruction
- `extractionResponse` (object): The current extraction result

**Code:**
```typescript
export function buildMetadataPrompt(
  instruction: string,
  extractionResponse: object,
): ChatMessage {
  return {
    role: "user",
    content: `Instruction: ${instruction}
Extracted content: ${JSON.stringify(extractionResponse, null, 2)}`,
  };
}
```

---

## Observe Operation Prompts

### 1. Observe System Prompt (`buildObserveSystemPrompt`)

**Purpose:** Instructs the AI to find elements on a page based on user's observation intent. Used to locate multiple elements that match a description without performing actions on them.

**Location:** `packages/core/lib/prompt.ts:111-130`

**Parameters:**
- `userProvidedInstructions` (optional string): Custom instructions from the user

**Key Features:**
- Helps automate the browser by finding elements based on observation needs
- Works with hierarchical accessibility tree (hybrid of DOM and accessibility tree)
- Returns an array of matching elements (or empty array if none found)
- Supports custom user instructions

**Code:**
```typescript
export function buildObserveSystemPrompt(
  userProvidedInstructions?: string,
): ChatMessage {
  const observeSystemPrompt = `
You are helping the user automate the browser by finding elements based on what the user wants to observe in the page.

You will be given:
1. a instruction of elements to observe
2. a hierarchical accessibility tree showing the semantic structure of the page. The tree is a hybrid of the DOM and the accessibility tree.

Return an array of elements that match the instruction if they exist, otherwise return an empty array.`;
  const content = observeSystemPrompt.replace(/\s+/g, " ");

  return {
    role: "system",
    content: [content, buildUserInstructionsString(userProvidedInstructions)]
      .filter(Boolean)
      .join("\n\n"),
  };
}
```

### 2. Observe User Message (`buildObserveUserMessage`)

**Purpose:** Formats the user's observation instruction and accessibility tree data into a user message for the LLM.

**Location:** `packages/core/lib/prompt.ts:132-141`

**Parameters:**
- `instruction` (string): What elements to observe
- `domElements` (string): The accessibility tree representation of the page

**Code:**
```typescript
export function buildObserveUserMessage(
  instruction: string,
  domElements: string,
): ChatMessage {
  return {
    role: "user",
    content: `instruction: ${instruction}
Accessibility Tree: \n${domElements}\n`,
  };
}
```

