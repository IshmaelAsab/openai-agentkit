# Building an AI Workflow Agent to Review & Update SRs (Service Requests)

This document is a step-by-step tutorial and design guide (in Markdown) for creating an AI workflow agent to help product support engineers review and update SRs (Service Requests) by recognizing product associations, validating fields, suggesting updates, and in some cases performing the update via an API. It focuses on practical architecture, schemas, function/tool definitions, prompt design, examples, and operational concerns.

Audience: support engineers, ML/AI engineers, and platform developers.

---

## 1. Goals and scope

- Automatically classify which product(s) an SR belongs to.
- Validate SR metadata against product-specific schemas.
- Suggest updates (priority, component, product version, attachments, tags, next action).
- Optionally perform safe updates by calling the SR management API with operator approval.
- Provide audit trail, confidence scores, and explainability for suggested changes.

Scope decisions to make up front:
- Human-in-the-loop vs fully automated updates.
- Which fields can the agent update automatically (e.g., tags) and which require approval (e.g., priority)?
- Latency and throughput constraints (real-time vs batch review).

---

## 2. High-level architecture

Components:
1. Ingest: fetch SRs from ticketing system (webhooks, polling). 
2. Preprocessor: sanitize text, extract structured fields, enrich with product metadata.
3. LLM Agent: runs classification, validation, suggestions. Uses function/tool calls for deterministic tasks (lookup product taxonomy, call rules engine).
4. Decisioning & Workflow: apply policy (auto-apply, suggest, escalate) and log result.
5. Action: call SR system APIs to update (with approval if required).
6. Feedback & Retraining: capture user corrections to refine models and prompts.

Diagram (text):

SR System -> Ingest -> Preprocessor -> LLM Agent + Tools -> Decisioning -> SR System (update) & UI for reviewer -> Feedback -> Retraining

---

## 3. Data & Schemas

Design canonical JSON schemas. Keep them small and explicit so the agent can validate easily.

Example: SR canonical schema (JSON Schema style):

```json
{
  "sr_id": "SR-12345",
  "summary": "Short text summary",
  "description": "Long description text",
  "reported_product": "raw user text or field",
  "product": { "id": "product-id", "name": "Product Name", "version": "1.2.3" },
  "component": "component-name",
  "priority": "P1|P2|P3|P4",
  "status": "Open|In Progress|Resolved|Closed",
  "tags": ["tag1","tag2"],
  "attachments": [{"name":"log.txt","type":"text/plain","url":"..."}],
  "reporter": {"name":"Alice","email":"alice@example.com"},
  "created_at":"2025-11-01T14:30:00Z",
  "last_updated":"2025-11-07T10:00:00Z"
}
```

Product taxonomy schema (simplified):

```json
{
  "product_id": "prd-001",
  "name": "Acme Storage",
  "aliases": ["AcmeStor","ASM"],
  "components": ["core","ui","cli","agent"],
  "supported_versions": ["1.x","2.x","3.0.0"],
  "owner_team": "storage-core"
}
```

Why JSON schemas?
- They allow deterministic validation and help the agent decide which fields are missing/incorrect.
- They’re easy to store and query.

---

## 4. Tools / Functions the agent should have

Encapsulate deterministic operations as tools/functions the LLM can call. Example functions:

- fetch_sr(sr_id): Get canonical SR JSON (from ticket system).
- classify_product(text, metadata): Return product_id(s) + confidence + reasons.
- validate_sr(sr_json, product_id): Return list of schema mismatches and suggested fixes.
- suggest_updates(sr_json, product_schema): Return updates with explanations + confidence.
- apply_updates(sr_id, updates, dry_run=true): Apply updates via SR API (with dry-run mode).
- fetch_product_docs(product_id, query): Return relevant docs or KB snippets.
- run_rules_engine(sr_json): Run deterministic business rules (e.g., if "disk" in text and product=storage => escalate).
- log_action(audit_entry): Append to audit log.

Define function signatures in your platform's function-calling format so the LLM can call them directly (if using a model with function calling). Example for JSON/HTTP or OpenAI function-calling style:

```json
{
  "name": "classify_product",
  "description": "Return product classification for SR text",
  "parameters": {
    "type": "object",
    "properties": {
      "sr_id": {"type":"string"},
      "text": {"type":"string"},
      "metadata": {"type":"object"}
    },
    "required": ["sr_id","text"]
  }
}
```

Notes:
- Keep deterministic functions small and auditable.
- Prefer returning structured data (ids, scores, explanations).

