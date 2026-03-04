# Dynamic UI Patterns — Power Platform Custom Connectors

Source: https://learn.microsoft.com/en-us/connectors/custom-connectors/openapi-extensions

---

## Quick Reference

| Need | Use |
|------|-----|
| Dropdown from live API data | `x-ms-dynamic-values` + `x-ms-dynamic-list` |
| Body schema changes based on user input | `x-ms-dynamic-schema` + `x-ms-dynamic-properties` |
| Response schema varies by operation or input | `x-ms-dynamic-schema` + `x-ms-dynamic-properties` |

Always define both the original and the new variant together for forward compatibility. Use the new variant (`x-ms-dynamic-list`, `x-ms-dynamic-properties`) whenever any parameter reference points to a property within a body parameter — `parameterReference` paths in the new variant disambiguate body property references that the original variant cannot express.

---

## Dynamic Dropdowns

Populate a parameter's dropdown with live data from an API call made at design time.

### `x-ms-dynamic-values` (original)

**Fields:**

| Field | Description |
|-------|-------------|
| `operationId` | Required. The operationId that returns the list |
| `parameters` | Key-value pairs passed to the operation. Values can be static or reference other parameters |
| `value-collection` | Dot-separated path to the array within the response. Omit if the response root is the array |
| `value-path` | Path within each item to the value that gets stored when the user selects an option |
| `value-title` | Path within each item to the label shown to the user in the dropdown |

**Example:**

```json
{
  "name": "categoryId",
  "in": "query",
  "type": "string",
  "x-ms-summary": "Category",
  "x-ms-dynamic-values": {
    "operationId": "GetCategories",
    "parameters": {
      "status": "active"
    },
    "value-collection": "categories",
    "value-path": "id",
    "value-title": "displayName"
  }
}
```

---

### `x-ms-dynamic-list` (new — preferred when referencing body properties)

Use when any `parameterReference` points to a property inside a body parameter. The `parameterReference` path syntax disambiguates body property references.

**Fields:**

| Field | Description |
|-------|-------------|
| `operationId` | Required. The operationId that returns the list |
| `itemsPath` | Dot-separated path to the array within the response |
| `itemValuePath` | Path within each item to the stored value |
| `itemTitlePath` | Path within each item to the display label |
| `parameters` | Object mapping parameter names to static values or `parameterReference` objects |

**Parameter reference syntax:**

| Syntax | Use |
|--------|-----|
| `{ "value": "2.0" }` | Static value |
| `{ "parameterReference": "region" }` | Dynamic reference to a top-level parameter named `region` |
| `{ "parameterReference": "body/categoryId" }` | Dynamic reference to `categoryId` inside the body parameter |

**Example — `x-ms-dynamic-values` and `x-ms-dynamic-list` together:**

```json
{
  "name": "itemId",
  "in": "query",
  "type": "string",
  "x-ms-summary": "Item",
  "x-ms-dynamic-values": {
    "operationId": "GetItems",
    "parameters": {
      "categoryId": {
        "parameter": "categoryId"
      }
    },
    "value-collection": "items",
    "value-path": "id",
    "value-title": "name"
  },
  "x-ms-dynamic-list": {
    "operationId": "GetItems",
    "itemsPath": "items",
    "itemValuePath": "id",
    "itemTitlePath": "name",
    "parameters": {
      "categoryId": {
        "parameterReference": "body/categoryId"
      }
    }
  }
}
```

---

## Dynamic Schema

Fetch the schema for a parameter or response body at design time by calling an API operation. Use this when the shape of the body changes depending on what the user has selected in another field (e.g., a type or category picker).

### `x-ms-dynamic-schema` (original)

**Fields:**

| Field | Description |
|-------|-------------|
| `operationId` | The operationId that returns the schema |
| `parameters` | Values passed to the schema operation |
| `value-path` | Dot-separated path in the response where the JSON Schema object lives |

**`x-ms-dynamic-properties` (new — preferred when referencing body properties)**

Use when any `parameterReference` points to a property inside a body parameter.

**Fields:**

| Field | Description |
|-------|-------------|
| `operationId` | The operationId that returns the schema |
| `parameters` | Object mapping parameter names to static values or `parameterReference` objects |
| `itemValuePath` | Dot-separated path in the response where the JSON Schema object lives |

**Example — both variants together on a response schema:**

```json
"responses": {
  "200": {
    "description": "Success",
    "schema": {
      "type": "object",
      "x-ms-dynamic-schema": {
        "operationId": "GetItemSchema",
        "parameters": {
          "category": {
            "parameter": "category"
          }
        },
        "value-path": "schema"
      },
      "x-ms-dynamic-properties": {
        "operationId": "GetItemSchema",
        "parameters": {
          "category": {
            "parameterReference": "body/category"
          }
        },
        "itemValuePath": "schema"
      }
    }
  }
}
```

