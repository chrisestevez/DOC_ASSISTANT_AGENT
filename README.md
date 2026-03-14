# Document Assistant Project Instructions

Welcome to the Document Assistant project! This project builds a sophisticated document processing system using LangChain and LangGraph. The AI assistant can answer questions, summarize documents, and perform calculations on financial and healthcare documents.

## Project Overview

This document assistant uses a multi-agent architecture with LangGraph to handle different types of user requests:
- **Q&A Agent**: Answers specific questions about document content
- **Summarization Agent**: Creates summaries and extracts key points from documents
- **Calculation Agent**: Performs mathematical operations on document data

### Prerequisites
- Python 3.9+
- OpenAI API key

### Installation

1. Clone the repository:
```bash
cd <repository_path>
```

2. Create a virtual environment:
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

3. Install dependencies:
```bash
pip install -r requirements.txt
```

4. Create a `.env` file:
```bash
cp .env.example .env
# Edit .env and add your OpenAI API key
```

### Running the Assistant

```bash
python main.py
```

## Project Structure
```
doc_assistant_project/
├── src/
│   ├── schemas.py        # Pydantic models
│   ├── retrieval.py      # Document retrieval
│   ├── tools.py          # Agent tools
│   ├── prompts.py        # Prompt templates
│   ├── agent.py          # LangGraph workflow
│   └── assistant.py      # Main agent
├── sessions/             # Saved conversation sessions
├── main.py               # Entry point
├── requirements.txt      # Dependencies
└── README.md             # This file
```

## Agent Architecture

The LangGraph agent follows this workflow:

![](./docs/langgraph_agent_architecture.png)

## Multi Agent Highlights

## Intent Node

The entry point for all user interactions. This node classifies the user's input and routes it to the appropriate downstream agent.

### Behavior
- Scores user intent on a 0–1 confidence scale
- Routes to one of three agents based on classification: **QA Agent**, **Summarization Agent**, or **Calculation Agent**
- If intent is ambiguous or confidence is low, defaults to the **QA Agent** for clarification

## AgentState
Serves as the single shared memory object passed through every node in the LangGraph workflow each node reads from it, performs its task, and returns a partial update that LangGraph merges back into the state automatically. It consolidates everything needed to manage a full agent session: the conversation history, classified intent, routing instructions, active documents, tool usage, and session identifiers, keeping all context in one place rather than scattered across individual node functions.

Conversation - the latest user input and full message history
Routing - the classified intent and which node to visit next
Context - a conversation summary and any active documents in scope
Task - the current response payload, tools used, and an append-only log of actions taken

Plus **session_id** and **user_id** for session management.

## StateGraph
StateGraph is LangGraph's core abstraction for building stateful, multi-step agent workflows.

### Behavior

* Defines a directed graph where each node is a function that reads and updates a shared state object (in this case AgentState).
* Each node receives the full current state and returns a partial update LangGraph merges it back automatically.
* The graph is stateless by default, but compiling with a checkpointer (e.g. InMemorySaver) snapshots the state after every node enabling pause/resume, human-in-the-loop patterns, and multi-turn memory.

In short, StateGraph gives you a structured, inspectable alternative to writing raw agent loops with built-in state management and routing.

## InMemorySaver 

InMemorySaver is LangGraph's built-in checkpointer that snapshots the full AgentState after every node execution, storing it in a Python dictionary in memory.

* Each snapshot is keyed by thread_id (from the run config), allowing multiple independent conversations to be tracked simultaneously.
* On each new invocation, LangGraph loads the latest checkpoint for that thread_id and resumes from where it left off enabling multi-turn memory without manually passing history.
* State is ephemeral and lives only for the lifetime of the process, making it suitable for development and testing but not production.

## Structured Outputs
The system uses Pydantic Schemas for structured outputs when interacting with the LLM.

### Behavior
* Pydantic schemas define the exact structure the LLM must return, ensuring outputs are typed, validated, and predictable rather than free-form text.
* When bound via **response_schema**, the LLM is constrained to populate the schema's fields LangGraph/LangChain handles the parsing and validation automatically.
* Fields can be typed with primitives (str, int, float), enums, nested models, or Optional giving fine-grained control over the shape of each agent's response.
* If the model returns a malformed response, Pydantic raises a ValidationError immediately, making failures explicit and easy to catch rather than silently propagating bad data.
* Schemas also serve as self-documenting contracts between nodes any downstream node and knows exactly what fields to expect from the previous agent's output.

## Example Conversation

### What's the total amount in invoice INV-001?
**intent**
```JSON
{
"intent_type": "calculation",
"confidence": 0.95,
"reasoning": "The user is asking for the total amount in a specific invoice, which implies a need to perform a calculation or retrieve a numerical value from the document. This aligns with the 'calculation' intent category."
}
```

**structured_response**
```JSON
{
"expression": "20000 + 2000",
"result": 22000.0,
"explanation": "The total amount in invoice INV-001 is calculated by adding the subtotal of $20,000 and the tax of $2,000.",
"units": "USD",
"timestamp": "2026-03-14T01:29:54.057775"
}
```

**tools_used**
```Python
[
"document_reader",
"calculator",
"CalculationResponse"
]
```

**actions_taken**
```Python
[
"classify_intent",
"calculation_agent",
"update_memory"
]
```

**conversation_summary**

