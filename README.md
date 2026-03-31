# snow-super-prompt

A **no-API, no-integration** ServiceNow automation toolkit that uses any free
web-based AI chatbot (Microsoft Copilot, ChatGPT, Gemini, etc.) as an
off-platform "translation engine." Users describe the ServiceNow change they
need in plain English; the AI produces a structured JSON payload; the user
pastes that payload into a ServiceNow Catalog Item; a Flow + Script Include
automatically performs the requested change and adds a work note to the RITM.

---

## Architecture

```
┌─────────────────────┐       Plain-English request
│   User / Admin      │──────────────────────────────►  AI Chatbot
│                     │◄──────────────────────────────  (Copilot / ChatGPT)
└─────────────────────┘       JSON payload
          │
          │  Paste JSON into
          ▼  Catalog Item
┌──────────────────────────────────────────────────────────────────┐
│                     ServiceNow Instance                          │
│                                                                  │
│  Catalog Item: "AI Super Prompt Executor"                        │
│    └─ Variable: v_json_payload  (multi-line text, mandatory)     │
│                                                                  │
│  Flow: "AI Super Prompt Executor Flow"  ◄── triggered on submit  │
│    └─ Script step calls SuperPromptProcessor.process()           │
│                                                                  │
│  Script Include: SuperPromptProcessor                            │
│    ├─ create_variable      → item_option_new                     │
│    ├─ update_variable      → item_option_new                     │
│    ├─ update_catalog_item  → sc_cat_item                         │
│    ├─ create_user          → sys_user                            │
│    └─ update_user          → sys_user                            │
│                                                                  │
│  RITM work note  ◄── added automatically with AI summary        │
└──────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
snow-super-prompt/
├── super_prompt.txt                          ← AI system prompt (paste into chatbot)
├── update_set/
│   └── snow_super_prompt_update_set.xml      ← Single importable ServiceNow Update Set
└── src/                                      ← Individual source files (reference)
    ├── script_includes/
    │   └── SuperPromptProcessor.js
    ├── catalog_item/
    │   └── super_prompt_catalog_item.xml
    └── flow/
        └── super_prompt_flow.xml
```

---

## Quick-Start Guide

### Step 1 — Import the Update Set

1. Log in to your ServiceNow instance as an **administrator**.
2. Navigate to **System Update Sets › Retrieved Update Sets**.
3. Click **Import Update Set from XML** and upload
   `update_set/snow_super_prompt_update_set.xml`.
4. Open the newly created update set, click **Preview Update Set**, resolve any
   conflicts, then click **Commit Update Set**.

### Step 2 — Post-import configuration

| Task | Where |
|------|-------|
| Assign the catalog item to a Service Catalog & category | **Service Catalog › Maintain Items › AI Super Prompt Executor** |
| Activate the flow | **Flow Designer › AI Super Prompt Executor Flow › Activate** |
| Confirm the Script Include is active | **System Definition › Script Includes › SuperPromptProcessor** |

### Step 3 — Initialise the AI Chatbot

1. Open `super_prompt.txt` and copy its entire contents.
2. Paste it into a **new chat** in your AI chatbot (Copilot, ChatGPT, etc.).
3. The AI will respond: *"Ready to generate ServiceNow JSON payloads."*

### Step 4 — Use It

1. Tell the AI what change you need, e.g.:
   > *"Create a mandatory single-line text variable called 'Cost Centre' on the
   > catalog item 'New Laptop Request'."*
2. Copy the JSON it produces.
3. Open the **AI Super Prompt Executor** catalog item in your ServiceNow
   Service Portal or Service Catalog.
4. Paste the JSON into the **JSON Payload** field and click **Submit**.
5. The flow triggers, the Script Include makes the change, and a work note is
   added to the generated RITM.

---

## Supported Actions

| `action` value | What it does | Key payload fields |
|---|---|---|
| `create_variable` | Creates a new variable on a catalog item | `type`, `question_text`, `name`, `mandatory`, `reference` |
| `update_variable` | Updates an existing variable | `name` (required to locate), plus any field to change |
| `update_catalog_item` | Updates fields on a catalog item record | any `sc_cat_item` field |
| `create_user` | Creates a new `sys_user` record | `first_name`, `last_name`, `email`, `user_name` |
| `update_user` | Updates an existing `sys_user` record | `email` or `user_name` required to locate |

---

## JSON Schema Reference

```json
{
  "operation_details": {
    "action": "<create_variable | update_variable | update_catalog_item | create_user | update_user>",
    "target_table": "<ServiceNow backend table name>"
  },
  "context": {
    "target_record_name": "<human-readable name of the parent record>"
  },
  "payload": {
    "<field_name_1>": "<value_1>",
    "<field_name_2>": "<value_2>"
  },
  "worknote_summary": "<professional summary added as a work note on the RITM>"
}
```

### Example — Create a reference variable

```json
{
  "operation_details": {
    "action": "create_variable",
    "target_table": "item_option_new"
  },
  "context": {
    "target_record_name": "New Hardware Request"
  },
  "payload": {
    "type": "Reference",
    "reference": "sys_user",
    "question_text": "Requested For",
    "name": "requested_for",
    "mandatory": "true"
  },
  "worknote_summary": "Created a mandatory reference variable 'Requested For' (sys_user) on the 'New Hardware Request' catalog item."
}
```

---

## Script Include: SuperPromptProcessor

**Location:** `src/script_includes/SuperPromptProcessor.js`
**API name:** `global.SuperPromptProcessor`

| Method | Signature | Description |
|--------|-----------|-------------|
| `process` | `process(jsonInput, ritmGr)` | Main entry point. Parses JSON, validates schema, dispatches to the correct handler, and adds a work note. Returns `{ success, message }`. |

All GlideRecord operations use `setValue` so that field-level ACLs and
business rules are respected in the same way as a manual update.

---

## Extending with New Actions

1. Add the new `action` string to `super_prompt.txt` under "Action Types".
2. Add a new `case` in `SuperPromptProcessor.prototype.process`.
3. Implement the handler method following the same pattern as the existing ones.
4. Export a new Update Set from your dev instance and replace
   `update_set/snow_super_prompt_update_set.xml`.

---

## License

MIT
