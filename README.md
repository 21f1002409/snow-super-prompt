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
│    ├─ create_business_rule → sys_script                          │
│    ├─ create_script_include → sys_script_include                 │
│    └─ create_client_script → sys_script_client                   │
│                                                                  │
│  RITM work note ◄── AI summary + links to ALL records affected   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
snow-super-prompt/
├── super_prompt.txt                          ← AI system prompt (paste into chatbot)
├── update_set/
│   └── snow_super_prompt_update_set.xml      ← Single importable ServiceNow Update Set
├── scripts/
│   └── deploy_via_table_api.py               ← Alternative: deploy via REST Table API
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

### Option A — Import the Update Set *(recommended)*

1. Log in to your ServiceNow instance as an **administrator**.
2. Navigate to **System Update Sets › Retrieved Update Sets**.
3. Click **Import Update Set from XML** and upload
   `update_set/snow_super_prompt_update_set.xml`.
4. Open the newly created update set, click **Preview Update Set**, resolve any
   conflicts, then click **Commit Update Set**.

### Option B — Deploy directly via Table API *(if the update set import fails)*

Use the bundled Python script to create or update every component directly in
the instance without touching update sets.

```bash
# Install the only dependency
pip install requests

# Deploy using CLI flags
python scripts/deploy_via_table_api.py \
    --instance  https://dev12345.service-now.com \
    --username  admin \
    --password  <password>

# Or use environment variables
export SN_INSTANCE=https://dev12345.service-now.com
export SN_USERNAME=admin
export SN_PASSWORD=<password>
python scripts/deploy_via_table_api.py

# Preview what would happen without making any changes
python scripts/deploy_via_table_api.py --dry-run

# Target a specific application scope
python scripts/deploy_via_table_api.py --scope x_my_app_12345
```

The script creates or updates:
- The **SuperPromptProcessor** Script Include (`sys_script_include`)
- The **AI Super Prompt Executor** Catalog Item (`sc_cat_item`)
- All three catalog variables (`item_option_new`)

At the end it prints a summary with a direct link to each record.

### Step 2 — Post-deployment configuration

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
   >
   > *"Create a Script Include called 'ProjectUtils' and a Business Rule
   > 'Set Project Active' on u_project."*
2. For **create_*** actions the AI will ask which **application scope** the new
   record should belong to (e.g. `global` or `x_my_app_12345`).  Press Enter
   to keep the default (`global`).
3. For **generic table operations** (`create_record` / `update_record`) the AI
   will walk you through a guided conversation:
   - It asks which table you want to target.
   - It asks you to paste the table's **field dictionary** (XML or plain list)
     so it can identify correct field names and required fields.
   - It asks you to paste **existing records as XML** for format reference.
   - It then asks for the specific values you want and generates the payload.
4. Copy the JSON it produces.
5. Open the **AI Super Prompt Executor** catalog item in your ServiceNow
   Service Portal or Service Catalog.
6. Paste the JSON into the **JSON Payload** field and click **Submit**.
7. The flow triggers, the Script Include makes the change, and a work note is
   added to the generated RITM with links to **every record affected**.

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

### Single Operation

