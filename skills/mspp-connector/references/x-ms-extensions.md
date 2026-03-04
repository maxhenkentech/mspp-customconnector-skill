# x-ms Extensions — Power Platform Custom Connectors

Source: https://learn.microsoft.com/en-us/connectors/custom-connectors/openapi-extensions

---

## Metadata and Labelling

### `summary` (native OpenAPI)

**Applies to:** Operations

**Purpose:** Operation title shown in the designer and maker portal. Displayed as the primary label for the action or trigger.

**Rules:**
- Sentence case (capitalize first word only)
- Maximum 120 characters
- No trailing period
- Keep concise — this is a title, not a sentence

**Example:**

```json
"paths": {
  "/items/{itemId}": {
    "get": {
      "summary": "Get item by ID",
      "operationId": "GetItemById"
    }
  }
}
```

---

### `x-ms-summary`

**Applies to:** Parameters, response schema properties

**Purpose:** Display label for a parameter or schema property in the designer. Replaces the raw field name with a human-readable label.

**Rules:**
- Title case (capitalize all major words)
- Keep short — appears inline next to the field
- Do not repeat the parameter name verbatim

**Example:**

```json
"parameters": [
  {
    "name": "itemId",
    "in": "path",
    "required": true,
    "type": "string",
    "x-ms-summary": "Item ID",
    "description": "The unique identifier of the item to retrieve."
  }
]
```

---

### `description` (native OpenAPI)

**Applies to:** Operations, parameters, schema properties

**Purpose:** Detailed explanation shown as a tooltip or secondary label. Provides context the summary cannot convey.

**Rules:**
- Sentence case
- End with a period
- Can be multi-sentence

**Example:**

```json
{
  "name": "status",
  "in": "query",
  "type": "string",
  "x-ms-summary": "Status",
  "description": "Filter results by status. Accepted values are active, inactive, and pending."
}
```

---

## Visibility

### `x-ms-visibility`

**Applies to:** Operations, parameters, schema properties

**Purpose:** Controls whether and where an item appears in the designer.

**Valid values:**

| Value | Behavior |
|-------|----------|
| `"important"` | Always shown. Displayed first in the parameter list |
| `"advanced"` | Hidden under "Show advanced options". Shown on demand |
| `"internal"` | Never shown to the user. Must have a `default` value if also `required` |

**Rules:**
- Parameters that are `required: true` and `x-ms-visibility: "internal"` must also declare a `default` value so the platform can supply it automatically.
- Use `"internal"` for parameters that are populated programmatically (e.g., by dynamic values or policy templates), not by the user.

**Example:**

```json
"parameters": [
  {
    "name": "category",
    "in": "query",
    "type": "string",
    "x-ms-summary": "Category",
    "x-ms-visibility": "important"
  },
  {
    "name": "pageSize",
    "in": "query",
    "type": "integer",
    "x-ms-summary": "Page Size",
    "x-ms-visibility": "advanced",
    "default": 20
  },
  {
    "name": "schemaType",
    "in": "query",
    "type": "string",
    "x-ms-visibility": "internal",
    "required": true,
    "default": "full"
  }
]
```

---

## Connector-Level Metadata

### `x-ms-connector-metadata`

**Applies to:** Top-level swagger object

**Purpose:** Required for connector certification. Declares website, privacy policy, and categories for the connector.

**Structure:** Array of objects, each with `propertyName` and `propertyValue`.

**Valid `propertyName` values:** `Website`, `Privacy policy`, `Categories`

**Valid category values:** AI, Business Intelligence, Business Management, Commerce, Communication, Content and Files, Data, Finance, Human Resources, IT Operations, Marketing, Productivity, Sales and CRM, Security, Website

**Example:**

```json
"x-ms-connector-metadata": [
  {
    "propertyName": "Website",
    "propertyValue": "https://www.myservice.example"
  },
  {
    "propertyName": "Privacy policy",
    "propertyValue": "https://www.myservice.example/privacy"
  },
  {
    "propertyName": "Categories",
    "propertyValue": "Data;Productivity"
  }
]
```

---

### `x-ms-capabilities` (connector level)

**Applies to:** Top-level swagger object

**Purpose:** Declares connector-level capabilities. Use `testConnection` to specify the operationId of the operation that verifies the connection is valid. Required when using the `setconnectionstatustounauthenticated` policy template.

**Example:**

```json
"x-ms-capabilities": {
  "testConnection": {
    "operationId": "TestConnection",
    "parameters": {}
  }
}
```

---

### `x-ms-capabilities` (operation level)

**Applies to:** Individual operations

**Purpose:** Declares operation-level capabilities. Set `chunkTransfer: true` to enable chunked file upload support for an operation.

**Example:**

```json
"/files/upload": {
  "post": {
    "operationId": "UploadFile",
    "x-ms-capabilities": {
      "chunkTransfer": true
    }
  }
}
```

---

## Operation Versioning

### `x-ms-api-annotation`

**Applies to:** Operations

**Purpose:** Groups operations into versioned families and declares deprecation relationships. Use alongside `deprecated: true` to point users to a replacement operation.

**Fields:**
- `family` — string identifier shared across all versions of the same logical operation
- `revision` — integer version number within the family (1, 2, 3, ...)
- `replacement` — object with `operationId` pointing to the newer version. Only set on the deprecated operation.

