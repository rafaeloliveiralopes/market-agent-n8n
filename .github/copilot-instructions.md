Aqui está um `copilot-instructions.md` ideal para este projeto, com foco em modularidade, mudanças mínimas, segurança, desempenho, e um fluxo de trabalho Git disciplinado (branch por atividade, checkout, validação, add e commit no padrão Angular).

Copie e cole exatamente como arquivo `copilot-instructions.md`:

```md
# Copilot Instructions - Market Agent n8n

You are working in a production grade, modular n8n automation project for grocery order handling via WhatsApp.
Your priority is to preserve what is already working.

This repo is workflow driven. Most changes will touch JSON workflows exported from n8n, plus docs and SQL.
Treat workflows as critical infrastructure.

## 0) Source of Truth
1) docs/PRD.md is the primary specification.
2) If PRD and workflows conflict, prefer the current working workflows and propose a minimal fix aligned to PRD.
3) Never invent features. If unclear, inspect the existing workflow JSON nodes by name.

Key workflows:
- workflows/Agente principal - Atendente mercado.json (Supervisor "Ana")
- workflows/Agente de Catálogo e Preços.json (catalog validation tool)

## 1) Architecture Summary
This is an n8n multi agent system coordinated by a supervisor pattern:
- Ana (Supervisor): orchestrates the conversation and tools
- Catalog Agent (toolWorkflow): mandatory validation of products and brands
- RAG: vector store retrieval from Supabase pgvector
- Training: updates the knowledge base (embedded in workflows)

Primary stack:
WhatsApp (Evolution API) -> n8n -> OpenAI (chat, embeddings, audio, vision) -> Supabase (Postgres + pgvector)
Redis is used for message batching/queue.

## 2) Non Negotiable Behavior Rules
These rules are hard requirements. Do not weaken them.

Conversation rules:
- ONE question per WhatsApp message. Never combine multiple requests.
- NEVER confirm a product or brand without calling the toolWorkflow named exactly: consultar_catalogo_precos
- NEVER invent prices, stock, delivery time, or availability
- FORBIDDEN: use words related to "felicidade"

Business rule:
- Minimum order value for delivery: R$ 120,00
  If total is below minimum, Ana must inform and suggest adding items.

## 3) Mandatory Order Flow (Sequence)
Ana must collect and confirm data in this order. Do not skip steps.

1) Products (must validate via consultar_catalogo_precos)
2) Brand preference (if applicable)
3) Quantity
4) Variations (size, flavor, volume, only if needed)
5) Customer full name
6) Full address
7) Phone confirmation
8) Substitution preference (when an item is missing)
9) Payment method
10) Change (if cash)

Timeout behavior must be preserved:
- Payment confirmation retry after 15 minutes
- Address confirmation retry after 20 minutes
- Phone confirmation retry after 15 minutes (and inform that delivery may call)

## 4) Catalog Validation Contract
The catalog toolWorkflow must be called for every product or brand mention.

Input:
{ "query": "refrigerante coca cola", "categoria": "bebidas" }

Output contract (must stay consistent with the current workflow):
{
  "encontrou": boolean,
  "total": number,
  "produtos": [
    {
      "id": any,
      "nome": string,
      "categoria": string,
      "tipo": string,
      "marca": string,
      "preco": number,
      "preco_formatado": string,
      "tamanho": string,
      "estoque_disponivel": number,
      "em_estoque": boolean
    }
  ],
  "estatisticas": {
    "preco_minimo": number,
    "preco_maximo": number,
    "preco_medio": number,
    "categorias_encontradas": string[]
  },
  "mensagem": string
}

If encontrou is false:
1) Inform unavailability without blaming the user
2) Suggest alternatives returned by the tool
3) Show price range using estatisticas (min and max)
4) Ask a single question: if they want one of the alternatives

Do not change tool name, tool schema, or output fields unless PRD explicitly requires it.

## 5) RAG Rules (Safety and Accuracy)
RAG is allowed only to answer store policy, delivery rules, operational details, and documented FAQs.
RAG is not allowed to replace catalog validation.

Hard rules:
- Use ONLY the output of the vector store tool as factual source
- If no relevant retrieval: say you could not find that information
- Never hallucinate policies, prices, or availability
- Never copy large chunks verbatim. Keep responses short and operational

## 6) Media Handling Rules
Do not break existing media pipeline nodes.

- Audio:
  Base64 -> file (.ogg) -> transcription -> normal text flow

- Image:
  Base64 -> file (.png) -> vision analysis -> normal text flow
  Ask for clarification if the image does not clearly identify a product

- Text:
  Direct flow

## 7) Performance Guidelines (No Bottlenecks)
Keep the system fast and stable.

- Avoid adding extra OpenAI calls in the hot path
- Avoid repeated Supabase queries for the same intent
- Keep Redis batching logic intact, it prevents message flooding
- Do not increase context window or RAG chunk size without a clear reason and test
- Prefer deterministic logic for message splitting and formatting when possible
- Do not refactor nodes unrelated to the task (no drive by cleanup)

## 8) Security Guidelines
- Do not log secrets, tokens, API keys, or full payloads containing PII
- Minimize PII exposure in logs and debug nodes (phone, address)
- Never hardcode credentials in JSON
- When suggesting changes, include a rollback plan and the smallest possible diff
- Prefer least privilege database roles and restricted Supabase keys inside n8n credentials

## 9) n8n Workflow Editing Rules (Critical)
- Prefer editing workflows in n8n UI and exporting the JSON
- Avoid manual JSON edits unless absolutely necessary
- If manual edits are required:
  - Change only the targeted node parameters
  - Preserve node names, ids, and connections structure
  - Validate JSON formatting after edits
- Never rename nodes that are referenced by expressions unless the task explicitly requires it
- Never rename toolWorkflow name consultar_catalogo_precos

## 10) Git Workflow Rules (Branch Check, Create, Checkout, Commit)

Every task must be isolated in its own branch.
Never work directly on main.

### 10.1 Before starting any activity
Always verify:
1) Current branch
2) Whether an appropriate branch already exists for the requested activity
3) Whether you should switch branches before making changes

Commands:
- Check current branch:
  git branch --show-current

- List local branches:
  git branch

- List remote branches (if needed):
  git branch -r

- Search for a branch by keyword:
  git branch --all | grep -i "<keyword>"

### 10.2 Branch selection rule
If an appropriate branch already exists for the activity:
- Switch to it and continue work there.

If no appropriate branch exists:
- Create a new branch using the correct prefix and switch to it.

Allowed prefixes:
feat/, fix/, chore/, refactor/, docs/, ci/, test/, perf/

### 10.3 Example: feature already has a branch
Scenario: You need to implement "catalog alternatives improvements".
First check if a branch exists:

git branch --all | grep -i "catalog"

If you find something like:
feat/catalog-alternatives

Then:
git checkout feat/catalog-alternatives

Now you can implement the requested change in this branch.

### 10.4 Example: no branch exists, create it
Scenario: You need to fix "payment confirmation retry logic".
First check:

git branch --all | grep -i "payment"

If nothing relevant exists, create and switch:
git checkout -b fix/payment-confirmation-retry

### 10.5 Rule for every new activity (mandatory)
Before doing a new activity, always:
1) Check the current branch
2) Decide if that branch is still the most appropriate for the new activity
3) If not appropriate, find an existing better branch or create a new one

Example:
- You are currently on:
  git branch --show-current
  feat/catalog-alternatives

- A new request arrives: "update PRD documentation"
This is not appropriate to do inside feat/catalog-alternatives.
Check existing docs branches:

git branch --all | grep -i "docs"

If found:
docs/prd-executive-update
Then switch:
git checkout docs/prd-executive-update

If not found:
git checkout -b docs/prd-executive-update

### 10.6 Implementation discipline
1) Ensure working tree is clean:
   git status

2) If you must pull updates from main, do it safely:
   git checkout main
   git pull --rebase

3) Return to the correct activity branch:
   git checkout <activity-branch>

4) Make minimal change, validate, then stage and commit.

### 10.7 Commit rules
Use Angular commit message format:
type(scope): message

Rules:
- The first letter after ":" must be lowercase
- Do not end the message with a period

Example:
git commit -m "fix(workflow): preserve payment confirmation timeout behavior"

## 11) Definition of Done (DoD) for Any Change
A change is done only if all are true:
- Workflows still import in n8n without errors
- consultar_catalogo_precos remains mandatory for product and brand mentions
- No new hallucination paths were introduced
- WhatsApp message format remains compatible
- Timeouts (15 min and 20 min) behave the same unless PRD says otherwise
- Performance impact is neutral or improved
- A rollback path exists (revert commit or restore prior exported workflow JSON)

## 12) How to Propose Changes (Required Format)
Before editing anything, always output:

1) What PRD section and what workflow nodes are impacted (by node name)
2) Minimal patch plan (bullet list)
3) Risk list (what could break)
4) Test plan (3 to 8 concrete test cases)

Only then apply changes.

```