---

## 5. Prompt & System design for the LLM agent

Design a system prompt that establishes the agent role, constraints (safety rules), and tools available. Example points to include:

- Role: "You are a product support assistant that classifies and proposes updates to SRs."
- Allowed actions: list the function names the model may call.
- Safety constraints: do not change priority without > 0.9 confidence and explicit approval; always provide confidence scores and explain reasoning; never expose internal PII in outputs.
- Output format: require JSON response with specific fields (decisions, confidence, reasoning, actions).

Example system instruction (short):

"You are an agent that receives a canonical SR JSON. You may call these functions: classify_product, validate_sr, suggest_updates, fetch_product_docs, apply_updates. For every suggested change provide: field, current_value, suggested_value, confidence (0-1), rationale. If confidence < 0.7, label as 'suggestion' — do not auto-apply. If confidence >= 0.9 and change is low-risk (tags, component), you may auto-apply with apply_updates(dry_run=false) provided audit approval flag is set. Always log actions."

Prompt engineering tips:
- Use few-shot examples inside system or user prompt so model learns preferred output structure.
- Emphasize concise, structured JSON outputs to minimize parsing errors.
- Provide product taxonomy and a short KB excerpt when available to the prompt.

---

## 6. Example agent workflow (conversation + function calling logic)

1. Event: New SR created or SR changed. Ingest fetches SR JSON and calls LLM with system prompt + SR JSON.
2. LLM calls classify_product(text) function.
3. LLM receives classification result. If ambiguous (>1 product with similar confidence), call fetch_product_docs for clarifying info.
4. LLM calls validate_sr(sr_json, product_id) to find schema mismatches.
5. LLM calls suggest_updates(sr_json, product_schema) to propose concrete changes.
6. LLM compiles recommended changes, confidence scores, and rationales.
7. Decision logic: if all suggested changes are low-risk and above confidence threshold and auto-apply policy enabled, call apply_updates; otherwise, prepare UI card for human reviewer.
8. Log audit and store outcome.

---

## 7. Example JSON exchange (mock)

Input SR (simplified):

```json
{
  "sr_id":"SR-9001",
  "summary":"Storage node failing writes intermittently",
  "description":"We see I/O timeouts on /dev/sdb. Host: node-12. Firmware 1.0.2. Happened after upgrade.",
  "reported_product":"AcmeStor",
  "product": null,
  "component":"",
  "priority":"",
  "tags":[]
}
```

Agent result (structured):

```json
{
  "sr_id":"SR-9001",
  "classifications":[{"product_id":"prd-001","product_name":"Acme Storage","confidence":0.98,"reason":"mentions I/O, disk, firmware, storage node"}],
  "validations":[{"field":"product","issue":"missing","suggested":"prd-001"},{"field":"priority","issue":"missing","suggested":"P1","confidence":0.72, "rationale":"I/O timeouts on storage nodes affecting writes => high impact"}],
  "suggested_updates":[{"field":"product","from":null,"to":{"id":"prd-001","name":"Acme Storage"},"confidence":0.98},{"field":"tags","from":[],"to":["storage","io-timeout"],"confidence":0.95}],
  "auto_apply": ["tags"],
  "explain":"Detected product and added tags. Priority suggested as P1 but below auto-apply threshold; recommend human review." 
}
```

---

## 8. Implementation snippets

Below are short examples showing how to wire LLM + function tools. Adapt to your stack.

Python (pseudocode, generic):

```python
# Pseudocode
sr = fetch_sr_from_ticketing(sr_id)
# system_prompt defined earlier
llm_input = {
  "system": SYSTEM_PROMPT,
  "user": f"Review this SR: {json.dumps(sr)}"
}
# Call model with function calling enabled; model may request functions.
response = call_llm_model(llm_input, functions=FUNCTION_DEFS)
# If model returned a function call, execute it and feed result back.
# Iterate until model returns final JSON output.
# Parse output and run apply_updates or create UI card.
```

Node.js (outline):

```js
const sr = await fetchSr(srId);
const response = await client.chat.completions.create({
  model: 'gpt-XYZ',
  messages: [{role:'system', content: SYSTEM_PROMPT}, {role:'user', content: JSON.stringify(sr)}],
  functions: FUNCTION_DEFINITIONS,
  function_call: 'auto'
});
// Handle function calls similarly to Python flow.
```

Important: implement retries and deterministic fallbacks for function calls. If a function fails, inform the user and mark SR for manual review.

---

