# Swagger 2.0 Support — Power Platform Custom Connectors

Source: https://learn.microsoft.com/en-us/connectors/custom-connectors/openapi-extensions

> Only OpenAPI 2.0 (Swagger) is supported. OpenAPI 3.0 is NOT supported.

---

## Required Top-Level Fields

Every connector swagger file must include these fields at the root level.

```json
{
  "swagger": "2.0",
  "info": {
    "title": "My Service Connector",
    "description": "Connects to My Service and exposes item management operations.",
    "version": "1.0"
  },
  "host": "api.myservice.example",
  "basePath": "/v1",
  "schemes": ["https"],
  "consumes": ["application/json"],
  "produces": ["application/json"],
  "paths": {}
}
```

**Rules:**

| Field | Rule |
|-------|------|
| `swagger` | Must be exactly `"2.0"` |
| `host` | Hostname only. No scheme (`https://`), no path (`/v1`). Valid: `api.myservice.example`. Invalid: `https://api.myservice.example/v1` |
| `basePath` | Must start with `/` |
| `schemes` | Only `"http"` and `"https"` are valid values |

---

## Operations

Each operation must satisfy all of these requirements:

| Requirement | Detail |
|-------------|--------|
| `operationId` | Required. Must be unique across all operations. PascalCase. Alphanumeric characters only. Platform enforces a maximum length. |
| `summary` | Required. Sentence case. Maximum 120 characters. No trailing period. |
| `description` | Required. Full sentence ending with a period. May be multi-sentence. |
| At least one response | Required. |
| At least one 2xx response | Required. |

**Example:**

```json
"/items/{itemId}": {
  "get": {
    "operationId": "GetItemById",
    "summary": "Get item by ID",
    "description": "Retrieves a single item using its unique identifier.",
    "parameters": [
      {
        "name": "itemId",
        "in": "path",
        "required": true,
        "type": "string",
        "x-ms-summary": "Item ID"
      }
    ],
    "responses": {
      "200": {
        "description": "Item retrieved successfully",
        "schema": {
          "$ref": "#/definitions/Item"
        }
      },
      "default": {
        "description": "Operation failed"
      }
    }
  }
}
```

---

## Parameters

### Supported `in` Locations

| Location | Notes |
|----------|-------|
| `query` | Query string parameter |
| `header` | Request header |
| `path` | Path segment. Must be `required: true` |
| `body` | Request body. Only one body parameter allowed per operation |
| `formData` | Form field. Cannot be mixed with `body` in the same operation |

### Rules

- Path parameters must be `required: true`.
- Only one `body` parameter is allowed per operation.
- Do not mix `body` and `formData` in the same operation.
- Body parameters must have a `schema` property.
- GET operations cannot have `body` or `formData` parameters.
- `string` with format `binary` is only valid at the top level of a `body` or `formData` parameter, or at the root of a response schema. It cannot appear inside a nested object.
- `collectionFormat` supported values: `csv`, `ssv`, `tsv`, `pipes`, `multi`.

**Example — body parameter:**

```json
{
  "name": "body",
  "in": "body",
  "required": true,
  "schema": {
    "$ref": "#/definitions/CreateItemRequest"
  }
}
```

**Example — formData parameter:**

```json
{
  "name": "file",
  "in": "formData",
  "required": true,
  "type": "string",
  "format": "binary",
  "x-ms-summary": "File"
}
```

---

## Schemas and Definitions

| Rule | Detail |
|------|--------|
| Arrays must define `items` | An array type without an `items` property is invalid |
| `required` array must have at least 1 value | If the `required` keyword is present, it must not be empty |
| `$ref` supported locations | `#/definitions/`, `#/parameters/`, `#/responses/` |
| No `$ref` on path items | Path item objects cannot use `$ref` |
| No circular `$ref` loops | Circular references cause validation failures |
| No `uniqueItems` keyword | The `uniqueItems` keyword is not supported |
| No sibling properties with `$ref` | When a `$ref` is present in an object, no other properties may appear in the same object |

**Example — valid definition:**

```json
"definitions": {
  "Item": {
    "type": "object",
    "properties": {
      "itemId": {
        "type": "string",
        "x-ms-summary": "Item ID"
      },
      "status": {
        "type": "string",
        "x-ms-summary": "Status"
      },
      "tags": {
        "type": "array",
        "items": {
          "type": "string"
        },
        "x-ms-summary": "Tags"
      }
    }
  }
}
```

---

## Security Definitions

Supported authentication types:

| Type | Description |
|------|-------------|
| `apiKey` | API key passed as a header or query parameter |
| `oauth2` | OAuth 2.0 (configured in apiProperties.json, not just swagger) |
| `basic` | HTTP Basic authentication |

For connectors that must support connections to different tenant instances (each with a different base URL), use `connectionParameterSets` in `apiProperties.json` rather than a single security definition. See the connectionParameterSets pattern below.

---

## Supported MIME Types

Use these values in `consumes` and `produces`:

- `application/json`
- `text/plain`
- `multipart/form-data`
- `application/x-www-form-urlencoded`

---

## Response Schemas

| Rule | Detail |
|------|--------|
| Successful responses should have a `schema` | Every 2xx response should declare a schema |
| `default` response must not have a `schema` | The `default` error response must not include a schema |
| Empty response schemas not allowed | A schema that is present but has no type or properties is not valid, except for dynamic responses |
| Dynamic responses | Use `x-ms-dynamic-schema` or `x-ms-dynamic-properties` on the response schema object when the shape varies at runtime |

