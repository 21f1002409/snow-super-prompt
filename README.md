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
│    ├─ Variable: v_json_payload          (mandatory)              │
│    ├─ Variable: v_table_schema          (optional helper)        │
│    └─ Variable: v_existing_records_xml  (optional helper)        │
│                                                                  │
│  Flow: "AI Super Prompt Executor Flow"  ◄── triggered on submit  │
│    └─ Script step calls SuperPromptProcessor.process()           │
│                                                                  │
│  Script Include: SuperPromptProcessor                            │
│    ├─ create_variable      → item_option_new                     │
│    ├─ update_variable      → item_option_new                     │
│    ├─ update_catalog_item  → sc_cat_item                         │
│    ├─ create_user          → sys_user                            │
│    ├─ update_user          → sys_user                            │
│    ├─ create_record        → any table  ◄── generic              │
│    ├─ update_record        → any table  ◄── generic              │
│    ├─ create_business_rule → sys_script         ◄── NEW          │
│    ├─ create_script_include → sys_script_include ◄── NEW         │
│    └─ create_client_script → sys_script_client  ◄── NEW          │
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
   >
   > *"Create a record in our custom table u_project with name 'Alpha'."*
2. For **generic table operations** (`create_record` / `update_record`) the AI
   will walk you through a guided conversation:
   - It asks which table you want to target.
   - It asks you to paste the table's **field dictionary** (XML or plain list)
     so it can identify correct field names and required fields.
   - It asks you to paste **existing records as XML** for format reference.
   - It then asks for the specific values you want and generates the payload.
3. Copy the JSON it produces.
4. Open the **AI Super Prompt Executor** catalog item in your ServiceNow
   Service Portal or Service Catalog.
5. Paste the JSON into the **JSON Payload** field and click **Submit**.
6. The flow triggers, the Script Include makes the change, and a work note is
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
| `create_record` | Creates a new record in **any** table | any fields valid for the target table |
| `update_record` | Updates an existing record in **any** table | `context.sys_id` **or** `context.lookup_field` + `context.lookup_value` required |
| `create_business_rule` | Creates a new Business Rule (`sys_script`) | `name`, `table_name`, `when` (before/after/async/display), `script`, optional: `action`, `order`, `filter_condition`, `description` |
| `create_script_include` | Creates a new Script Include (`sys_script_include`) | `name`, `script` (Class.create() pattern), optional: `api_name`, `access` (public/private), `description` |
| `create_client_script` | Creates a new Client Script (`sys_script_client`) | `name`, `table`, `type` (onLoad/onSubmit/onChange/onCellEdit), `script`, `field_name` required for onChange/onCellEdit |

---

## JSON Schema Reference

```json
{
  "operation_details": {
    "action": "<create_variable | update_variable | update_catalog_item | create_user | update_user | create_record | update_record>",
    "target_table": "<ServiceNow backend table name>"
  },
  "context": {
    "target_record_name": "<human-readable name / identifier of the record>",
    "sys_id": "<optional — sys_id of the record to update (update_record only)>",
    "lookup_field": "<optional — field name for record lookup (update_record only)>",
    "lookup_value": "<optional — value of lookup_field (update_record only)>"
  },
  "payload": {
    "<field_name_1>": "<value_1>",
    "<field_name_2>": "<value_2>"
  },
  "worknote_summary": "<professional summary added as a work note on the RITM>"
}
```

For `update_record`, the Script Include resolves the target record in this
priority order:
1. `context.sys_id` — direct lookup by sys_id (fastest, most precise).
2. `context.lookup_field` + `context.lookup_value` — lookup by any unique field.

Omit `sys_id`, `lookup_field`, and `lookup_value` for all other actions.

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

### Example — Create a record in any table (generic)

```json
{
  "operation_details": {
    "action": "create_record",
    "target_table": "u_project"
  },
  "context": {
    "target_record_name": "Alpha"
  },
  "payload": {
    "u_name": "Alpha",
    "u_status": "active",
    "u_owner": "abc123"
  },
  "worknote_summary": "Created a new record 'Alpha' in the custom table u_project."
}
```

### Example — Update a record in any table by lookup field (generic)

```json
{
  "operation_details": {
    "action": "update_record",
    "target_table": "change_request"
  },
  "context": {
    "target_record_name": "CHG0012345",
    "lookup_field": "number",
    "lookup_value": "CHG0012345"
  },
  "payload": {
    "assignment_group": "IT Operations"
  },
  "worknote_summary": "Updated assignment_group to 'IT Operations' on change request CHG0012345."
}
```

### Example — Create a Business Rule

```json
{
  "operation_details": {
    "action": "create_business_rule",
    "target_table": "sys_script"
  },
  "context": {
    "target_record_name": "Auto-set Urgency from Priority"
  },
  "payload": {
    "name": "Auto-set Urgency from Priority",
    "table_name": "incident",
    "when": "before",
    "action": "insert",
    "active": "true",
    "order": "100",
    "description": "Automatically sets urgency to match priority on new incidents.",
    "script": "(function executeRule(current, previous) {\n    if (current.priority == 1) {\n        current.urgency = 1;\n    }\n})(current, previous);"
  },
  "worknote_summary": "Created Business Rule 'Auto-set Urgency from Priority' on the incident table (before insert)."
}
```

### Example — Create a Script Include

```json
{
  "operation_details": {
    "action": "create_script_include",
    "target_table": "sys_script_include"
  },
  "context": {
    "target_record_name": "IncidentUtils"
  },
  "payload": {
    "name": "IncidentUtils",
    "api_name": "global.IncidentUtils",
    "active": "true",
    "access": "public",
    "description": "Utility methods for incident management.",
    "script": "var IncidentUtils = Class.create();\nIncidentUtils.prototype = {\n    initialize: function() {},\n\n    getHighPriorityCount: function() {\n        var gr = new GlideAggregate('incident');\n        gr.addQuery('priority', '1');\n        gr.addQuery('active', true);\n        gr.addAggregate('COUNT');\n        gr.query();\n        return gr.next() ? parseInt(gr.getAggregate('COUNT')) : 0;\n    },\n\n    type: 'IncidentUtils'\n};"
  },
  "worknote_summary": "Created Script Include 'IncidentUtils' with a public 'getHighPriorityCount' method."
}
```

### Example — Create a Client Script

```json
{
  "operation_details": {
    "action": "create_client_script",
    "target_table": "sys_script_client"
  },
  "context": {
    "target_record_name": "Show Resolution Notes on Resolved"
  },
  "payload": {
    "name": "Show Resolution Notes on Resolved",
    "table": "incident",
    "type": "onChange",
    "field_name": "state",
    "active": "true",
    "description": "Makes resolution_notes mandatory when incident state is set to Resolved.",
    "script": "function onChange(control, oldValue, newValue, isLoading) {\n    if (isLoading) { return; }\n    var isMandatory = (newValue == '6');\n    g_form.setMandatory('resolution_notes', isMandatory);\n    g_form.setVisible('resolution_notes', isMandatory);\n}"
  },
  "worknote_summary": "Created onChange Client Script 'Show Resolution Notes on Resolved' on the incident form, watching the state field."
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
