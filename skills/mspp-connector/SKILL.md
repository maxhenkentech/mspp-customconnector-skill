---
name: mspp-connector
description: This skill should be used when the user asks to "build a custom connector", "create a Power Platform connector", "add a custom connector", "update a custom connector", "deploy a connector with paconn", "validate a swagger for Power Platform", "add policy templates to a connector", "use x-ms-dynamic-values", "use x-ms-dynamic-schema", "add dynamic properties to a connector", "write script.csx", "connect to a SOAP API from Power Automate", "add a webhook trigger to a connector", "use x-ms-trigger", "Power Apps custom connector", or "add connection parameters", or when they mention paconn, apiDefinition.swagger.json, apiProperties.json, or Power Automate custom connectors.
---

# Microsoft Power Platform Custom Connector Builder

This skill guides the complete authoring, validation, and deployment lifecycle for Microsoft Power Platform custom connectors. It covers the connector file structure, OpenAPI authoring rules, x-ms extensions, policy templates, dynamic UI patterns, webhook triggers, and the paconn CLI deploy workflow. Detailed reference material lives in the `references/` files listed at the end of this document — load them on demand.

---

## Connector File Structure

Every connector is a folder containing up to five files. Only the first two are required for deployment.

| File | Purpose | Required |
|------|---------|----------|
| `apiDefinition.swagger.json` | OpenAPI 2.0 definition — operations, parameters, schemas, x-ms extensions | Yes |
| `apiProperties.json` | Authentication, branding, policy templates, script operation names | Yes |
| `icon.png` | Connector icon displayed in the designer — must be < 1 MB | No |
| `script.csx` | C# custom code executed per-request — use as a last resort only | No |
| `settings.json` | Stores paconn CLI arguments (environment, connector ID) — never deployed | No |

---

## Core Authoring Workflow

Follow these steps in order when creating or updating a connector.

1. **Start from the API specification.** Import an existing OpenAPI 2.0 or Postman collection if available, or draft `apiDefinition.swagger.json` by hand. Confirm that the top-level `swagger` field is `"2.0"`, that `info` contains a non-empty `title` and `version`, that `host` matches the live API hostname, and that `basePath` and `schemes` are correct. Power Platform does not support OpenAPI 3.x — convert down to 2.0 if necessary.

2. **Ensure all operations have quality metadata.** Apply every rule in the Quality Checklist below to every operation, parameter, and response property before adding any extensions. The Power Automate designer surfaces `summary`, `description`, `x-ms-summary`, and `x-ms-visibility` directly to makers. Poor metadata degrades the maker experience and can cause the connector review process to reject the submission.

3. **Add x-ms extensions for visibility, summaries, dynamic UI, and triggers.** At minimum set `x-ms-visibility` on every operation and `x-ms-summary` on every parameter and response property. Add dynamic value or schema extensions wherever a parameter's valid values depend on other inputs — for example, a "folder" parameter that should only show folders available in the account selected in a prior parameter. Consult `references/x-ms-extensions.md` for the full extension catalog and `references/dynamic-patterns.md` for dynamic UI patterns.

4. **Add policy templates where possible instead of custom code.** Route-rewriting, header injection, query-string manipulation, CORS handling, and many payload transformations are available as declarative policy templates configured in `apiProperties.json` under `policyTemplateInstances`. Reach for `script.csx` only after confirming no policy template covers the requirement. See `references/policy-templates.md` for all ten templateIds with their parameters and complete JSON examples.

5. **Configure authentication and branding in `apiProperties.json`.** Set the `connectionParameters` block for the chosen auth type (API Key, OAuth 2.0, Basic, Windows, etc.). Fill in `iconBrandColor` with a hex color that matches the service's brand. Reference `icon.png` in `iconUrl` if one exists. List any operations that invoke custom code in the `scriptOperations` array — operations omitted from this array will not execute `script.csx` even if the file is present.

6. **Validate.** Run `paconn validate` against the connector folder before every deployment. Fix every reported error and warning before proceeding. Validation checks operationId uniqueness, required field presence, x-ms extension correctness, and schema integrity. A clean validation pass is mandatory.

7. **Deploy.** Use `paconn create` for a brand-new connector or `paconn update` for an existing one. Pass `--api-def`, `--api-prop`, and `--icon` explicitly or rely on a `settings.json` file in the connector folder. For detailed CLI flags and the guided interactive workflow see `references/paconn-guide.md`.