```json
{
  "operation_details": {
    "action": "<create_variable | update_variable | update_catalog_item | create_user | update_user | create_record | update_record | create_business_rule | create_script_include | create_client_script>",
    "target_table": "<ServiceNow backend table name>",
    "scope": "<optional — application scope for NEW records, e.g. global or x_my_app_12345>"
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

### Batch Mode (multiple records in one submission)

```json
{
  "operations": [
    {
      "operation_details": { "action": "<action1>", "target_table": "<table1>", "scope": "<optional>" },
      "context": { "target_record_name": "<name1>" },
      "payload": { "<field>": "<value>" },
      "worknote_summary": "<per-operation summary>"
    },
    {
      "operation_details": { "action": "<action2>", "target_table": "<table2>" },
      "context": { "target_record_name": "<name2>" },
      "payload": { "<field>": "<value>" },
      "worknote_summary": "<per-operation summary>"
    }
  ],
  "worknote_summary": "<overall batch summary added to the RITM work note>"
}
```

### Scope handling

| Record type | How scope is resolved |
|---|---|
| **New record** (`create_*`) | From `operation_details.scope` if provided; otherwise defaults to the instance default scope (usually `global`). |
| **Existing record** (`update_*`) | Read directly from the record's `sys_scope` field — no override. |

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
    "target_table": "u_project",
    "scope": "x_my_app_12345"
  },
  "context": {
    "target_record_name": "Alpha"
  },
  "payload": {
    "u_name": "Alpha",
    "u_status": "active",
    "u_owner": "abc123"
  },
  "worknote_summary": "Created a new record 'Alpha' in the custom table u_project (scope: x_my_app_12345)."
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

### Example — Batch (two records in one payload)

```json
{
  "operations": [
    {
      "operation_details": {
        "action": "create_script_include",
        "target_table": "sys_script_include"
      },
      "context": { "target_record_name": "ProjectUtils" },
      "payload": {
        "name": "ProjectUtils",
        "api_name": "global.ProjectUtils",
        "active": "true",
        "access": "public",
        "description": "Utility methods for project management.",
        "script": "var ProjectUtils = Class.create();\nProjectUtils.prototype = {\n    initialize: function() {},\n    type: 'ProjectUtils'\n};"
      },
      "worknote_summary": "Created Script Include 'ProjectUtils'."
    },
    {
      "operation_details": {
        "action": "create_business_rule",
        "target_table": "sys_script"
      },
      "context": { "target_record_name": "Set Project Active" },
      "payload": {
        "name": "Set Project Active",
        "table_name": "u_project",
        "when": "before",
        "action": "insert",
        "active": "true",
        "order": "100",
        "description": "Sets u_status to active on new project records.",
        "script": "(function executeRule(current, previous) {\n    current.u_status = 'active';\n})(current, previous);"
      },
      "worknote_summary": "Created Business Rule 'Set Project Active' on u_project (before insert)."
    }
  ],
  "worknote_summary": "Batch: Created Script Include 'ProjectUtils' and Business Rule 'Set Project Active' on u_project."
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
| `process` | `process(jsonInput, ritmGr)` | Main entry point. Parses JSON (single-op or batch), validates schema, dispatches to the correct handler(s), and adds an enriched work note listing all records affected. Returns `{ success, message }`. |
| `_processBatch` | `_processBatch(data, ritmGr)` | Internal. Iterates over the `operations` array, dispatches each, collects results, and writes one combined worknote. |
| `_dispatchAction` | `_dispatchAction(data)` | Internal. Routes a single validated operation to the correct handler. |
| `_buildWorknote` | `_buildWorknote(data, results)` | Internal. Builds the work note text: summary + "Records Affected" block with a direct link and scope for every record. |

All GlideRecord operations use `setValue` so that field-level ACLs and
business rules are respected in the same way as a manual update.

### Worknote format

Every RITM work note produced by SuperPromptProcessor follows this structure:

```
[AI Super Prompt] <worknote_summary from the JSON>

Records Affected (N)
============================================
  [1] Action : create_script_include
      Record : ProjectUtils
      Table  : sys_script_include
      Scope  : global
      Link   : https://dev12345.service-now.com/sys_script_include.do?sys_id=xxx
  --------------------------------------------
  [2] Action : create_business_rule
      Record : Set Project Active
      Table  : sys_script
      Scope  : global
      Link   : https://dev12345.service-now.com/sys_script.do?sys_id=yyy
============================================
```

---

## Extending with New Actions

1. Add the new `action` string to `super_prompt.txt` under "Action Types".
2. Add a new `case` in `SuperPromptProcessor._dispatchAction`.
3. Implement the handler method following the same pattern as the existing ones
   (return `{ success, message, sys_id, table, scope }`).
4. Re-run `python scripts/deploy_via_table_api.py` to push the updated Script
   Include to the instance, **or** export a new Update Set and replace
   `update_set/snow_super_prompt_update_set.xml`.

---

## License

MIT
