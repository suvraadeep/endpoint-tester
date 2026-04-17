# API Endpoint Executability Validator Agent

## Overview

This project focuses on building an AI-driven agent that validates whether API endpoints are executable in real-world scenarios. The system is designed to simulate a production-level verification layer that ensures endpoints are functional, accessible, and properly configured before integration.

The agent takes a set of API endpoint definitions and determines whether each endpoint:

* Executes successfully
* Exists in the API
* Fails due to insufficient permissions
* Fails due to other errors

The solution is designed to generalize across different applications without relying on hardcoded logic.

---

## Problem Statement

When working with large-scale API integrations, not all endpoints are reliable. Some may:

* Not exist
* Require specific permissions
* Depend on other endpoints
* Fail due to incorrect request construction

The goal of this project was to build an intelligent agent that:

* Dynamically constructs valid API requests
* Resolves dependencies between endpoints
* Executes endpoints using authenticated access
* Accurately classifies outcomes

---

## Setup

### 1. Composio Account and API Key

To execute API calls, I used Composio for managed authentication.

Steps followed:

1. Created an account on [https://platform.composio.dev](https://platform.composio.dev)
2. Generated an API key from the settings section
3. Used the API key during setup

---

### 2. Google Account Connection

The agent interacts with Gmail and Google Calendar APIs.

Setup considerations:

* Used a personal Google account
* Preferred a secondary account to avoid unintended changes
* Ensured the account had existing emails and calendar events

---


### 3. Bun Runtime

The project is built using Bun.

Installation:

```bash
curl -fsSL https://bun.sh/install | bash
```

Verification:

```bash
bun --version
```

---

## Approach

The system is designed using an agent-based architecture where each endpoint is evaluated independently.

Key aspects:

* No fixed execution order
* Dynamic dependency resolution
* Intelligent request construction
* Robust error handling

---

## Core Features

### Endpoint Execution

All API calls are executed using `proxyExecute()` which handles authentication and request execution.

Supports:

* GET, POST, PUT, PATCH, DELETE
* Query parameters
* Request bodies

---

### Endpoint Classification

Each endpoint is classified into one of the following:

| Status              | Description                               |
| ------------------- | ----------------------------------------- |
| valid               | Successfully executed with a 2xx response |
| invalid_endpoint    | Endpoint does not exist                   |
| insufficient_scopes | Permission issues                         |
| error               | Other failures                            |

---

### Dependency Resolution

For endpoints requiring dynamic data such as IDs, the agent:

* Detects path parameters
* Identifies supporting endpoints
* Extracts valid values
* Substitutes them into requests

This enables handling of dependent endpoints without manual intervention.

---

### Request Construction

The agent builds requests based on endpoint metadata:

* Populates query parameters with valid defaults
* Constructs minimal request bodies
* Resolves path variables dynamically

---

### Error Handling

The system distinguishes between:

* Invalid endpoints
* Incorrect request construction
* Permission issues

It minimizes incorrect classifications by retrying with improved inputs when needed.

---

## Sample Data

The project uses a dataset of 16 endpoints:

* 10 Gmail endpoints
* 6 Google Calendar endpoints

These include:

* Valid endpoints
* Fake endpoints
* Endpoints with permission issues

The agent is designed to generalize beyond this dataset.

---

## Project Structure

```bash
src/
├── agent.ts
├── types.ts
├── run.ts
├── endpoints.json
├── index.ts
└── connect.ts

ARCHITECTURE.md
report.json
```

---

## Execution Flow

1. Load endpoint definitions
2. Initialize an agent for each endpoint
3. Construct request
4. Resolve dependencies if required
5. Execute API call
6. Analyze response
7. Classify endpoint
8. Store results

---

## Output Format

The system generates a structured report:

```json
{
  "timestamp": "...",
  "total_endpoints": 16,
  "results": [
    {
      "tool_slug": "GMAIL_LIST_MESSAGES",
      "status": "valid",
      "http_status_code": 200
    }
  ],
  "summary": {
    "valid": 13,
    "invalid_endpoint": 2,
    "insufficient_scopes": 1,
    "error": 0
  }
}
```

---

## Key Challenges

### Avoiding False Negatives

Ensuring valid endpoints are not misclassified due to incorrect requests.

Solution:

* Construct minimal valid inputs
* Retry with alternative parameter configurations
* Validate request structure before execution

---

### Handling Dependencies

Many endpoints require prior data retrieval.

Solution:

* Detect path parameters
* Identify related endpoints dynamically
* Extract and reuse identifiers

---

### Generalization Across APIs

The system must work across different APIs without customization.

Solution:

* Avoid hardcoded logic
* Use schema-driven request construction
* Apply generic dependency patterns

---

## Architecture

To be documented in `ARCHITECTURE.md`.

---

## Results

* Successfully validated endpoints across multiple categories
* Handled dependent endpoints dynamically
* Built a scalable and extensible architecture

---

## Improvements

* Smarter request body generation using schema inference
* Caching dependency results
* More advanced retry strategies
* Confidence scoring for classifications

---

## Conclusion

This project demonstrates how AI agents can automate API validation at scale. The system focuses on correctness, adaptability, and robustness, making it suitable for large-scale API integration workflows.