---

## Quality Checklist

### Operations

| Field | Rule |
|-------|------|
| `operationId` | Unique across the connector; PascalCase; alphanumeric characters only — no spaces, hyphens, or underscores. Example: `GetItemById` |
| `summary` | Sentence case; 120 characters or fewer; no trailing period. Example: `Get item by identifier` |
| `description` | Full sentence with a trailing period. Example: `Returns a single item matching the provided identifier.` |
| `x-ms-visibility` | `"important"` for primary, frequently used operations; `"advanced"` for secondary or rarely used operations; `"internal"` for platform-managed operations (e.g., webhook deregistration) that should not appear in the designer |

### Parameters and Response Properties

| Field | Rule |
|-------|------|
| `x-ms-summary` | Title case label shown in the Power Automate designer. Example: `Item Identifier` |
| `description` | Full sentence ending with a period. Example: `The unique identifier of the item to retrieve.` |
| `x-ms-visibility` | `"important"`, `"advanced"`, or `"internal"`. Internal parameters must also specify a `default` value and must be `required` |

### Schema Definitions

- Every `$ref` must resolve to a named definition under `#/definitions/`. Inline anonymous schemas nested in operation responses are permitted but make reuse harder — prefer `$ref` for any schema used by more than one operation.
- Every array-type property must include an `items` schema. An array without `items` will fail validation and produce an unusable output in the designer.
- List required fields explicitly in a `required` array at the object schema level — do not rely on implicit required behavior. The `required` array must contain only field names that actually exist as keys in `properties`.
- Avoid `additionalProperties: true` on request body schemas. Power Platform ignores extra fields silently, but an open schema makes the designer unable to show individual properties to the maker.
- Use `format` consistently: `"date-time"` for ISO 8601 timestamps, `"int32"` / `"int64"` for integers, `"binary"` for file uploads. Mismatched formats cause silent coercion errors at runtime.

---

## Decision Tree: Policies vs Dynamic Properties vs script.csx

Use this tree to choose the right extension mechanism before writing any code.

```
Is the need about transforming payloads, setting/removing headers,
routing to different endpoints, or URL rewriting?
│
├── YES → Can a policy template cover it?  (see references/policy-templates.md)
│         │
│         ├── YES → Use a policy template in apiProperties.json. Done.
│         │
│         └── NO  → Is it a SOAP/XML API, or does it require complex
│                   multi-step orchestration unavailable in policies?
│                   │
│                   ├── YES → Use script.csx (last resort).
│                   │         (see references/script-csx.md)
│                   │
│                   └── NO  → Reconsider the requirement. A policy
│                             template or a change to the API itself
│                             is almost always the better path.
│
└── NO  → Is the need a dynamic dropdown, dynamic list of options,
           or a schema that changes based on earlier user input?
          │
          ├── Dynamic dropdown / list of valid values
          │   └── Use x-ms-dynamic-values or x-ms-dynamic-list
          │       (see references/dynamic-patterns.md)
          │
          └── Dynamic schema / dynamic properties on a parameter
              └── Use x-ms-dynamic-schema or x-ms-dynamic-properties
                  (see references/dynamic-patterns.md)
```

**Note:** Authentication and connection parameter configuration are not shown in this tree. Configure auth types (OAuth 2.0, API key, Basic) and connection parameters in `apiProperties.json`. For multi-instance connectors (different tenant URLs), use `connectionParameterSets`. See `references/swagger-support.md` for the `connectionParameterSets` pattern.

---

## Trigger and Webhook Patterns

Power Platform supports push triggers backed by a webhook registration model. Implement this pattern in `apiDefinition.swagger.json`.

**Required elements:**

- Mark the webhook registration operation (typically a POST) with `"x-ms-trigger": "single"` for single-event triggers or `"x-ms-trigger": "batch"` for batch-event triggers. Use `"single"` when the API sends one event per HTTP POST to the callback URL. Use `"batch"` when the API sends an array of events in a single POST.
- On the callback URL parameter, set `"x-ms-notification-url": true`. This parameter must also carry `"x-ms-visibility": "internal"`, `"required": true`, and must not appear in the user-visible portion of the flow designer. Power Automate injects the callback URL value automatically at runtime.
- Add an `x-ms-notification-content` block at the **path level** — as a sibling of the `post` and `delete` keys, not nested inside either operation. This block declares the schema of the event payload that the remote service will POST to the callback URL. Power Automate uses it to make individual event fields available as dynamic content tokens in the flow.
- Add a paired DELETE operation on the same path for webhook deregistration. Power Automate calls this DELETE automatically when a flow is deleted, turned off, or when the connection is removed. The DELETE operation must be marked `"x-ms-visibility": "internal"` so it does not appear in the action list.
- Return the webhook registration identifier from the POST response. Power Automate captures this value and passes it as a path parameter to the DELETE operation for cleanup.