The same pattern applies to a body parameter's `schema` property when the request body shape varies by input.

---

## The Schema Operation Pattern

The operation referenced by `x-ms-dynamic-schema` or `x-ms-dynamic-properties` must follow these rules:

1. Accept the same parameters the calling operation uses to determine the schema.
2. Return a valid JSON Schema object at the path specified by `value-path` or `itemValuePath`.
3. Be marked `x-ms-visibility: "important"` on the operation itself.
4. Mark any helper parameters that are not shown to users as `x-ms-visibility: "internal"`.

**Example schema operation definition:**

```json
"/schema/item": {
  "get": {
    "operationId": "GetItemSchema",
    "summary": "Get item schema",
    "x-ms-visibility": "important",
    "parameters": [
      {
        "name": "category",
        "in": "query",
        "type": "string",
        "required": true,
        "x-ms-summary": "Category",
        "x-ms-visibility": "internal"
      }
    ],
    "responses": {
      "200": {
        "description": "JSON Schema for the item type",
        "schema": {
          "type": "object"
        }
      }
    }
  }
}
```

**Example schema response body:**

```json
{
  "schema": {
    "type": "object",
    "properties": {
      "itemId": {
        "type": "string",
        "x-ms-summary": "Item ID"
      },
      "status": {
        "type": "string",
        "x-ms-summary": "Status",
        "enum": ["active", "inactive"]
      },
      "quantity": {
        "type": "integer",
        "x-ms-summary": "Quantity"
      }
    },
    "required": ["itemId", "status"]
  }
}
```

The value at `"schema"` is the JSON Schema object. The `value-path` or `itemValuePath` points to `"schema"` in this case.

---

## Enums and Multi-Select Arrays

A static `enum` can be defined directly on a `string` parameter to give the maker a fixed dropdown without any API call:

```json
{
  "name": "status",
  "in": "query",
  "type": "string",
  "x-ms-summary": "Status",
  "enum": ["active", "inactive", "pending"]
}
```

If the API accepts **multiple selections** from a fixed list, the parameter becomes an `array` with the `enum` on the `items` string — not on the array itself:

```json
{
  "name": "statuses",
  "in": "query",
  "type": "array",
  "items": {
    "type": "string",
    "enum": ["active", "inactive", "pending"]
  },
  "collectionFormat": "csv",
  "x-ms-summary": "Statuses"
}
```

**Critical caveat:** When the parameter is `type: array`, Power Automate sends the value as an array (e.g., `["active", "pending"]`). Many APIs do not accept an array for what they treat as a single-value field — they expect a plain string. Verify the API actually supports multi-value for that parameter before using this pattern. If the API only accepts one value, keep the parameter as `type: string` with `enum` directly on it. Using the array form with a single-value API will result in a serialisation error or a `400 Bad Request` at runtime.

---

## When No Schema Endpoint Exists

If the API has no endpoint that returns an OpenAPI-compatible schema, `x-ms-dynamic-schema` and `x-ms-dynamic-properties` can still be used — but only by creating an internal connector operation backed by `script.csx` that calls a proprietary metadata endpoint, transforms the response into a JSON Schema object, and returns it.

This is viable but adds meaningful complexity:
- A new internal operation must be defined in the swagger (`x-ms-visibility: "internal"`) solely to serve the schema.
- The script must correctly map the API's field types to JSON Schema types and construct the full schema object with `properties`, `required`, and `x-ms-summary` on each field.
- The operation must be listed in `scriptOperations` in `apiProperties.json`.
- Errors in the schema construction surface as broken or empty field lists in the designer — debugging is non-trivial.

Use this approach when the dynamic field set is genuinely needed and there is no alternative. See `references/script-csx.md` for the full pattern with a working implementation.

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using only `x-ms-dynamic-values` when referencing body properties | Add `x-ms-dynamic-list` with `parameterReference` paths that use the `body/propertyName` syntax |
| Using only `x-ms-dynamic-schema` when referencing body properties | Add `x-ms-dynamic-properties` with `parameterReference` paths that use the `body/propertyName` syntax |
| Schema operation not marking helper params internal | Add `x-ms-visibility: "internal"` to all params not directly entered by the user |
| Referenced schema operation returns wrong format | Ensure `value-path` or `itemValuePath` resolves to a valid JSON Schema object, not a scalar or array |
| `x-ms-dynamic-values` with no `value-collection` when response has a wrapper | Specify `value-collection` pointing to the array property within the response (e.g., `"items"` or `"data.results"`) |
