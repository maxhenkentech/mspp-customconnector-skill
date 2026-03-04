# Policy Templates — Power Platform Custom Connectors

Source: https://learn.microsoft.com/en-us/connectors/custom-connectors/policy-templates

---

## Structure

Define policy template instances in `apiProperties.json` under the `policyTemplateInstances` array.

```json
"policyTemplateInstances": [
  {
    "templateId": "<templateId>",
    "title": "Human-readable title",
    "parameters": {
      "x-ms-apimTemplateParameter.<param>": "<value>",
      "x-ms-apimTemplate-policySection": "Request",
      "x-ms-apimTemplate-operationName": ["OperationId1"]
    }
  }
]
```

Omit `x-ms-apimTemplate-operationName` to apply the policy to all operations.

---

## Expression Syntax

Use these expressions as parameter values to reference runtime data.

| Expression | Returns |
|-----------|---------|
| `@headers('name')` | Request header value |
| `@queryParameters('name')` | Query parameter value |
| `@connectionParameters('name')` | Connection parameter value |
| `@body()` | Full body |
| `@item()` | Current item in a collection transform |
| `@environmentVariables('name')` | Power Platform environment variable (preview) |

---

## Templates

### 1. setheader — Set HTTP Header

**Status:** GA

Sets or overrides a request or response header. Use on Request, Response, or Failure sections.

**Use cases:**
- Inject an `Accept` or `Content-Type` header on all outgoing requests
- Add an API key header from a connection parameter
- Remove or normalize a response header

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `x-ms-apimTemplateParameter.name` | Header name |
| `x-ms-apimTemplateParameter.value` | Header value. Supports expressions |
| `x-ms-apimTemplateParameter.existsAction` | Action when header exists: `override`, `skip`, or `append` |
| `x-ms-apimTemplate-policySection` | `Request`, `Response`, or `Failure` |
| `x-ms-apimTemplate-operationName` | Array of operationIds. Omit for all operations |

**Example — Set `Accept: application/json` on all requests:**

```json
{
  "templateId": "setheader",
  "title": "Set Accept header",
  "parameters": {
    "x-ms-apimTemplateParameter.name": "Accept",
    "x-ms-apimTemplateParameter.value": "application/json",
    "x-ms-apimTemplateParameter.existsAction": "override",
    "x-ms-apimTemplate-policySection": "Request"
  }
}
```

---

### 2. setqueryparameter — Set Query String Parameter

**Status:** GA

Adds or updates a query string parameter on the outgoing request.

**Use cases:**
- Append a fixed API version to every call
- Add a tenant ID from a connection parameter
- Force a response format query flag

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `x-ms-apimTemplateParameter.name` | Query parameter name |
| `x-ms-apimTemplateParameter.value` | Query parameter value. Supports expressions |
| `x-ms-apimTemplateParameter.existsAction` | `override`, `skip`, or `append` |
| `x-ms-apimTemplate-operationName` | Array of operationIds. Omit for all operations |

**Example — Add `api-version=2024-01-01` to all requests:**

```json
{
  "templateId": "setqueryparameter",
  "title": "Set api-version query parameter",
  "parameters": {
    "x-ms-apimTemplateParameter.name": "api-version",
    "x-ms-apimTemplateParameter.value": "2024-01-01",
    "x-ms-apimTemplateParameter.existsAction": "override"
  }
}
```

---

### 3. dynamichosturl — Set Host URL

**Status:** GA

Replaces the host URL dynamically at runtime. Bypasses the `host` value in the swagger definition for targeted operations. Use this when each connection points to a different base URL (multi-tenant or multi-instance scenarios).

**Use cases:**
- Build the host from a connection parameter such as `instanceUrl`
- Route specific operations to a different subdomain
- Support per-user tenant URLs without separate connectors

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `x-ms-apimTemplateParameter.urlTemplate` | Full host URL. Supports expressions |
| `x-ms-apimTemplate-operationName` | Array of operationIds. Omit for all operations |

**Example — Build host from `instanceUrl` connection parameter:**

```json
{
  "templateId": "dynamichosturl",
  "title": "Set dynamic host from connection parameter",
  "parameters": {
    "x-ms-apimTemplateParameter.urlTemplate": "@connectionParameters('instanceUrl')"
  }
}
```

---

### 4. routerequesttoendpoint — Route Request

**Status:** GA

Routes the request to a different relative path on the same backend. Useful when multiple connector operations share the same backend endpoint but need different swagger definitions (different input/output schemas). The host remains the same; only the path and HTTP method change.

**Use cases:**
- Map a versioned operation (`GetItemsV2`) to `/v2/items` while keeping `GetItems` on `/items`
- Normalize different operation names to a single backend path
- Redirect a legacy operation path to a new backend route

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `x-ms-apimTemplateParameter.newPath` | New relative path. Supports expressions |
| `x-ms-apimTemplateParameter.httpMethod` | HTTP method for the rerouted request |
| `x-ms-apimTemplate-operationName` | Array of operationIds this routing applies to |