**Example:**

```json
"/items": {
  "get": {
    "operationId": "GetItems",
    "deprecated": true,
    "x-ms-api-annotation": {
      "family": "GetItems",
      "revision": 1,
      "replacement": {
        "operationId": "GetItemsV2"
      }
    }
  }
},
"/v2/items": {
  "get": {
    "operationId": "GetItemsV2",
    "x-ms-api-annotation": {
      "family": "GetItems",
      "revision": 2
    }
  }
}
```

---

### `x-ms-operation-context`

**Applies to:** Operations

**Purpose:** Simulates a trigger firing during connector testing. Specifies which trigger operationId and parameters to simulate, enabling end-to-end testing without needing a real event.

**Example:**

```json
"x-ms-operation-context": {
  "simulate": {
    "operationId": "OnNewItem",
    "parameters": {
      "category": "general"
    }
  }
}
```

---

## Triggers and Webhooks

### `x-ms-trigger`

**Applies to:** Operations

**Purpose:** Marks an operation as a trigger. Absent means the operation is an action.

**Valid values:**

| Value | Behavior |
|-------|----------|
| `"single"` | Response is a single object. Platform polls and fires when the object changes |
| `"batch"` | Response is an array. Platform polls and fires once per new item in the array |

**Example:**

```json
"operationId": "OnNewItem",
"x-ms-trigger": "batch",
"responses": {
  "200": {
    "schema": {
      "type": "array",
      "items": {
        "$ref": "#/definitions/Item"
      }
    }
  }
}
```

---

### `x-ms-trigger-hint`

**Applies to:** Operations (triggers)

**Purpose:** Human-readable hint displayed to users explaining how to fire the trigger event. Shown in the designer beneath the trigger name.

**Example:**

```json
"x-ms-trigger-hint": "Fire this trigger by creating a new item in the system."
```

---

### `x-ms-notification-content`

**Applies to:** Path level (sibling of the trigger registration operation)

**Purpose:** Defines the schema of the webhook payload the backend will POST to the callback URL. Declared at path level alongside the subscription operation, not inside the operation itself.

**Example:**

```json
"/webhooks/items": {
  "x-ms-notification-content": {
    "description": "Item event payload",
    "schema": {
      "$ref": "#/definitions/ItemEventPayload"
    }
  },
  "post": {
    "operationId": "RegisterItemWebhook",
    "x-ms-trigger": "single"
  }
}
```

---

### `x-ms-notification-url`

**Applies to:** Parameters

**Purpose:** Marks the parameter that receives the webhook callback URL. The platform injects the callback URL into this parameter automatically.

**Rules:**
- Set to `true` on the parameter
- Parameter must be `required: true`
- Parameter must be `x-ms-visibility: "internal"`
- The parameter receives the platform-generated callback URL — never exposed to the user

**Example:**

```json
{
  "name": "callbackUrl",
  "in": "body",
  "required": true,
  "x-ms-notification-url": true,
  "x-ms-visibility": "internal",
  "schema": {
    "type": "string"
  }
}
```

---

## URL Encoding

### `x-ms-url-encoding`

**Applies to:** Path parameters

**Purpose:** Controls how path parameter values are URL-encoded before being substituted into the path.

**Valid values:**

| Value | Behavior |
|-------|----------|
| `"single"` | Default. Standard single URL encoding |
| `"double"` | Double-encodes the value. Use when the parameter itself contains URL-encoded characters (e.g., a path segment that is already encoded) |

**Example:**

```json
{
  "name": "itemPath",
  "in": "path",
  "required": true,
  "type": "string",
  "x-ms-url-encoding": "double",
  "x-ms-summary": "Item Path"
}
```

---

## Enumerations

### `x-ms-enum`

**Applies to:** Parameters and schema properties with an `enum` array

**Purpose:** Provides human-readable display names for each enum value. Shown in dropdowns in the designer.

**Structure:**
- `name` — internal name for the enum type
- `modelAsString` — set `true` to treat values as open strings (not strictly validated)
- `values` — array of objects, each with `value` (must match an entry in `enum`) and `name` (display label)

**Rule:** Every value listed in `enum` must appear in the `values` array.

**Example:**

```json
{
  "name": "status",
  "in": "query",
  "type": "string",
  "x-ms-summary": "Status",
  "enum": ["active", "inactive", "pending"],
  "x-ms-enum": {
    "name": "ItemStatus",
    "modelAsString": false,
    "values": [
      { "value": "active", "name": "Active" },
      { "value": "inactive", "name": "Inactive" },
      { "value": "pending", "name": "Pending" }
    ]
  }
}
```

---

## Certification Extensions

### `x-ms-require-user-consent`

**Applies to:** Top-level swagger object

**Purpose:** Controls whether users see a consent prompt before establishing a connection. Set to `true` to force a consent dialog, `false` to suppress it.

**Example:**

```json
"x-ms-require-user-consent": true
```

---

### `x-ms-no-generic-test`

**Applies to:** Top-level swagger object

**Purpose:** Suppresses the generic test run that the connector validation tooling performs automatically. Set to `true` when the connector does not support a generic test call (e.g., no safe read-only endpoint available for testing).

**Example:**

```json
"x-ms-no-generic-test": true
```