**Compact example:**

```json
"/webhooks": {
  "x-ms-notification-content": {
    "description": "Event payload delivered to the callback URL.",
    "schema": {
      "$ref": "#/definitions/EventPayload"
    }
  },
  "post": {
    "operationId": "RegisterWebhook",
    "summary": "Register a webhook",
    "description": "Registers a callback URL to receive event notifications.",
    "x-ms-trigger": "single",
    "x-ms-visibility": "important",
    "parameters": [
      {
        "name": "body",
        "in": "body",
        "required": true,
        "schema": {
          "type": "object",
          "required": ["callbackUrl"],
          "properties": {
            "callbackUrl": {
              "type": "string",
              "x-ms-notification-url": true,
              "x-ms-visibility": "internal",
              "x-ms-summary": "Callback URL"
            },
            "event": {
              "type": "string",
              "x-ms-summary": "Event Type",
              "description": "The event type that triggers notification."
            }
          }
        }
      }
    ],
    "responses": {
      "201": {
        "description": "Webhook registered successfully.",
        "schema": {
          "type": "object",
          "properties": {
            "id": {
              "type": "string",
              "x-ms-summary": "Webhook Identifier",
              "description": "The identifier of the registered webhook."
            }
          }
        }
      }
    }
  }
},
"/webhooks/{id}": {
  "delete": {
    "operationId": "UnregisterWebhook",
    "summary": "Unregister a webhook",
    "x-ms-visibility": "internal",
    "parameters": [
      {
        "name": "id",
        "in": "path",
        "required": true,
        "type": "string",
        "x-ms-summary": "Webhook Identifier",
        "x-ms-visibility": "internal",
        "description": "The identifier of the webhook registration to delete."
      }
    ],
    "responses": {
      "200": { "description": "Webhook unregistered successfully." },
      "default": { "description": "Operation failed." }
    }
  }
}
```

---

## Guided paconn Deploy Workflow

**Rule: Always run `paconn validate` before `paconn create` or `paconn update`.**

Validation catches schema errors, missing required fields, invalid x-ms extension usage, and broken `$ref` pointers before they cause a failed or broken deployment. Treat every validation warning as an error — warnings in custom connectors commonly surface as silent runtime failures or broken designer experiences.

**Standard three-step sequence:**

```bash
# Step 1 — Validate (mandatory before every deployment)
paconn validate --api-def apiDefinition.swagger.json

# Step 2a — Create a new connector (first deployment)
paconn create --api-def apiDefinition.swagger.json \
              --api-prop apiProperties.json \
              --icon icon.png

# Step 2b — Update an existing connector
paconn update --api-def apiDefinition.swagger.json \
              --api-prop apiProperties.json \
              --icon icon.png
```

Store the environment ID, connector ID, and environment URL in `settings.json` so subsequent `paconn update` calls resolve arguments automatically without manual flags. The `settings.json` file is never uploaded to Power Platform — it is a local developer convenience file only. For the full CLI reference, interactive guided mode, authentication setup, and the complete `settings.json` schema see `references/paconn-guide.md`.

---

## Reference Files

Load these files on demand when the task requires their specific content. Do not load all of them upfront.

| File | When to load |
|------|-------------|
| `references/paconn-guide.md` | CLI commands, deployment flags, settings.json schema, guided paconn experience |
| `references/policy-templates.md` | All ten policy template templateIds, parameters, and JSON examples |
| `references/x-ms-extensions.md` | Full catalog of supported x-ms-* extensions with syntax and examples |
| `references/dynamic-patterns.md` | x-ms-dynamic-values, x-ms-dynamic-list, x-ms-dynamic-schema, x-ms-dynamic-properties |
| `references/swagger-support.md` | Supported vs unsupported Swagger 2.0 properties for Power Platform |
| `references/script-csx.md` | When and how to use script.csx, ScriptBase API, SOAP and XML patterns |