**Example — Route `GetItemsV2` to `/v2/items`:**

```json
{
  "templateId": "routerequesttoendpoint",
  "title": "Route GetItemsV2 to v2 endpoint",
  "parameters": {
    "x-ms-apimTemplateParameter.newPath": "/v2/items",
    "x-ms-apimTemplateParameter.httpMethod": "GET",
    "x-ms-apimTemplate-operationName": ["GetItemsV2"]
  }
}
```

---

### 5. setproperty — Set Property

**Status:** Preview

Adds or updates a property in the request or response body. The value can be a constant, a path expression pointing to another property in the body, or a combination.

**Use cases:**
- Inject a computed field into a response object
- Concatenate two body properties into a new combined property
- Normalize a missing field by setting a default value

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `x-ms-apimTemplateParameter.newPropertyParentPathTemplate` | JSON path to the parent object where the new property is added |
| `x-ms-apimTemplateParameter.newPropertySubPathTemplate` | Property name to set within the parent |
| `x-ms-apimTemplateParameter.propertyValuePathTemplate` | Value or expression for the new property |
| `x-ms-apimTemplate-policySection` | `Request` or `Response` |
| `x-ms-apimTemplate-operationName` | Array of operationIds. Omit for all operations |

**Example — Concatenate `firstName` and `lastName` into `fullName` on response:**

```json
{
  "templateId": "setproperty",
  "title": "Build fullName from firstName and lastName",
  "parameters": {
    "x-ms-apimTemplateParameter.newPropertyParentPathTemplate": "@body()",
    "x-ms-apimTemplateParameter.newPropertySubPathTemplate": "fullName",
    "x-ms-apimTemplateParameter.propertyValuePathTemplate": "@concat(body().firstName, ' ', body().lastName)",
    "x-ms-apimTemplate-policySection": "Response"
  }
}
```

---

### 6. convertarraytoobject — Convert Array To Object

**Status:** Preview

Converts an array in the body into a JSON object keyed by a specified property of each item. The original array can optionally be removed.

**Use cases:**
- Convert an `items` array into an `itemsById` map for O(1) lookup by consumers
- Reindex a list by a unique identifier before returning to Power Apps
- Normalize an API response that returns arrays when the caller expects keyed objects

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `x-ms-apimTemplateParameter.propertyParentPath` | JSON path to the parent object containing the array |
| `x-ms-apimTemplateParameter.propertySubPath` | Name of the array property within the parent |
| `x-ms-apimTemplateParameter.keyWithinCollectionPath` | Property within each item to use as the key |
| `x-ms-apimTemplateParameter.newPropertyPath` | Path for the resulting object |
| `x-ms-apimTemplateParameter.retainKey` | `true` to keep the key property in each item, `false` to remove it |
| `x-ms-apimTemplate-policySection` | `Request` or `Response` |
| `x-ms-apimTemplate-operationName` | Array of operationIds. Omit for all operations |

**Example — Convert `items` array to `itemsById` keyed by `id`:**

```json
{
  "templateId": "convertarraytoobject",
  "title": "Convert items array to itemsById map",
  "parameters": {
    "x-ms-apimTemplateParameter.propertyParentPath": "@body()",
    "x-ms-apimTemplateParameter.propertySubPath": "items",
    "x-ms-apimTemplateParameter.keyWithinCollectionPath": "id",
    "x-ms-apimTemplateParameter.newPropertyPath": "itemsById",
    "x-ms-apimTemplateParameter.retainKey": "true",
    "x-ms-apimTemplate-policySection": "Response"
  }
}
```

---

### 7. convertobjecttoarray — Convert Object To Array

**Status:** Preview

Inverse of `convertarraytoobject`. Converts a keyed JSON object into an array of objects, each with a configurable `name` and `value` property.

**Use cases:**
- Normalize a map response into an array that Power Apps can iterate over
- Convert a keyed metadata object into a list for display in a gallery
- Flatten a dictionary response into a schema-friendly array

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `x-ms-apimTemplateParameter.propertyParentPath` | JSON path to the parent object containing the keyed object |
| `x-ms-apimTemplateParameter.propertySubPath` | Name of the keyed object property within the parent |
| `x-ms-apimTemplateParameter.newPropertyPath` | Path for the resulting array |
| `x-ms-apimTemplateParameter.keyName` | Property name used for the key in each array item |
| `x-ms-apimTemplateParameter.valueName` | Property name used for the value in each array item |
| `x-ms-apimTemplate-policySection` | `Request` or `Response` |
| `x-ms-apimTemplate-operationName` | Array of operationIds. Omit for all operations |

**Example — Convert a keyed object into an array with `name` and `value` properties:**

```json
{
  "templateId": "convertobjecttoarray",
  "title": "Convert metadata object to array",
  "parameters": {
    "x-ms-apimTemplateParameter.propertyParentPath": "@body()",
    "x-ms-apimTemplateParameter.propertySubPath": "metadata",
    "x-ms-apimTemplateParameter.newPropertyPath": "metadataList",
    "x-ms-apimTemplateParameter.keyName": "name",
    "x-ms-apimTemplateParameter.valueName": "value",
    "x-ms-apimTemplate-policySection": "Response"
  }
}
```

