# Authentication Patterns — Power Platform Custom Connectors

Source: https://learn.microsoft.com/en-us/connectors/custom-connectors/define-auth-custom-connectors

---

## Choosing an Auth Type

Ask the API owner or check the API documentation to determine the correct type. The options below are all supported by Power Platform.

| Auth Type | When to use |
|-----------|------------|
| No auth | Public APIs that require no credentials |
| API Key | APIs that accept a single key in a header or query parameter |
| Basic | APIs that use HTTP Basic auth (username + password) |
| OAuth 2.0 — Authorization Code | APIs where the user grants delegated access interactively (most common for SaaS) |
| OAuth 2.0 — Client Credentials | Service-to-service APIs where there is no interactive user sign-in |
| Windows | On-premises APIs behind Windows Authentication (requires on-premises data gateway) |

---

## Multiple Environments (connectionParameterSets)

If the API has more than one environment (e.g., sandbox and production, or a per-tenant base URL), use `connectionParameterSets` instead of `connectionParameters`. This lets the maker choose the environment when creating a connection, and the correct host is applied at runtime via the `dynamichosturl` policy template.

See the full `connectionParameterSets` pattern in `references/swagger-support.md`.

**Rule:** If a single set of connection parameters applies to all users (one fixed host), use `connectionParameters`. If the host or auth endpoint differs per environment or per tenant, use `connectionParameterSets`.

---

## Pattern 1: No Authentication

```json
{
  "properties": {
    "connectionParameters": {}
  }
}
```

No parameters are needed. The swagger `host` is used as-is.

---

## Pattern 2: API Key — Header Injection

The user supplies a key at connection time. A `setheader` policy template injects it on every request. The key never appears in the swagger definition.

**apiProperties.json:**

```json
{
  "properties": {
    "connectionParameters": {
      "apiKey": {
        "type": "securestring",
        "uiDefinition": {
          "displayName": "API Key",
          "description": "Your API key. Find this in your account settings.",
          "tooltip": "Paste the API key from your account dashboard.",
          "constraints": {
            "required": "true"
          }
        }
      }
    },
    "policyTemplateInstances": [
      {
        "templateId": "setheader",
        "title": "Inject API key header",
        "parameters": {
          "x-ms-apimTemplateParameter.name": "X-Api-Key",
          "x-ms-apimTemplateParameter.value": "@connectionParameters('apiKey')",
          "x-ms-apimTemplateParameter.existsAction": "override",
          "x-ms-apimTemplate-policySection": "Request"
        }
      }
    ]
  }
}
```

**Deploying with paconn:** No `--secret` flag required.

---

## Pattern 3: API Key — Query Parameter Injection

Same as above but injects the key as a query string parameter instead of a header.

**apiProperties.json:**

```json
{
  "properties": {
    "connectionParameters": {
      "apiKey": {
        "type": "securestring",
        "uiDefinition": {
          "displayName": "API Key",
          "description": "Your API key.",
          "constraints": {
            "required": "true"
          }
        }
      }
    },
    "policyTemplateInstances": [
      {
        "templateId": "setqueryparameter",
        "title": "Inject API key query parameter",
        "parameters": {
          "x-ms-apimTemplateParameter.name": "api_key",
          "x-ms-apimTemplateParameter.value": "@connectionParameters('apiKey')",
          "x-ms-apimTemplateParameter.existsAction": "override"
        }
      }
    ]
  }
}
```

**Deploying with paconn:** No `--secret` flag required.

---

## Pattern 4: Basic Authentication

The user provides a username and password at connection time. Power Platform handles the `Authorization: Basic <base64>` header construction automatically when `type: "string"` and `type: "securestring"` are used together as `username` and `password` keys.

**apiProperties.json:**

```json
{
  "properties": {
    "connectionParameters": {
      "username": {
        "type": "string",
        "uiDefinition": {
          "displayName": "Username",
          "description": "Your account username.",
          "constraints": {
            "required": "true"
          }
        }
      },
      "password": {
        "type": "securestring",
        "uiDefinition": {
          "displayName": "Password",
          "description": "Your account password.",
          "constraints": {
            "required": "true"
          }
        }
      }
    }
  }
}
```

**Deploying with paconn:** No `--secret` flag required.

---

## Pattern 5: OAuth 2.0 — Authorization Code

The most common pattern for SaaS APIs. The maker is redirected to the provider's login page to grant access. Power Platform handles the token exchange and refresh cycle automatically.