### Define All Expected Status Codes

Do not limit responses to `200` and `default`. Define every status code the API documentation says the endpoint can return. This helps Power Automate surface accurate error handling to flow authors and prevents unexpected `default` branch routing.

**Common status codes by operation type:**

| Code | Meaning | Typical operations |
|------|---------|-------------------|
| `200` | OK | GET, PUT, PATCH, DELETE with body |
| `201` | Created | POST that creates a resource |
| `204` | No Content | DELETE, PUT/PATCH with no response body |
| `400` | Bad Request | Any — invalid input, missing required fields |
| `401` | Unauthorized | Any — auth token missing or expired |
| `403` | Forbidden | Any — valid auth but insufficient permissions |
| `404` | Not Found | GET/PUT/DELETE by ID |
| `409` | Conflict | POST/PUT — resource already exists or version conflict |
| `422` | Unprocessable Entity | POST/PUT — semantic validation failure |
| `429` | Too Many Requests | Any — rate limit hit |
| `500` | Internal Server Error | Any — unexpected server-side failure |

**Example — GET by ID with full response coverage:**

```json
"responses": {
  "200": {
    "description": "Item retrieved successfully.",
    "schema": {
      "$ref": "#/definitions/Item"
    }
  },
  "400": {
    "description": "The request is invalid. Check the parameter values."
  },
  "401": {
    "description": "Authentication failed. Reconnect the connection."
  },
  "403": {
    "description": "Access denied. You do not have permission to read this item."
  },
  "404": {
    "description": "Item not found. The identifier does not match any existing item."
  },
  "429": {
    "description": "Rate limit exceeded. Retry after the period specified in the Retry-After header."
  },
  "500": {
    "description": "An unexpected server error occurred."
  },
  "default": {
    "description": "Operation failed."
  }
}
```

**Note:** Error response bodies (`4xx`, `5xx`) should not include a `schema` unless the API returns a consistent, structured error object. If the API does return a structured error body, define an `ErrorResponse` definition and reference it.

---

## Reserved Parameter Names

| Name | Reason |
|------|--------|
| `connectionId` | Reserved by the platform. Cannot be used as a parameter name in any operation |

---

## Path Rules

- No wildcard `*` in paths.
- `?` and `#` characters in paths must be URL-encoded.
- No duplicate paths. Path comparison is case-insensitive.
- No duplicate `operationId` values across any path or HTTP method.

---

## Not Supported

| Feature | Status |
|---------|--------|
| OpenAPI 3.0 | Not supported — use Swagger 2.0 only |
| `uniqueItems` keyword | Not supported |
| Circular `$ref` | Not supported |
| `$ref` on path items | Not supported |
| Sibling properties alongside `$ref` | Not supported |
| Wildcard `*` in paths | Not supported |
| `body` + `formData` in same operation | Not supported |
| Multiple `body` parameters per operation | Not supported |
| `GET` with `body` or `formData` parameters | Not supported |
| `string/binary` nested inside object schemas | Not supported — only valid at top-level body/formData or root response schema |
| On-premises data gateway with `script.csx` | Not supported |
| Service Principal auth via `paconn` | Not supported — use `pac` CLI |

---

## connectionParameterSets Pattern

Use `connectionParameterSets` in `apiProperties.json` when each connection points to a different base URL (multi-tenant or multi-instance connectors). Pair with the `dynamichosturl` policy template to apply the connection's URL at runtime.

```json
{
  "properties": {
    "connectionParameterSets": {
      "uiDefinition": {
        "displayName": "Authentication type",
        "description": "Select how to connect to My Service."
      },
      "values": [
        {
          "name": "apiKeyAuth",
          "uiDefinition": {
            "displayName": "API Key",
            "description": "Connect using an API key."
          },
          "parameters": {
            "instanceUrl": {
              "type": "string",
              "uiDefinition": {
                "displayName": "Instance URL",
                "description": "The base URL of your My Service instance (e.g. https://myinstance.myservice.example).",
                "tooltip": "Find this in your My Service account settings.",
                "constraints": {
                  "required": "true"
                }
              }
            },
            "apiKey": {
              "type": "securestring",
              "uiDefinition": {
                "displayName": "API Key",
                "description": "Your My Service API key.",
                "constraints": {
                  "required": "true"
                }
              }
            }
          }
        },
        {
          "name": "basicAuth",
          "uiDefinition": {
            "displayName": "Basic Authentication",
            "description": "Connect using a username and password."
          },
          "parameters": {
            "instanceUrl": {
              "type": "string",
              "uiDefinition": {
                "displayName": "Instance URL",
                "description": "The base URL of your My Service instance.",
                "constraints": {
                  "required": "true"
                }
              }
            },
            "username": {
              "type": "string",
              "uiDefinition": {
                "displayName": "Username",
                "constraints": {
                  "required": "true"
                }
              }
            },
            "password": {
              "type": "securestring",
              "uiDefinition": {
                "displayName": "Password",
                "constraints": {
                  "required": "true"
                }
              }
            }
          }
        }
      ]
    },
    "policyTemplateInstances": [
      {
        "templateId": "dynamichosturl",
        "title": "Set host from connection parameter",
        "parameters": {
          "x-ms-apimTemplateParameter.urlTemplate": "@connectionParameters('instanceUrl')"
        }
      }
    ]
  }
}
```

The `instanceUrl` connection parameter provides the runtime host. The swagger `host` field still needs a valid placeholder value (e.g., `api.myservice.example`) but is overridden at runtime by the policy template.