---

### 8. stringtoarray — Convert Delimited String Into Array Of Objects

**Status:** Preview

Splits a delimited string property into an array of objects. Each object has a single child property holding the split token. This produces an array of objects, not an array of strings — `childPropertyName` is required.

**Use cases:**
- Split a semicolon-separated `tags` field into a `tagList` array
- Convert a comma-delimited `categories` string into a list for iteration
- Normalize a pipe-separated field from a legacy API into a structured array

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `x-ms-apimTemplateParameter.propertyParentPath` | JSON path to the parent object containing the string |
| `x-ms-apimTemplateParameter.propertySubPath` | Name of the string property to split |
| `x-ms-apimTemplateParameter.delimiterList` | Delimiter characters used to split (e.g., `";"`) |
| `x-ms-apimTemplateParameter.childPropertyName` | Property name for each token within the resulting array items |
| `x-ms-apimTemplateParameter.newPropertyPath` | Path for the resulting array |
| `x-ms-apimTemplate-policySection` | `Request` or `Response` |
| `x-ms-apimTemplate-operationName` | Array of operationIds. Omit for all operations |

**Example — Split semicolon-separated `tags` into `tagList` array with `value` property:**

```json
{
  "templateId": "stringtoarray",
  "title": "Split tags string into tagList array",
  "parameters": {
    "x-ms-apimTemplateParameter.propertyParentPath": "@body()",
    "x-ms-apimTemplateParameter.propertySubPath": "tags",
    "x-ms-apimTemplateParameter.delimiterList": ";",
    "x-ms-apimTemplateParameter.childPropertyName": "value",
    "x-ms-apimTemplateParameter.newPropertyPath": "tagList",
    "x-ms-apimTemplate-policySection": "Response"
  }
}
```

---

### 9. setvaluefromurl — Set Header/Query Parameter Value From URL

**Status:** Preview

Makes an internal HTTP call to another backend endpoint before forwarding the original request. Extracts a value from the internal response and sets it as a request header or query parameter. Use this before reaching for `script.csx`.

**Use cases:**
- Fetch an ETag or version token from a resource endpoint and inject it as `If-Match` before a PUT
- Retrieve a session token from an auth endpoint and set it as a header
- Look up a resource ID from a secondary endpoint and pass it as a query parameter

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `x-ms-apimTemplateParameter.parameterTemplate` | Target: `@headers('name')` or `@queryParameters('name')` |
| `x-ms-apimTemplateParameter.httpMethod` | HTTP method for the internal call |
| `x-ms-apimTemplateParameter.parameterValueUrl` | URL of the internal endpoint to call. Supports expressions |
| `x-ms-apimTemplateParameter.parameterValuePathTemplate` | JSON path to extract the value from the internal response |
| `x-ms-apimTemplate-policySection` | `Request` |
| `x-ms-apimTemplate-operationName` | Array of operationIds this applies to |

**Example — Fetch ETag from `/items/{id}/etag` and set as `If-Match` header before PUT:**

```json
{
  "templateId": "setvaluefromurl",
  "title": "Inject ETag as If-Match header",
  "parameters": {
    "x-ms-apimTemplateParameter.parameterTemplate": "@headers('If-Match')",
    "x-ms-apimTemplateParameter.httpMethod": "GET",
    "x-ms-apimTemplateParameter.parameterValueUrl": "https://api.myservice.example/items/@queryParameters('itemId')/etag",
    "x-ms-apimTemplateParameter.parameterValuePathTemplate": "etag",
    "x-ms-apimTemplate-policySection": "Request",
    "x-ms-apimTemplate-operationName": ["UpdateItem"]
  }
}
```

---

### 10. setconnectionstatustounauthenticated — Set Connection Status To Unauthenticated

**Status:** Preview

Marks the connection as unauthenticated when the backend returns a specific HTTP status code. Forces the user to reconnect. Must be paired with a test connection operation declared via `x-ms-capabilities.testConnection` in the swagger.

**Use cases:**
- Mark connection invalid when the backend returns 401 Unauthorized
- Force reconnection on token expiry detected by a specific status code
- Surface auth failures as connection errors rather than runtime errors

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `x-ms-apimTemplateParameter.statusCode` | HTTP status code (integer) that triggers the unauthenticated status |
| `x-ms-apimTemplate-operationName` | Array of operationIds. Typically scoped to the test connection operation |

**Example — Mark connection invalid when backend returns 401:**

```json
{
  "templateId": "setconnectionstatustounauthenticated",
  "title": "Mark connection unauthenticated on 401",
  "parameters": {
    "x-ms-apimTemplateParameter.statusCode": 401,
    "x-ms-apimTemplate-operationName": ["TestConnection"]
  }
}
```