**Required information from the API docs:**
- Authorization URL (the endpoint the maker is redirected to)
- Token URL (the endpoint that exchanges the auth code for a token)
- Refresh URL (usually the same as the token URL)
- Client ID (your registered app's ID)
- Client secret (your registered app's secret — provided at deploy time, not stored in the file)
- Scopes (the permissions requested)

**apiProperties.json:**

```json
{
  "properties": {
    "connectionParameters": {
      "token": {
        "type": "oauthSetting",
        "oAuthSettings": {
          "identityProvider": "oauth2",
          "clientId": "YOUR_CLIENT_ID",
          "scopes": [
            "read",
            "write"
          ],
          "redirectMode": "Global",
          "redirectUrl": "https://global.consent.azure-apim.net/redirect",
          "properties": {
            "IsFirstParty": "False",
            "IsOnbehalfofLoginSupported": false
          },
          "customParameters": {
            "authorizationUrl": {
              "value": "https://api.example.com/oauth/authorize"
            },
            "tokenUrl": {
              "value": "https://api.example.com/oauth/token"
            },
            "refreshUrl": {
              "value": "https://api.example.com/oauth/token"
            }
          }
        },
        "uiDefinition": {
          "displayName": "Log in with Example Service",
          "description": "Sign in with your Example Service account.",
          "constraints": {
            "required": "true"
          }
        }
      }
    }
  }
}
```

**Deploying with paconn:** The `--secret` flag (`-r`) is **required** to pass the OAuth client secret. Never store the secret in `apiProperties.json` or `settings.json`.

On first deployment (no `settings.json` yet), pass all file paths explicitly:

```bash
# First deployment — explicit flags required
paconn create \
  --api-def apiDefinition.swagger.json \
  --api-prop apiProperties.json \
  --icon icon.png \
  --secret YOUR_OAUTH_CLIENT_SECRET

# If there is no script.csx, omit --script.
# If the secret is not yet fixed or will be provided by the user,
# pass "dummy" as a placeholder: --secret "dummy"

# Subsequent updates — use settings.json (after adding connectorId from first deploy)
paconn update -s settings.json --secret YOUR_OAUTH_CLIENT_SECRET
```

**Important notes:**

- `clientId` is safe to commit to source control. `clientSecret` is NOT — it is never written to any file.
- `redirectUrl` must be registered in your OAuth application as an allowed redirect URI. Use exactly `https://global.consent.azure-apim.net/redirect`.
- If the provider requires PKCE or additional parameters in the authorization request, add them under `customParameters`.

---

## Pattern 6: OAuth 2.0 — Client Credentials

Use when the connector authenticates as an application (service account), not as a specific user. There is no interactive login — the client ID and secret are exchanged directly for an access token.

**apiProperties.json:**

```json
{
  "properties": {
    "connectionParameters": {
      "token": {
        "type": "oauthSetting",
        "oAuthSettings": {
          "identityProvider": "oauth2",
          "clientId": "YOUR_CLIENT_ID",
          "scopes": [],
          "redirectMode": "Global",
          "redirectUrl": "https://global.consent.azure-apim.net/redirect",
          "properties": {
            "IsFirstParty": "False",
            "IsOnbehalfofLoginSupported": false
          },
          "customParameters": {
            "tokenUrl": {
              "value": "https://api.example.com/oauth/token"
            },
            "refreshUrl": {
              "value": "https://api.example.com/oauth/token"
            },
            "grantType": {
              "value": "client_credentials"
            }
          }
        },
        "uiDefinition": {
          "displayName": "Connect to Example Service",
          "description": "Connect using your application credentials.",
          "constraints": {
            "required": "true"
          }
        }
      }
    }
  }
}
```

**Deploying with paconn:** Same as Authorization Code — `--secret` is required. On first deployment pass all file paths explicitly; use `settings.json` for subsequent updates.

```bash
# First deployment
paconn create \
  --api-def apiDefinition.swagger.json \
  --api-prop apiProperties.json \
  --icon icon.png \
  --secret YOUR_OAUTH_CLIENT_SECRET

# Subsequent updates
paconn update -s settings.json --secret YOUR_OAUTH_CLIENT_SECRET
```

---

## Pattern 7: Windows Authentication

Use for on-premises APIs protected by Windows Integrated Authentication. Requires an on-premises data gateway — `script.csx` does NOT work with on-premises gateway connections.

**apiProperties.json:**

```json
{
  "properties": {
    "connectionParameters": {
      "username": {
        "type": "string",
        "uiDefinition": {
          "displayName": "Username",
          "description": "Windows domain username (DOMAIN\\username).",
          "constraints": {
            "required": "true"
          }
        }
      },
      "password": {
        "type": "securestring",
        "uiDefinition": {
          "displayName": "Password",
          "description": "Windows account password.",
          "constraints": {
            "required": "true"
          }
        }
      },
      "authType": {
        "type": "string",
        "allowedValues": [
          {
            "value": "windows"
          }
        ],
        "uiDefinition": {
          "displayName": "Authentication type",
          "description": "Authentication type — set to Windows.",
          "constraints": {
            "required": "true",
            "allowedValues": [
              {
                "text": "Windows",
                "value": "windows"
              }
            ]
          }
        }
      }
    }
  }
}
```

**Deploying with paconn:** No `--secret` flag required.

---

## Multi-Environment + OAuth 2.0 (connectionParameterSets)

When the API has multiple environments (e.g., sandbox and production) AND uses OAuth 2.0, use `connectionParameterSets` so the maker can select both the environment and the auth configuration at connection time. Each set has its own `oAuthSettings` with the appropriate authorization and token URLs for that environment.

```json
{
  "properties": {
    "connectionParameterSets": {
      "uiDefinition": {
        "displayName": "Environment",
        "description": "Select the environment to connect to."
      },
      "values": [
        {
          "name": "production",
          "uiDefinition": {
            "displayName": "Production",
            "description": "Connect to the production environment."
          },
          "parameters": {
            "token": {
              "type": "oauthSetting",
              "oAuthSettings": {
                "identityProvider": "oauth2",
                "clientId": "YOUR_PROD_CLIENT_ID",
                "scopes": ["read", "write"],
                "redirectMode": "Global",
                "redirectUrl": "https://global.consent.azure-apim.net/redirect",
                "properties": {
                  "IsFirstParty": "False",
                  "IsOnbehalfofLoginSupported": false
                },
                "customParameters": {
                  "authorizationUrl": {
                    "value": "https://account.example.com/oauth/authorize"
                  },
                  "tokenUrl": {
                    "value": "https://account.example.com/oauth/token"
                  },
                  "refreshUrl": {
                    "value": "https://account.example.com/oauth/token"
                  }
                }
              },
              "uiDefinition": {
                "displayName": "Sign in",
                "description": "Sign in with your production account.",
                "constraints": {
                  "required": "true"
                }
              }
            }
          }
        },
        {
          "name": "sandbox",
          "uiDefinition": {
            "displayName": "Sandbox",
            "description": "Connect to the sandbox environment for testing."
          },
          "parameters": {
            "token": {
              "type": "oauthSetting",
              "oAuthSettings": {
                "identityProvider": "oauth2",
                "clientId": "YOUR_SANDBOX_CLIENT_ID",
                "scopes": ["read", "write"],
                "redirectMode": "Global",
                "redirectUrl": "https://global.consent.azure-apim.net/redirect",
                "properties": {
                  "IsFirstParty": "False",
                  "IsOnbehalfofLoginSupported": false
                },
                "customParameters": {
                  "authorizationUrl": {
                    "value": "https://account-d.example.com/oauth/authorize"
                  },
                  "tokenUrl": {
                    "value": "https://account-d.example.com/oauth/token"
                  },
                  "refreshUrl": {
                    "value": "https://account-d.example.com/oauth/token"
                  }
                }
              },
              "uiDefinition": {
                "displayName": "Sign in",
                "description": "Sign in with your sandbox account.",
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
        "title": "Set host from selected environment",
        "parameters": {
          "x-ms-apimTemplateParameter.urlTemplate": "@connectionParameters('token/oAuthSettings/customParameters/tokenUrl/value')"
        }
      }
    ]
  }
}
```

> For multi-environment connectors, pair `connectionParameterSets` with the `dynamichosturl` policy template so the correct API host is applied per environment. The swagger `host` field remains a static placeholder but is overridden at runtime.

**Deploying with paconn:** If any set uses OAuth 2.0, `--secret` is required. On first deployment pass all file paths explicitly:

```bash
# First deployment
paconn create \
  --api-def apiDefinition.swagger.json \
  --api-prop apiProperties.json \
  --icon icon.png \
  --secret YOUR_OAUTH_CLIENT_SECRET

# Subsequent updates
paconn update -s settings.json --secret YOUR_OAUTH_CLIENT_SECRET
```