The user asked for the total amount in invoice INV-001. The invoice includes consulting services, software development, and support & maintenance, totaling a subtotal of $20,000 with an additional 10% tax of $2,000. The total amount calculated is $22,000.

### Summarize all contracts
**intent**
```JSON
{
"intent_type": "summarization",
"confidence": 0.95,
"reasoning": "The user explicitly requests a summary of all contracts, which aligns with the 'summarization' category. There is no indication of a need for calculations or specific questions about the documents, making 'qa' and 'calculation' less applicable. The request is clear, reducing the likelihood of 'unknown' being the appropriate classification."
}
```

**structured_response**
```JSON
{
"original_length": 503,
"summary": "The Service Agreement (ID: CON-001) is a contract between DocDacity Solutions Inc. and Healthcare Partners LLC, effective January 1, 2024. It covers services such as document processing platform access, 24/7 technical support, monthly data analytics reports, and compliance monitoring. The contract is valued at $180,000 over a 12-month period, with a monthly fee of $15,000. Either party can terminate the agreement with a 60-day written notice.",
"key_points": [
"Contract between DocDacity Solutions Inc. and Healthcare Partners LLC",
"Effective date: January 1, 2024",
"Services include document processing, technical support, data analytics, and compliance monitoring",
"Contract duration: 12 months",
"Total contract value: $180,000",
"Monthly fee: $15,000",
"Termination with 60 days notice"
],
"document_ids": [
"CON-001"
],
"timestamp": "2026-03-14T02:17:26.437883"
}
```

**tools_used**
```Python
[
"document_search",
"document_reader",
"SummarizationResponse"
]
```

**actions_taken**
```Python
[
"classify_intent",
"summarization_agent",
"update_memory"
]
```
**conversation_summary**

The Service Agreement (ID: CON-001) is a contract between DocDacity Solutions Inc. and Healthcare Partners LLC, effective January 1, 2024. It covers services such as document processing platform access, 24/7 technical support, monthly data analytics reports, and compliance monitoring. The contract is valued at $180,000 over a 12-month period, with a monthly fee of $15,000. Either party can terminate the agreement with a 60-day written notice.
     

### Calculate the sum of all invoice totals
**intent**
```JSON
{
"intent_type": "calculation",
"confidence": 0.95,
"reasoning": "The user's request explicitly asks to 'calculate the sum,' indicating a need for mathematical operations or numerical computations. The mention of 'invoice totals' suggests that the user is referring to a set of numerical values that need to be added together, which aligns with the 'calculation' category."
}
```

**structured_response**
```JSON
{
"expression": "22000 + 69300 + 214500",
"result": 305800.0,
"explanation": "The sum of all invoice totals is calculated by adding the total amounts from each invoice: $22,000 from Invoice #12345, $69,300 from Invoice #12346, and $214,500 from Invoice #12347.",
"units": "USD",
"timestamp": "2026-03-14T02:25:46.154448"
}
```

**tools_used**
```Python
[
"document_statistics",
"document_search",
"document_reader",
"document_reader",
"document_reader",
"calculator",
"CalculationResponse"
]
```

**actions_taken**
```Python
[
"classify_intent",
"calculation_agent",
"update_memory"
]
```

**conversation_summary**

The user requested the sum of all invoice totals. Three invoices were identified: Invoice #12345 with a total of $22,000, Invoice #12346 with a total of $69,300, and Invoice #12347 with a total of $214,500. The sum of these invoice totals was calculated to be $305,800.
     
### Find documents with amounts over $50,000
**intent**
```JSON
{
"intent_type": "qa",
"confidence": 0.85,
"reasoning": "The user is asking to find documents based on a specific criterion (amounts over $50,000). This request involves querying or searching through documents to identify those that meet the specified condition. It does not explicitly require performing calculations, but rather filtering or retrieving information based on a given threshold. Therefore, the intent is best classified as 'qa' with a high confidence score."
}
```

**structured_response**
```JSON
{
{
"question": "Find documents with amounts over $50,000",
"answer": "I found 3 documents with amounts over $50,000:\n\n1. **Invoice #12347** (ID: INV-003)\n   - Type: Invoice\n   - Amount: $214,500.00\n   - Date: 2024-03-01\n   - Client: Global Corp\n\n2. **Service Agreement** (ID: CON-001)\n   - Type: Contract\n   - Amount: $180,000.00\n   - Date: January 1, 2024\n   - Provider: DocDacity Solutions Inc.\n\n3. **Invoice #12346** (ID: INV-002)\n   - Type: Invoice\n   - Amount: $69,300.00\n   - Date: 2024-02-20\n   - Client: TechStart Inc.",
"sources": [
"INV-003",
"CON-001",
"INV-002"
],
"confidence": 0.95,
"timestamp": "2026-03-14T03:21:22.828653"
}
}
```

**tools_used**
```Python
[
"document_search",
"AnswerResponse"
]
```

**actions_taken**
```Python
[
"classify_intent",
"qa_agent",
"update_memory"
]
```

**conversation_summary**

The user requested documents with amounts over $50,000. Three documents were found:
1. Invoice #12347 (ID: INV-003) with an amount of $214,500.00 for Global Corp.
2. Service Agreement (ID: CON-001) with an amount of $180,000.00 for DocDacity Solutions Inc.
3. Invoice #12346 (ID: INV-002) with an amount of $69,300.00 for TechStart Inc.
      