## 9. Tools & libraries to consider

- LLM provider SDK (OpenAI or other) with function-calling support.
- A rules engine for deterministic business rules (Drools, durable_rules, or simple internal engine).
- Text processing & embedding libraries (spaCy, HuggingFace, sentence-transformers) if you want vector search for KB.
- Vector DB (Milvus, Pinecone, Weaviate, or open-source alternatives) for KB similarity lookups.
- Observability: Sentry, Elastic, Prometheus for logging and metrics.
- CI/CD pipelines for prompt and function changes (store prompts and function definitions in version control).

---

## 10. Testing, evaluation & metrics

Metrics to track:
- Classification accuracy (product assigned vs gold-label).
- Precision/recall for suggested field updates.
- Auto-apply error rate (how often auto-applied changes were reverted).
- Human reviewer satisfaction and time saved.

Testing approach:
- Start in "suggest-only" mode and collect human feedback for 2-4 weeks.
- Label a validation set of SRs and compute accuracy for product classification & field suggestions.
- Run A/B tests (agent suggestions vs no-agent).

Feedback loop:
- Capture reviewer actions (accepted/modified/rejected) to create supervised training data.
- Retrain classification models or refine prompts and rules periodically.

---

## 11. Safety, audit & compliance

- Add immutable audit logs containing: SR id, who/what suggested change, timestamp, function outputs, LLM outputs, and final applied changes.
- Approvals: require explicit human approval for high-risk fields or changes that affect billing, SLAs, or data deletion.
- PII handling: redact or limit sensitive fields when sending to an external LLM unless you have a compliant hosting arrangement.
- Rate limiting and throttling for API calls.

---

## 12. Example function definitions (detailed)

classify_product
- Input: {sr_id, text, metadata}
- Output: [{product_id, name, confidence, evidence}]
- Implementation notes: ensemble of LLM classification + lightweight ML model or embedding-KNN over product docs.

validate_sr
- Input: {sr_json, product_id}
- Output: [{field, issue_type (missing/invalid/mismatch), expected_format, suggestion}]

suggest_updates
- Input: {sr_json, product_schema}
- Output: [{field, from, to, confidence, rationale, resource_refs}]

apply_updates
- Input: {sr_id, updates, dry_run}
- Output: {applied: [fields], failed: [errors], audit_id}

---

## 13. Example prompts & few-shot examples

System:
"You are Support-AI. For every SR, produce JSON with keys: classifications, validations, suggested_updates. Use the functions provided when helpful. Give confidence [0-1]."

Few-shot:
- Provide 2-3 short examples of SR text and expected structured outputs so the model learns the mapping.

---

## 14. Deployment & rollout plan

1. Implement ingest, preprocessor, and function scaffolding.
2. Start LLM in suggestion-only mode (no apply_updates). Provide UI for reviewers to accept suggestions.
3. Monitor metrics, iterate on prompts, thresholds, and rules.
4. After stable performance and low revert rate, enable auto-apply for low-risk changes with strict auditing.
5. Expand coverage to more product lines and edge cases.

---

## 15. Operational checklist

- [ ] Define canonical SR and product schemas.
- [ ] Implement fetch_sr and apply_updates APIs with dry_run.
- [ ] Build rules engine for business constraints.
- [ ] Create LLM system prompt and function definitions.
- [ ] Instrument logging, metrics, and auditing.
- [ ] Run pilot in suggestion-only mode.
- [ ] Collect labeled feedback for 4 weeks.
- [ ] Decide on auto-apply thresholds and policies.

---

## 16. Next steps & customization ideas

- Use embeddings+vector search to surface relevant KB articles and include them in reasoning.
- Build a UI component (Slack, Jira sidebar, internal portal) that displays suggested updates as review cards.
- Add multi-language support for SRs.
- Add escalation automation for SLA breaches.
- Use model grounding: include product docs as context and prefer deterministic tool outputs.

---

## 17. Appendix: Sample small dataset for offline testing

Create a CSV with columns: sr_id, summary, description, labeled_product_id, labeled_priority, labeled_component, final_tags. Use it to compute accuracy offline.

---

If you'd like, I can:
- Generate concrete function JSON definitions for your platform (OpenAI function-calling format, or REST endpoint signatures).
- Create example prompt templates with few-shot examples tuned to your product taxonomy.
- Scaffold a simple demo in Python or Node that wires the LLM and the functions.

Tell me which you'd like next and share any existing schemas or sample SRs you already have so I can make the examples concrete.
