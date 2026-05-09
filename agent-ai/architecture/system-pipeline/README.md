# Pipeline Documentation

## 1. Overview

This pipeline processes user queries using:

- **Function Calling (tools)**
- **RAG (Retrieval-Augmented Generation) as one of the tools**
- **LLM for final response generation**

The pipeline ensures:

- selection of the appropriate **tool (or multiple tools)**
- argument extraction
- data retrieval via tools
- generation of the final response


## 2. High-Level Flow

```text
User Query 
→ Tool Decision (LLM)
→ Argument Extraction (LLM)
→ Tool Execution (one or multiple tools)
→ Context Builder 
→ Response Builder 
→ Final Response
```

## 3. Pipeline Steps

### 3.1 Query

**Input:** User text query

**Examples:**
- "Какая сейчас пара?"
- "Где найти декана?"
- "Как перевестись на другой факультет?"

**Output:** Raw user query → passed to Tool Decision

### 3.2 Tool Decision (Function Calling)

**Goal:** Determine which tools need to be invoked

> All data sources are represented as tools:
> - `get_schedule`
> - `get_teacher_info`
> - `rag_retrieval`
> - `router` *(planned for future implementation)*

One or multiple tools can be selected.

**Logic:**

| Query Type        | Tool                 |
|-------------------|----------------------|
| Schedule-related  | `get_schedule`       |
| Teacher/staff     | `get_teacher_info`   |
| General questions | `rag_retrieval`      |
| Mixed query       | multiple tools       |

**Output:**

```json
{
  "tool_calls": [
    { "tool": "get_schedule", "arguments": {} }
  ]
}
```

or (multi-tool):

```json
{
  "tool_calls": [
    { "tool": "get_schedule", "arguments": {} },
    { "tool": "get_teacher_info", "arguments": {} }
  ]
}
```

### 3.3 Argument Extraction + Normalization

**Goal:** Extract parameters for each selected tool

**Description**:
- LLM extracts relevant arguments from the user query
- Extracted values are normalized (e.g., time formats, names, entities)
- Normalization ensures compatibility with tool input requirements

| Query                                  | Arguments             |
|----------------------------------------|-----------------------|
| "Какая сейчас пара?"                   | `time=now`            |
| "Где найти Иванова?"                   | `person_name=Ivanov`  |
| "Как перевестись на другой факультет?" | `user_question`       |

**Normalization examples**:
- "сейчас" → time=now
- "Иванова" → person_name=Ivanov
- "завтра" → date=YYYY-MM-DD

**Output:**

```json
{
  "tool_calls": [
    {
      "tool": "rag_retrieval",
      "arguments": { "user_question": "..." }
    }
  ]
}
```

### 3.4 Tool Execution

The LLM invokes the selected tools. All tools are invoked in the same way (via function calling).

**Available tools:**
- `get_schedule`
- `get_teacher_info`
- `rag_retrieval`
- `router` *(planned)*

#### RAG as a Tool

If `rag_retrieval` is selected, it runs an internal retrieval pipeline:

1. Reformulate query
2. Embedding
3. Vector Search
4. Reranking

**Output:**

```json
{
  "documents": [
    { "text": "...", "score": 0.92 },
    { "text": "...", "score": 0.87 }
  ]
}
```

#### Multi-Tool Execution

If the query contains multiple intents, all relevant tools are called and their results are passed further.

**Example:** "Кто ведёт пару и где найти преподавателя?"

Calls: `get_schedule` + `get_teacher_info`

### 3.5 Context Builder

**Goal:** Combine results from all invoked tools into a single context

**Format:**

```json
{
  "context": [
    { "tool": "get_schedule", "data": {} },
    { "tool": "get_teacher_info", "data": {} },
    { "tool": "rag_retrieval", "data": "documents..." }
  ]
}
```

### 3.6 Response Builder

**Goal:** Generate the final answer using LLM

**Input:**
- query
- aggregated context

**Output:**

```json
{
  "answer": "..."
}
```

### 3.7 Final Response

**Goal:** Return the final response to the user in a human-readable format, combining data from all tools.

## 4. Data Flow Summary

| Step                | Input     | Output        |
|---------------------|-----------|---------------|
| Query               | user text | raw query     |
| Tool Decision       | query     | tool_calls    |
| Argument Extraction | query     | arguments     |
| Tool Execution      | arguments | data          |
| Context Builder     | data      | context       |
| Response Builder    | context   | answer        |
| Final Response      | answer    | user response |

