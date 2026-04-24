# Oracle AI Agent Studio — Technical Deep Dive

> A comprehensive guide covering the architecture, key concepts, tools, configuration, deployment, and best practices for building AI agents in Oracle Fusion using AI Agent Studio.

---

## Table of Contents

1. [Overall Architecture](#1-overall-architecture)
2. [Key Concepts](#2-key-concepts)
   - [Agent Teams (Workflows)](#21-agent-teams-workflows)
   - [Agents (Workers)](#22-agents-workers)
   - [Tools](#23-tools)
   - [Topics](#24-topics)
3. [Agent Team Patterns](#3-agent-team-patterns)
   - [Single-Agent Teams](#31-single-agent-teams)
   - [Multi-Agent Teams (Supervisor-Worker)](#32-multi-agent-teams-supervisor-worker)
4. [Seeded Templates](#4-seeded-templates)
5. [Tools In-Depth](#5-tools-in-depth)
   - [Business Object Tool](#51-business-object-tool)
   - [Document Tool (RAG)](#52-document-tool-rag)
   - [Deep Link Tool](#53-deep-link-tool)
   - [External REST Tool](#54-external-rest-tool)
   - [Email Tool](#55-email-tool)
   - [Calculator Tool](#56-calculator-tool)
   - [getUserSession Tool](#57-getusersession-tool)
6. [Working with System Prompts and Topics](#6-working-with-system-prompts-and-topics)
7. [Tips and Tricks](#7-tips-and-tricks)
   - [System Prompt Best Practices](#71-system-prompt-best-practices)
   - [Topic Best Practices](#72-topic-best-practices)
   - [Multi-Agent Configuration Tips](#73-multi-agent-configuration-tips)
   - [Common Mistakes to Avoid](#74-common-mistakes-to-avoid)
8. [Human in the Loop](#8-human-in-the-loop)
9. [Agent Team Configuration Details](#9-agent-team-configuration-details)
10. [Testing, Deployment, and Invocation](#10-testing-deployment-and-invocation)
11. [Security and Access Control](#11-security-and-access-control)
12. [LLM Integration](#12-llm-integration)
13. [Future Roadmap](#13-future-roadmap)
14. [Reference Diagrams (Image Files)](#14-reference-diagrams-image-files)

---

## 1. Overall Architecture

Oracle AI Agent Studio is the central design-time tool for defining, configuring, testing, versioning, and deploying AI agents and agent teams within Oracle Fusion.

### How It All Fits Together

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Agent Studio (Design Time)                  │
│  Define & configure agents, agent teams, tools, topics              │
│  Test & evaluate │ Version & deploy                                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ Generates
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Agent Configuration (Metadata)                    │
│  Stored in Customer Instance                                        │
│  Describes: workflow, agents, tools, topics                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ Loaded at execution time
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Fusion AI Services (Spectra Services)                   │
│  Lives within the customer instance                                 │
│  Pulls agent config using unique agent team code                    │
│  Converts config into executable framework elements                 │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────┐      │
│  │  Execution Engine                                         │      │
│  │  • Tools execute here (email, calculator, etc.)           │      │
│  │  • External API calls go through a proxy                  │      │
│  │  • Document tool & Business Object tool access Fusion     │      │
│  │    data layer                                             │      │
│  └───────────────────────────────────────────────────────────┘      │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────┐      │
│  │  LLM Calls (multiple per turn)                            │      │
│  │  • Planning stage: create plan to solve task              │      │
│  │  • Tool calling: delegate work, generate API calls        │      │
│  │  • Response generation: format final answer for user      │      │
│  └───────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
                               ▲
                               │ Invoke API
                               │
┌──────────────────────────────┴──────────────────────────────────────┐
│  Callers: Fusion UI │ Backend Process │ VB Page │ HCM Journey │     │
│           External Environment │ Custom Application                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Architecture Points

- **Agent Studio** is where you define and configure all artifacts (agents, teams, tools, topics).
- An **Agent Configuration** (metadata) is stored in the customer instance, describing the workflow, agents, tools, and topics.
- An **Invoke API** is auto-generated for each agent team. It uses the unique **agent team code** to load the correct configuration.
- **Fusion AI Services (Spectra)** live within the customer instance and handle all execution.
- **Tools execute within the Spectra instance** — even external REST calls go through a proxy running inside the instance.
- **Document tool** and **Business Object tool** access information from the Fusion data layer.
- **LLMs are called multiple times per turn**:
  1. **Planning** — the LLM creates a plan for how to solve the task.
  2. **Tool Calling** — the LLM generates API calls, slots variables from context and session history.
  3. **Response Generation** — the LLM takes tool outputs and creates a formatted response for the end user.

---

## 2. Key Concepts

The four core artifacts of any agent workflow:

### 2.1 Agent Teams (Workflows)

The **Agent Team** is the top-level wrapper that:

- Acts as the interface between the end user and the system.
- Receives user requests (questions, goals, tasks) via the **Invoke API**.
- Performs **planning** — decides how to break down the task.
- **Delegates work** to worker agents.
- **Orchestrates the conversation** — collects responses from agents/tools and generates a final response to the user via an additional LLM call.
- Can be called from a UI, back-end process, VB page, HCM Journey, or any external system with proper credentials.

> **Important:** The agent team does **not** have direct access to tools. It has access to **worker agents**, and those worker agents have access to tools. This enforces a separation of duties.

### 2.2 Agents (Workers)

**Agents** (also called **Workers**) are responsible for specific types of tasks:

- Each agent team requires **at least one agent**.
- Agents have specific **capabilities and behaviors** defined through:
  - **System prompts** — role, persona, instructions, boundaries.
  - **Tools** — give agents the ability to perform actions.
  - **Topics** — refine scope and behavior.
- Agents are designed around **task-specific behavior** (e.g., access payroll information, make recommendations, handle returns).

**Analogy:** Think of a car service center. The **service advisor** is the agent team (workflow/supervisor) — your interface. The **mechanics** are the worker agents — each specializing in different tasks (oil change, transmission, etc.). The **tools and equipment** are the tools available to each mechanic.

### 2.3 Tools

**Tools** are the core of how agents accomplish their tasks:

- Without tools, agents can only answer generic questions with limited context.
- Tools are responsible for **grabbing context** the agent needs and **performing actions**.
- Tools are defined as **separate constructs** and can be of several types (see [Section 5](#5-tools-in-depth)).
- Tools are **selected by agents** at execution time based on the context of the user's request.
- An agent can have **multiple tools** assigned to it.

### 2.4 Topics

**Topics** are a reusable abstraction for fine-tuning agent behavior:

- They allow you to take a generally configured agent and **customize it for a specific domain** or unique organizational requirements.
- **Reusable** across any number of agent teams and agents.

**Use Cases for Topics:**

| Use Case | Example |
|---|---|
| **Define subject area / set boundaries** | Agent responds differently depending on region |
| **Manage conversations** | Consistent greetings, conversation endings, follow-up questions |
| **Tool usage guidance** | Instructions on when and how to use specific tools |
| **Sequence control** | Define step-by-step workflows |
| **Functional information** | Business rules specific to the organization |
| **Non-functional behavior** | Output formatting, handling out-of-context queries |

> **Why use Topics instead of System Prompts?** Topics are reusable across all agent teams and agents. It's much easier to update a single topic and deploy it across all teams than to update every system prompt individually.

---

## 3. Agent Team Patterns

### 3.1 Single-Agent Teams

The simplest configuration:

```
Agent Team (Workflow)
  └── Agent (Worker)
        ├── Tool (e.g., Document Tool with benefits documents)
        ├── Tool (e.g., Business Object Tool)
        └── Topic (e.g., how to interact with users)
```

- One agent team contains one agent.
- That agent has one or more tools and optionally topics.
- Suitable for straightforward, single-domain tasks.

### 3.2 Multi-Agent Teams (Supervisor-Worker)

The pattern supported in the first release is **Supervisor-Worker**:

```
Agent Team (Workflow)
  └── Supervisor
        ├── Worker Agent 1
        │     ├── Tool A
        │     ├── Tool B
        │     └── Topic X
        ├── Worker Agent 2
        │     ├── Tool C
        │     └── Topic Y
        └── Worker Agent 3
              └── Tool D
```

**How it works:**

1. The **Supervisor** receives the user's request.
2. The Supervisor **plans** and determines which worker agent(s) to delegate to.
3. The Supervisor **communicates with Worker Agent 1**, which executes and responds back to the Supervisor.
4. The Supervisor then **communicates with Worker Agent 2**, and so on.
5. **Agents do not interact with each other directly** — all communication flows through the Supervisor.
6. The Supervisor **orchestrates the final response** back to the user.

**Key Characteristics:**

- This is a stable **hub-and-spoke network**.
- Sequential steps can be supported through instructions.
- The Supervisor uses agent descriptions to decide which agent to delegate to.
- Each worker agent can have its own tools and topics devoted to its specific task.

**Supervisor System Prompt (default):**
The supervisor is pre-configured with a general prompt that tells it to:
- Plan and break down tasks into smaller subtasks.
- Delegate those subtasks to the appropriate worker agents.
- You can modify and augment this prompt.

### Agent Partitioning Strategies

| Strategy | Description | Example |
|---|---|---|
| **By Business Domain** | Agents organized around different business areas | Job requisition: one agent for job descriptions, one for scheduling interviews, one for hiring steps |
| **By Function** | Agents organized around functional specialties | One for data retrieval, one for recommendations, one for actions |

**Benefits of proper partitioning:**
- Focused expertise per agent
- Better accuracy and more helpful responses
- Logical grouping of tools

---

## 4. Seeded Templates

Oracle delivers **preconfigured (seeded) templates** out of the box:

- Available on the Agent Studio landing page.
- Designed by Oracle engineering for specific tasks within specific applications and product families.
- **Great learning resources** — examples of high-quality system prompts, topic usage, and tool configurations.

### Instantiating a Template

1. Select a template (e.g., "Return Order Policy Assistant").
2. Walk through a guided process to create **unique names** for each artifact:
   - Agent team name and code (the code is used in the Invoke API).
   - Agent names and codes.
   - Tool names and codes.
3. Assign a **product family** and **product**.
4. The template is **instantiated** (copied) into your environment.
5. You can **add, remove, or modify** tools and topics after instantiation.
6. The instance must be **published** before it is available to end users.

### Example: Return Order Policy Assistant

This seeded template demonstrates an excellent system prompt:

- **Persona and Role:** "An intelligent, structured assistant that supports customer service reps; role is to guide and inform them on managing returns."
- **Boundary Conditions:** Specifies the types of questions the agent can handle.
- **Step-by-Step Instructions:**
  1. Extract the order number.
  2. Get order details.
  3. Get order line.
  4. Look at the return policy (document tool).
  5. Come back with an answer.
  6. If return is eligible, send a deep link to the Fusion page with context.
  7. Use calculator as needed.
- **Output Formatting:** Specifies how to format the response.
- **Do-Nots:** Clear list of what the agent should NOT do.

---

## 5. Tools In-Depth

### Tools Overview

| Tool Type | Purpose | Data Source |
|---|---|---|
| **Business Object** | CRUD operations on Fusion business objects | Fusion database (REST APIs) |
| **Document** | RAG-based semantic search over uploaded documents | Uploaded documents (vectorized) |
| **Deep Link** | Send users to a specific Fusion page with context | Fusion application URLs |
| **External REST** | Connect to any external REST API | Any external API endpoint |
| **Email** | Send emails to specific roles | Organization email |
| **Calculator** | Perform complex mathematical expressions | N/A |
| **getUserSession** | Get business context of the current user | Fusion user session |

### 5.1 Business Object Tool

Connects to **Fusion business objects** for data retrieval and actions:

- **Operations supported:** GET, POST, PATCH, DELETE (full CRUD).
- **Seeded business objects** are delivered out of the box by Oracle Fusion teams, curated for optimal performance with pruned response fields.
- **Custom business objects** are also supported (including custom fields).
- **Data security stays intact** — uses the actual Fusion user token and functional security.

#### Configuration Steps

1. Create a new Business Object tool with a unique tool code.
2. Select the business object (e.g., `absences`, or a custom BO like `TestBO_c`).
3. Specify the resource type (e.g., monolith resource).
4. **Read from OpenAPI Spec** — the tool reads the OpenAPI spec to show all available functions and capabilities.
5. Select the desired function (e.g., "Get All Absence Records").
6. **Prune fields** — select only the fields that are critical or important (e.g., person number, absence type, start date, end date). This reduces unnecessary context passed to the session history.

#### Best Practices

- **Prune response fields** aggressively — don't pass all fields back to the LLM context. Only include what's needed.
- Use **"Add Field from Spec"** to read available fields from the OpenAPI specification.
- Custom objects published in a sandbox are available in the OpenAPI spec.
- Consider enabling **Human in the Loop** for POST/PATCH/DELETE operations (see [Section 8](#8-human-in-the-loop)).

### 5.2 Document Tool (RAG)

The Document Tool implements a **Retrieval-Augmented Generation (RAG)** system:

#### How It Works

1. **Upload documents** into the tool.
2. Documents are **processed**: chunked, vectorized, and embedded into a vector database stored in the customer instance.
3. When a request comes in, a **semantic similarity search** is performed against the vector embeddings.
4. Relevant **chunks** are retrieved and provided as context to the LLM.
5. The LLM generates a response based on the retrieved context.
6. **Document sources/references** can be shown to the end user.

#### Document Lifecycle

| Status | Description |
|---|---|
| **Draft** | Document has been uploaded but not yet processed |
| **Ready to Publish** | Document is queued for processing by the ESS job |
| **Published** | Document has been vectorized and is available for semantic search |
| **Ready to Delete** | Document is marked for removal |

#### Publishing Documents

- An **ESS (Enterprise Scheduler Service) job** called **"Process Agent Documents"** handles vectorization.
- This job runs on a **regular cycle**, or you can manually trigger it from the ESS screen.
- Steps: Navigate to Scheduled Processes > Schedule New Process > "Process Agent Documents" > Submit.

### 5.3 Deep Link Tool

Sends users into the **Fusion application** at a specific page with relevant context:

- Useful when you **don't want the agent to perform the operation** but instead redirect the user to the appropriate Fusion page.
- **Context can be passed** via query parameters (e.g., order number).
- The LLM **slots the parameter values** when the deep link tool is invoked at runtime.

#### Configuration

1. Define the relative URL for the target Fusion page.
2. Add **query parameters** (e.g., `orderNumber`).
3. Provide a **description** for each parameter so the LLM knows what values to slot.

#### Example

```
URL: /order
Query Parameter: orderNumber
Description: "The order number to load on the Fusion page"
```

Many pre-built deep link examples are available (e.g., "Add Absences" which navigates directly to the absence creation page).

### 5.4 External REST Tool

Connects to **any external REST API**:

#### Configuration

1. **Base URL** — the endpoint of the external service (e.g., `https://api.weather.gov`).
2. **Authentication** — supports multiple methods via **Trap proxy**:
   - None
   - Basic Authentication
   - OAuth 2.0 (Client Credentials and other flows)
   - OCI API Signatures
3. **Functions** — define the API operations (e.g., `getGridPoints`, `getAnswer`).
4. **Parameters** — define URL parameters, headers, body, and query parameters.
5. **Sample queries** — provide example values so the LLM knows the expected format (e.g., latitude/longitude format for weather API).

#### How It Works

- All external calls go through a **proxy called Trap** running inside the customer instance.
- The execution of the call happens within the Spectra framework; the call itself goes out to the external service.
- You provide all parameter information **for the LLM to generate the proper API call** at runtime (slotting variables from context).
- You can **test functions** from within Agent Studio.
- Calls can also be tested via **Postman** or **cURL**.

#### Examples from the Demo

| Service | Auth Type | Use Case |
|---|---|---|
| **Salesforce** | OAuth 2.0 Credentials | Query accounts, contacts |
| **Spotify** | OAuth 2.0 Credentials | Search for songs, artists |
| **SerpAPI** (web search) | None (API key in query) | Perform general web searches |
| **Weather.gov** | None | Get weather forecasts |

### 5.5 Email Tool

- Sends emails to specific **roles** within the organization.
- Easy to set up — just generates the email content.
- You select the target role for delivery.

### 5.6 Calculator Tool

- Performs **complex mathematical expressions** beyond what LLMs can reliably handle.
- Recommended because LLMs can hallucinate when performing math.
- Optional addition to any agent.

### 5.7 getUserSession Tool

- Retrieves the **business context of the user** interacting with the agent.
- Recommended for **most agents and workflows** to understand who the user is and their context.
- Provides user-specific information that can be used by other tools and the LLM.

---

## 6. Working with System Prompts and Topics

### System Prompts

Each agent has a **system prompt** that defines:

| Element | Description |
|---|---|
| **Role & Persona** | Who the agent is and what it represents |
| **Abilities / Capabilities** | What the agent can do |
| **Detailed Instructions** | Step-by-step guidance on how to handle requests |
| **Tool Usage** | Which tools to use, in what order, and under what conditions |
| **Output Formatting** | How responses should be structured |
| **Boundary Conditions** | What the agent can and cannot do |
| **Do-Nots** | Explicit restrictions to prevent hallucination or off-topic responses |

### Topics

Topics **refine the scope** of an agent and are ideal for:

- **Organization-specific customizations** when using Oracle-seeded templates.
- Reusable instructions that apply across multiple agent teams.
- **Conversation management** — greetings, endings, follow-ups.
- **Environment-specific configuration** — regional rules, organizational policies.

### When to Use System Prompts vs. Topics

| Use System Prompts For | Use Topics For |
|---|---|
| Core role and persona definition | Reusable instructions across multiple agents/teams |
| Agent-specific tool usage instructions | Organization-specific customizations |
| Task-specific step-by-step workflows | Conversation management (greetings, follow-ups) |
| Agent-specific boundary conditions | Non-functional behavior (output formatting, out-of-context handling) |
| — | Functional information shared across agents |
| — | Sequence control applicable to multiple agents |

### LLM-Specific Optimization

- Different LLMs take instructions better in different formats.
- As you switch between models, you may need to **adjust how you write prompts and topics** to match the LLM's strengths.

---

## 7. Tips and Tricks

### 7.1 System Prompt Best Practices

1. **Be concise and specific about the role** — clearly define the agent's persona and purpose.
2. **Define the abilities** of the agent — what it can do and what types of questions it can handle.
3. **Set boundary conditions** — explicitly tell the agent what it **cannot** do.
4. **Keep it on task** — provide enough specificity to prevent hallucination.
5. **Include output formatting instructions** — specify bullets, tables, or structured formats.
6. **Include "do-nots"** — explicitly restrict undesirable behaviors.
7. **Don't overload with too many details** — give enough guidance but avoid over-specifying, which creates too much context for the LLM to process.

### 7.2 Topic Best Practices

- Use topics to define **sequences** that don't need to be in the system prompt.
- Provide **functional information** (business rules, domain knowledge).
- Provide **tool usage guidance** (when and how to use specific tools).
- Handle **non-functional concerns**: greetings, out-of-context query handling, output formatting.
- Think about **reusability** — if an instruction applies to multiple agents, put it in a topic.

### 7.3 Multi-Agent Configuration Tips

- **Supervisor responsibilities:**
  - Understand user intent.
  - Plan and break tasks into subtasks.
  - Route to the appropriate worker agent.
  - Orchestrate responses back to the user.
- **Worker agent responsibilities:**
  - Execute specific tasks using assigned tools.
- **Partitioning guidance:**
  - Divide agents by **business domain** (e.g., payroll, recruiting, benefits) for workflows spanning multiple areas.
  - Divide agents by **function** (e.g., data retrieval, recommendations, actions) for focused expertise.
- **Use topics** on the Supervisor to guide which agent to call for which types of questions.
- Provide **clear descriptions** for each worker agent — the supervisor uses these descriptions to decide which agent to delegate to.

### 7.4 Common Mistakes to Avoid

| Mistake | Consequence | Fix |
|---|---|---|
| **Being too vague** | Confuses the LLM; leads to hallucination or inconsistent answers | Clearly define the role, persona, and task scope |
| **No output formatting** | Agent produces unstructured text dumps | Specify desired format (bullets, tables, etc.) |
| **No boundary conditions** | Agent may answer off-topic questions or hallucinate | Tell it what it can do AND what it should NOT do |
| **Overloading with details** | Too much context for the LLM to process effectively | Provide enough guidance without over-specifying |
| **Not using topics for reusability** | Duplicated instructions across system prompts | Extract common instructions into topics |
| **Not pruning BO response fields** | Too much irrelevant data in context | Select only critical fields from business objects |

---

## 8. Human in the Loop

**Human in the Loop** is a feature that requires user approval before an agent performs certain actions:

- Can be set at the **task level** or **tool level**.
- When triggered, the agent **pauses and asks the user** within the session: "Will you allow me to use this tool and perform this action?"
- The user must approve (yes/no) before the agent proceeds.

**When to Use:**

- When performing **write operations** on Fusion business objects (POST, PATCH, DELETE).
- Any action that has **irreversible consequences**.
- Operations that require **human judgment or approval**.

---

## 9. Agent Team Configuration Details

When creating an agent team, the following properties are configured:

| Property | Description |
|---|---|
| **Agent Team Name** | Unique human-readable name |
| **Agent Team Code** | Unique identifier used in the Invoke API to reference the agent team |
| **Product Family** | The Fusion product family this team belongs to |
| **Product** | The specific product within the family |
| **Max Interactions** | Limits supervisor-worker or worker-tool interactions (e.g., 10) to prevent runaway processes |
| **Description** | Human-readable description of the agent team's purpose |
| **LLM Selection** | Which LLM to use (provided by Oracle in the current release) |
| **Starter Questions** | Pre-defined questions shown at the start of a chat experience |
| **Follow-up Questions** | Enable and prompt the LLM to generate follow-up questions using session history |
| **Security / Access Roles** | Define which roles can access the agent team (e.g., "AI Agent End User") |

### Max Interactions

- This setting tells the supervisor/agent: "Don't interact more than N times."
- Prevents **runaway processes** where the supervisor keeps trying different workers without success.
- Eventually delivers an **error condition** or the best available answer.

---

## 10. Testing, Deployment, and Invocation

### Testing in Agent Studio

- Run agents directly within Agent Studio during design time.
- A **debug panel (yellow box)** shows:
  - Which agent was selected.
  - Which tool was called.
  - Input and output of each tool call.
  - Decisions made by the agent team/agent.

### Agent Explorer (End-User Interface)

- Published agent teams are available via the **Agent Explorer**.
- Users see available agents based on their **assigned roles**.
- The end-user experience does **not** show intermediate debug steps — only the final response.

### Invoke API

Every agent team has an auto-generated **Invoke API**:

```
POST /invoke-async
Body: {
  "code": "<unique-agent-team-code>",
  "message": "<user question or task>"
}
```

- **Asynchronous call** — returns a job ID immediately.
- Poll using the job ID to check status and get the response.
- Also returns a **trace ID** for debugging.

### Where You Can Call the Invoke API From

- Fusion UI (embedded chat)
- VB (Visual Builder) pages
- HCM Journeys
- Backend processes
- External environments / custom applications

### Requirements for Invocation

- Proper credentials (OAuth token, bearer tokens).
- Must be an actual **Fusion user** with the appropriate role.
- The user context is used for security and functional access.

---

## 11. Security and Access Control

- **Role-based access** — agent teams are assigned to specific organizational roles.
- **Functional security** — enforced when accessing Fusion business objects.
- **User token** — the actual Fusion user token is used for all operations, ensuring data security.
- **Custom business objects** respect the same sandbox and security model.

---

## 12. LLM Integration

### Current State

- Oracle provides access to LLMs including **Llama** and **OpenAI** models.
- Some models are hosted within **OCI (Oracle Cloud Infrastructure)**.
- Access is managed through Oracle's agents interface.

### LLM Usage During Execution

LLMs are called **multiple times per user turn**:

1. **Planning Stage** — creates a plan to solve the task, evaluating available agents and tools.
2. **Tool Calling** — generates API call signatures, slots variables from context and session history.
3. **Response Generation** — takes tool outputs and creates a formatted response for the end user.

---

## 13. Future Roadmap

| Feature | Description |
|---|---|
| **Bring Your Own LLM (BYOLLM)** | Support for connecting your own LLM with your own API key |
| **MCP Tools Support** | Connect to Model Context Protocol (MCP) tools and call them remotely from within Fusion-hosted agents |
| **Additional Tools and Capabilities** | More tool types and agent capabilities will be added over time |

---

## 14. Reference Diagrams (Image Files)

The following diagrams are available in the same folder as supplementary visual references:

| File | Description |
|---|---|
| `Agent Architecture.jpg` | Overall architecture diagram showing Agent Studio, Agent Configuration, Fusion AI Services, and LLM interactions |
| `Key Concept - Agent Teams Workflow - Agent Worker.jpg` | Hierarchical relationship between Agent Teams, Workflows, and Worker Agents |
| `Key Concept - Tools - Topics.jpg` | How Tools and Topics fit into the agent framework |
| `Single Agent Multi Agent Teams.jpg` | Visual comparison of single-agent vs. multi-agent team patterns |
| `Business Object Tool.jpg` | Business Object Tool configuration and capabilities |
| `Business Object Action.jpg` | Business Object action operations (POST, PATCH, DELETE) |
| `Document Tool.jpg` | Document Tool (RAG) workflow — upload, chunk, vectorize, search |
| `Deep Link Tool.jpg` | Deep Link Tool configuration with query parameters |
| `External REST Tool.jpg` | External REST Tool setup with authentication and parameters |
| `Tools UseCase.jpg` | Overview of all tool types and their use cases |
| `Working with system prompts and Topics.jpg` | Guide to writing system prompts and topics |
| `Tips and Tricks.jpg` | System prompt best practices |
| `Tips and Tricks 2.jpg` | Topic best practices |
| `Tips and Tricks MultiAgent.jpg` | Multi-agent configuration best practices |
| `common mistakes.jpg` | Common mistakes to avoid when configuring agents |

---

## Quick-Reference: Building Your First Agent Team

### Step-by-Step Checklist

1. **Design your tools** — identify what data sources, APIs, and documents are needed.
2. **Create tools** — set up Business Object, Document, External REST, Deep Link tools as required.
3. **Create topics** — define reusable instructions for conversation management, tool usage, and organizational rules.
4. **Create agents (workers)** — give each agent a focused role with specific tools and topics.
5. **Write system prompts** — define role, persona, instructions, boundaries, output format, and do-nots.
6. **Create the agent team** — add agents, configure the supervisor (for multi-agent), set max interactions.
7. **Assign topics to the supervisor** — guide routing and delegation decisions.
8. **Test in Agent Studio** — use the debug panel to verify agent/tool selection and output quality.
9. **Publish** — make the agent team available via Agent Explorer.
10. **Configure security** — assign the appropriate roles for end-user access.
11. **Integrate** — use the Invoke API to embed the agent in Fusion UI, VB pages, or external systems.

---

*Document generated from the Oracle AI Agent Studio Technical Deep Dive session content and accompanying reference diagrams.*
