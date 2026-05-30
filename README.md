# Open WebUI test task

This is my fork of Open WebUI for the developer test task. I kept the changes small on purpose: one clear visual branding change, one visible UI element, and a short write-up for the RAG part of the task.

## Links

- GitHub fork: [GitHub](https://github.com/Web-Master-1022/Open-Web-UI)

## Running it locally

I used Docker for the local run. From the project root:

```powershell
docker build --build-arg USE_SLIM=true -t freiheit-open-webui:local .
docker rm -f open-webui
docker run -d -p 3000:8080 -e HF_HUB_OFFLINE=1 -v open-webui:/app/backend/data --name open-webui --restart always freiheit-open-webui:local
```

Then open:

```text
http://localhost:3000
```

I used `HF_HUB_OFFLINE=1` because the default image can spend a long time downloading Hugging Face files on first startup. For this task the important part is the local Open WebUI instance and the UI changes.

Useful commands:

```powershell
docker logs -f open-webui
docker stop open-webui
docker start open-webui
```

If Ollama runs on the Windows host, I would start it with this instead:

```powershell
docker rm -f open-webui
docker run -d -p 3000:8080 -e OLLAMA_BASE_URL=http://host.docker.internal:11434 -e HF_HUB_OFFLINE=1 -v open-webui:/app/backend/data --name open-webui --restart always freiheit-open-webui:local
```

## UI changes

I changed the UI in three places:

- `src/app.css`: added a green/emerald accent to the app background and sidebar.
- `src/lib/components/layout/InternalBrandBanner.svelte`: added a small banner with the text `Freiheit Media - Internal LLM`.
- `src/routes/(app)/+layout.svelte`: mounted the banner inside the main authenticated app layout.

The goal was to make the customization obvious in a quick demo without changing core Open WebUI behavior.

## Part B: RAG answer quality and customer safety

Assumption: each customer has a separate Knowledge Base, and the active model is connected to exactly one customer Knowledge Base at a time.

### System prompt

```text
You are answering questions for one customer only.

Use only the currently active Knowledge Base. Do not use information from other customers, previous chats, general knowledge, or guesses.

Before answering, decide whether the active Knowledge Base contains:
1. enough information to answer,
2. only partial information,
3. no relevant information.

If the information is complete, answer directly and cite the smallest useful source name, for example: "Based on Onboarding Guide - 2024".

If the information is partial, answer only the part that is supported. Then clearly say what is missing.

If the information is not present, say that the active Knowledge Base does not contain the answer. Do not fill the gap with assumptions.

If documents conflict, say that the Knowledge Base contains conflicting information and name the conflicting documents.

Do not invent policies, prices, contacts, dates, SLAs, legal terms, security controls, or technical steps.
```

### Example Knowledge Base

For the examples below, I assume the active Knowledge Base contains these documents:

- `Onboarding Guide - 2024`: new employees must complete security training within 7 days, set up MFA before production tool access, and contact IT through the helpdesk portal.
- `Support SLA - Enterprise`: enterprise customers receive a first response within 4 business hours for priority 2 incidents.
- `Billing FAQ - 2024`: invoices are sent monthly to the billing contact on file.

### Example 1: complete answer

Question:

```text
How quickly do enterprise customers get a first response for a priority 2 incident?
```

Expected answer:

```text
Enterprise customers receive a first response within 4 business hours for priority 2 incidents. Based on Support SLA - Enterprise.
```

### Example 2: partial answer

Question:

```text
How do new employees get production access, and who approves it?
```

Expected answer:

```text
New employees must complete security training within 7 days and set up MFA before getting access to production tools. The Knowledge Base does not say who approves the access, so that part needs to be checked with IT or the access owner. Based on Onboarding Guide - 2024.
```

### Example 3: no answer in the Knowledge Base

Question:

```text
What is the customer's refund policy for annual contracts?
```

Expected answer:

```text
The active Knowledge Base does not contain the refund policy for annual contracts. I cannot answer that from the available documents.
```

### Why I would use this approach

The main risk in a customer-specific RAG setup is that the assistant sounds confident even when the retrieved context is incomplete. The prompt above makes the model handle missing information explicitly instead of guessing.

It also helps prevent customer data leakage. The model is told to use only the active Knowledge Base, so it should not mix policies, contacts, or terms from another customer.

Typical failure modes this reduces:

- answering from general model knowledge instead of the customer's documents
- mixing up customers
- inventing missing SLAs, contacts, or approval steps
- treating partial information as complete

Later I would improve this with structured output, for example `status`, `answer`, `missing_info`, and `sources`, plus retrieval filters that enforce the customer ID before the model receives any context.