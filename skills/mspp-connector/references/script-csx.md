# Custom Code (script.csx) — Power Platform Custom Connectors

Source: https://learn.microsoft.com/en-us/connectors/custom-connectors/write-code

> `script.csx` is the last resort. Exhaust all policy template options before writing custom code.

---

## When to Use

Use `script.csx` only when one of the following conditions applies:

1. **SOAP/XML APIs** — The backend requires a SOAP envelope or XML request/response that policy templates cannot construct or parse.
2. **Complex conditional transformation** — The request or response transform requires branching logic that no combination of policy templates can express.
3. **Dynamic schema from a non-OpenAPI endpoint** — The API has a metadata or schema endpoint that does not return an OpenAPI-compatible schema. Use `script.csx` to call that endpoint, parse the response, and append a `schema` property in a format that `x-ms-dynamic-schema` can consume. This enables Power Automate to show dynamic fields in the flow designer without a native schema endpoint. See the pattern below.
4. **Secondary HTTP calls beyond `setvaluefromurl`** — The secondary call requires parsing, conditional logic, or chaining that `setvaluefromurl` cannot handle.

---

## Decision Checklist Before Writing Custom Code

Work through this list before creating a `script.csx` file. If any item can be answered YES, use the policy template instead.

- [ ] Can `setheader` or `setqueryparameter` handle auth or header injection?
- [ ] Can `setproperty`, `convertarraytoobject`, or `convertobjecttoarray` handle the transform?
- [ ] Can `routerequesttoendpoint` handle the routing?
- [ ] Can `setvaluefromurl` handle the secondary API call?
- [ ] Does the API expose a schema endpoint that returns an OpenAPI-compatible schema? If YES, use `x-ms-dynamic-schema` directly without a script.

If all five answers are NO, `script.csx` is appropriate.

---

## Constraints and Restrictions

### Hard Limits

| Constraint | Value |
|-----------|-------|
| Max execution time | 2 minutes |
| Max file size | 1 MB |
| Script files per connector | 1 (one `script.csx` per connector) |
| Runtime | .NET Standard 2.0 |
| On-premises data gateway | Not supported |
| `HttpClient` (direct creation) | Currently allowed but **will be blocked** in a future release — use `Context.SendAsync` exclusively |
| Logging / tracing | No debug trace or logging output available to the developer (planned for a future release) |

### Execution Timeout — Important Note

The 2-minute execution timeout applies to **newly created connectors only**. Existing connectors created before the timeout was introduced will continue to run without a timeout limit until the connector is updated (i.e., saved with new content). After the first update, the 2-minute limit is enforced permanently. Plan accordingly before updating long-running connectors.

### Virtual Network (VNet) Restriction

In environments linked to a Virtual Network (VNet), `Context.SendAsync` routes requests through a **public endpoint only**. It **cannot reach private endpoints on the VNet**. This means `script.csx` cannot be used to call services that are only accessible from within the private network. If the backend API is behind a VNet and not publicly exposed, `script.csx` is not a viable solution.

### HTTP Transport

- Use `Context.SendAsync` to make all outbound HTTP calls. It inherits the connection parameters and platform-level security context.
- Creating `HttpClient` instances directly is currently permitted but **will be blocked in a future platform update**. Migrate all direct `HttpClient` usage to `Context.SendAsync` before that change is released.
- Do not use `HttpClientFactory` or other DI-based HTTP abstractions — they are not available in the script runtime.

### Compilation and Deployment Errors

Syntax errors or compilation failures in `script.csx` do not produce a friendly error message. The connector deployment (`paconn update`) may succeed, but at runtime the connector returns an **internal server error (HTTP 500)** with no descriptive detail. Test the script locally against a .NET Standard 2.0 target before deploying. If a runtime 500 error appears after deployment, the most common cause is a compilation failure in the script.

### Cross-Region OperationId Encoding

In certain regions or cross-region call paths, the `OperationId` passed to the script via `Context.OperationId` is **Base64-encoded** instead of plain text. Always use the `GetOperationId()` helper (see pattern below) to decode before comparison to avoid `default` case mismatches in the switch statement.

### Supported Namespaces (Fixed Set)

Only the namespaces listed in the Supported Namespaces section are available. NuGet package installation or custom assembly references are **not supported**. If a required library is not in the supported list, `script.csx` cannot be used for that requirement.

---

## Required Class Structure

Every `script.csx` file must define a class named `Script` that extends `ScriptBase` and overrides `ExecuteAsync`. Route by `OperationId` using a switch statement. Always handle the `default` case.

```csharp
public class Script : ScriptBase
{
    public override async Task<HttpResponseMessage> ExecuteAsync()
    {
        switch (this.Context.OperationId)
        {
            case "GetItem":
                return await HandleGetItem().ConfigureAwait(false);
            case "CreateItem":
                return await HandleCreateItem().ConfigureAwait(false);
            default:
                return new HttpResponseMessage(HttpStatusCode.BadRequest)
                {
                    Content = CreateJsonContent(
                        $"Unhandled OperationId: '{this.Context.OperationId}'"
                    )
                };
        }
    }
}
```

---

## ScriptBase and IScriptContext API

```csharp
public abstract class ScriptBase
{
    // Access the current request context
    public IScriptContext Context { get; }

    // Cancellation token passed by the platform
    public CancellationToken CancellationToken { get; }

    // Helper to create an HttpContent from a JSON string
    public static StringContent CreateJsonContent(string serializedJson);

    // Entry point — must be overridden
    public abstract Task<HttpResponseMessage> ExecuteAsync();
}

public interface IScriptContext
{
    // Unique ID for the current request (use for logging)
    string CorrelationId { get; }

    // The operationId from the swagger definition
    string OperationId { get; }

    // The incoming HTTP request (read and modify this)
    HttpRequestMessage Request { get; }

    // Logger for diagnostic output
    ILogger Logger { get; }

    // Send an HTTP request through the connector pipeline
    Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken
    );
}
```

---

## Supported Namespaces

The following namespaces are available without additional imports:

- `System`
- `System.Collections.Generic`
- `System.Linq`
- `System.Net`
- `System.Net.Http`
- `System.Net.Http.Headers`
- `System.Text`
- `System.Text.RegularExpressions`
- `System.Threading`
- `System.Threading.Tasks`
- `System.Web`
- `System.Xml`
- `System.Xml.Linq`
- `Microsoft.Extensions.Logging`
- `Newtonsoft.Json`
- `Newtonsoft.Json.Linq`

---

## Pattern: Forward and Transform Response

Use this when the request passes through unchanged but the response body needs transformation.

```csharp
private async Task<HttpResponseMessage> HandleGetItem()
{
    // Forward the original request to the backend
    var response = await this.Context.SendAsync(
        this.Context.Request,
        this.CancellationToken
    ).ConfigureAwait(false);

    // Only transform on success
    if (response.IsSuccessStatusCode)
    {
        var responseBody = await response.Content.ReadAsStringAsync()
            .ConfigureAwait(false);

        var result = JObject.Parse(responseBody);

        // Example: wrap the response in an envelope
        var transformed = new JObject
        {
            ["data"] = result,
            ["count"] = result["items"]?.Count() ?? 0
        };

        response.Content = CreateJsonContent(transformed.ToString());
    }

    return response;
}
```

---

## Pattern: Dynamic Schema from a Non-OpenAPI Metadata Endpoint

Use this pattern when the API has a metadata or field-description endpoint that returns schema information in a proprietary format (not OpenAPI). The script calls that endpoint, transforms the response into a JSON Schema object, and appends it to the response under a `schema` key. The connector's `x-ms-dynamic-schema` extension then points to this operation to provide dynamic field tokens in the Power Automate flow designer.

**How it fits into the connector definition:**

In `apiDefinition.swagger.json`, define a "get schema" operation and a "get data" operation. Set `x-ms-dynamic-schema` on the body parameter of the data operation to reference the schema operation:

```json
"/data": {
  "post": {
    "operationId": "CreateDataRecord",
    "summary": "Create a data record",
    "description": "Creates a new record with a dynamically determined structure.",
    "parameters": [
      {
        "name": "body",
        "in": "body",
        "required": true,
        "schema": {
          "type": "object",
          "x-ms-dynamic-schema": {
            "operationId": "GetDataSchema",
            "parameters": {
              "recordType": {
                "parameter": "recordType"
              }
            },
            "value-path": "schema"
          }
        }
      },
      {
        "name": "recordType",
        "in": "query",
        "required": true,
        "type": "string",
        "x-ms-summary": "Record Type",
        "description": "The type of record to create."
      }
    ]
  }
},
"/schema": {
  "get": {
    "operationId": "GetDataSchema",
    "summary": "Get data record schema",
    "description": "Returns the JSON Schema for the specified record type.",
    "x-ms-visibility": "internal",
    "parameters": [
      {
        "name": "recordType",
        "in": "query",
        "required": true,
        "type": "string",
        "x-ms-summary": "Record Type",
        "description": "The record type whose schema to retrieve."
      }
    ],
    "responses": {
      "200": {
        "description": "Schema retrieved successfully.",
        "schema": {
          "type": "object",
          "properties": {
            "schema": {
              "type": "object",
              "x-ms-summary": "Schema",
              "description": "The JSON Schema for the record type."
            }
          }
        }
      }
    }
  }
}
```

List `GetDataSchema` in `scriptOperations` in `apiProperties.json` to route it through the script.

**script.csx implementation:**

The script calls the proprietary metadata endpoint, maps the response into a standard JSON Schema, and returns it under the `schema` key.

```csharp
public class Script : ScriptBase
{
    public override async Task<HttpResponseMessage> ExecuteAsync()
    {
        switch (this.Context.OperationId)
        {
            case "GetDataSchema":
                return await HandleGetDataSchema().ConfigureAwait(false);
            default:
                return new HttpResponseMessage(HttpStatusCode.BadRequest)
                {
                    Content = CreateJsonContent(
                        $"Unhandled OperationId: '{this.Context.OperationId}'"
                    )
                };
        }
    }

    private async Task<HttpResponseMessage> HandleGetDataSchema()
    {
        // Extract the recordType query parameter from the incoming request URI
        var query = System.Web.HttpUtility.ParseQueryString(
            this.Context.Request.RequestUri.Query
        );
        var recordType = query["recordType"] ?? string.Empty;

        // Call the proprietary metadata endpoint on the backend
        // (the base host is inherited from Context.Request.RequestUri)
        var metadataUri = new UriBuilder(this.Context.Request.RequestUri)
        {
            Path = "/api/metadata/fields",
            Query = $"type={Uri.EscapeDataString(recordType)}"
        }.Uri;

        var metadataRequest = new HttpRequestMessage(HttpMethod.Get, metadataUri);

        // Forward existing auth headers from the original request
        foreach (var header in this.Context.Request.Headers)
        {
            metadataRequest.Headers.TryAddWithoutValidation(header.Key, header.Value);
        }

        var metadataResponse = await this.Context.SendAsync(
            metadataRequest,
            this.CancellationToken
        ).ConfigureAwait(false);

        var metadataBody = await metadataResponse.Content
            .ReadAsStringAsync()
            .ConfigureAwait(false);

        var metadataFields = JArray.Parse(metadataBody);

        // Build a JSON Schema object from the proprietary field list
        var properties = new JObject();
        var required = new JArray();

        foreach (var field in metadataFields)
        {
            var name = field["name"]?.ToString();
            var fieldType = field["type"]?.ToString();
            var label = field["label"]?.ToString() ?? name;
            var isRequired = field["required"]?.Value<bool>() ?? false;

            if (string.IsNullOrEmpty(name)) continue;

            // Map the backend field type to a JSON Schema type
            var jsonType = fieldType switch
            {
                "integer" => "integer",
                "boolean" => "boolean",
                "date" => "string",
                _ => "string"
            };

            properties[name] = new JObject
            {
                ["type"] = jsonType,
                ["x-ms-summary"] = label,
                ["description"] = $"The {label} field."
            };

            if (fieldType == "date")
            {
                properties[name]["format"] = "date-time";
            }

            if (isRequired)
            {
                required.Add(name);
            }
        }

        // Wrap in a schema object under the "schema" key
        // x-ms-dynamic-schema's "value-path": "schema" points here
        var schemaEnvelope = new JObject
        {
            ["schema"] = new JObject
            {
                ["type"] = "object",
                ["properties"] = properties,
                ["required"] = required
            }
        };

        return new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = CreateJsonContent(schemaEnvelope.ToString())
        };
    }
}
```

**Key points:**

- The `value-path: "schema"` in `x-ms-dynamic-schema` tells Power Automate to look for the schema inside the `schema` property of the response — match this key exactly.
- The schema returned must be a valid JSON Schema object (`type`, `properties`, `required`). Power Automate will surface each property as a dynamic content token in the flow designer.
- Use `x-ms-summary` and `description` on each property within the schema to control how the tokens appear to the maker.
- List only `GetDataSchema` (not the main data operation) in `scriptOperations` if the main data operation does not need script processing.

---

## Pattern: SOAP/XML Request

This is the primary use case for `script.csx`. Use when the backend requires a SOAP envelope that cannot be constructed with policy templates alone.

**Approach:**
1. Inject credentials into the request as custom headers using `setheader` policy templates in `apiProperties.json` (so they are never hardcoded in the script).
2. Read those headers in the script using a `GetHeader` helper.
3. Build the SOAP envelope using `XDocument` and `XElement`.
4. Send to the SOAP endpoint via `Context.SendAsync`.
5. Parse the XML response and return as JSON.

```csharp
public class Script : ScriptBase
{
    public override async Task<HttpResponseMessage> ExecuteAsync()
    {
        switch (this.Context.OperationId)
        {
            case "GetItem":
                return await HandleSoapGetItem().ConfigureAwait(false);
            default:
                return new HttpResponseMessage(HttpStatusCode.BadRequest)
                {
                    Content = CreateJsonContent(
                        $"Unhandled OperationId: '{this.Context.OperationId}'"
                    )
                };
        }
    }

    private async Task<HttpResponseMessage> HandleSoapGetItem()
    {
        // Read the original JSON body to get input parameters
        var requestBody = await this.Context.Request.Content
            .ReadAsStringAsync()
            .ConfigureAwait(false);
        var input = JObject.Parse(requestBody);
        var itemId = input["itemId"]?.ToString();

        // Read credentials injected by setheader policy templates
        var username = GetHeader("x-connector-username");
        var password = GetHeader("x-connector-password");

        // Define namespaces
        XNamespace soapEnv = "http://schemas.xmlsoap.org/soap/envelope/";
        XNamespace serviceNs = "http://www.example.com/service";

        // Build the SOAP envelope
        var envelope = new XDocument(
            new XElement(soapEnv + "Envelope",
                new XAttribute(XNamespace.Xmlns + "soapenv", soapEnv),
                new XAttribute(XNamespace.Xmlns + "svc", serviceNs),
                new XElement(soapEnv + "Header",
                    new XElement(serviceNs + "AuthHeader",
                        new XElement(serviceNs + "Username", username),
                        new XElement(serviceNs + "Password", password)
                    )
                ),
                new XElement(soapEnv + "Body",
                    new XElement(serviceNs + "SoapOperation",
                        new XElement(serviceNs + "itemId", itemId)
                    )
                )
            )
        );

        // Build the SOAP request
        var soapRequest = new HttpRequestMessage(
            HttpMethod.Post,
            this.Context.Request.RequestUri
        );
        soapRequest.Content = new StringContent(
            envelope.ToString(),
            System.Text.Encoding.UTF8,
            "text/xml"
        );
        soapRequest.Headers.Add("SOAPAction", "http://www.example.com/service/SoapOperation");

        // Send the SOAP request
        var soapResponse = await this.Context.SendAsync(
            soapRequest,
            this.CancellationToken
        ).ConfigureAwait(false);

        var soapResponseBody = await soapResponse.Content
            .ReadAsStringAsync()
            .ConfigureAwait(false);

        // Parse the XML response
        var xmlDoc = XDocument.Parse(soapResponseBody);
        XNamespace responseNs = "http://www.example.com/service";

        var operationResult = xmlDoc.Descendants(responseNs + "operationResult").FirstOrDefault();

        // Map XML fields to a JSON response
        var jsonResult = new JObject
        {
            ["itemId"] = itemId,
            ["field1"] = operationResult?.Element(responseNs + "field1")?.Value,
            ["field2"] = operationResult?.Element(responseNs + "field2")?.Value
        };

        var httpResponse = new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = CreateJsonContent(jsonResult.ToString())
        };
        return httpResponse;
    }

    private string GetHeader(string name)
    {
        if (this.Context.Request.Headers.TryGetValues(name, out var values))
        {
            return values.FirstOrDefault() ?? string.Empty;
        }
        return string.Empty;
    }
}
```

---

## Pattern: Base64-Encoded OperationId (Cross-Region Compatibility)

In some cross-region deployments, the `OperationId` passed to the script may be Base64-encoded. Add a decode step to handle both encoded and plain-text forms.

```csharp
private string GetOperationId()
{
    try
    {
        var decoded = System.Text.Encoding.UTF8.GetString(
            Convert.FromBase64String(this.Context.OperationId)
        );
        return decoded;
    }
    catch
    {
        // Not Base64 — return as-is
        return this.Context.OperationId;
    }
}
```

Use `GetOperationId()` instead of `this.Context.OperationId` in the switch statement when cross-region support is required.

---

## scriptOperations in apiProperties.json

Control which operations are routed through the script. Operations not listed bypass the script entirely.

```json
"scriptOperations": ["GetItem", "CreateItem"]
```

An empty array means all operations run through the script. Omit `scriptOperations` entirely if all operations should use the script.

---

## apiProperties.json Example with Script and setheader Policies

Use `setheader` policy templates to inject credentials as custom request headers before the script runs. The script reads these headers with `GetHeader`. This keeps credentials out of the script file and in the connection parameters where they belong.

```json
{
  "properties": {
    "connectionParameters": {
      "username": {
        "type": "string",
        "uiDefinition": {
          "displayName": "Username",
          "description": "Your service account username.",
          "constraints": {
            "required": "true"
          }
        }
      },
      "password": {
        "type": "securestring",
        "uiDefinition": {
          "displayName": "Password",
          "description": "Your service account password.",
          "constraints": {
            "required": "true"
          }
        }
      }
    },
    "scriptOperations": ["GetItem", "CreateItem"],
    "policyTemplateInstances": [
      {
        "templateId": "setheader",
        "title": "Inject username header for script",
        "parameters": {
          "x-ms-apimTemplateParameter.name": "x-connector-username",
          "x-ms-apimTemplateParameter.value": "@connectionParameters('username')",
          "x-ms-apimTemplateParameter.existsAction": "override",
          "x-ms-apimTemplate-policySection": "Request"
        }
      },
      {
        "templateId": "setheader",
        "title": "Inject password header for script",
        "parameters": {
          "x-ms-apimTemplateParameter.name": "x-connector-password",
          "x-ms-apimTemplateParameter.value": "@connectionParameters('password')",
          "x-ms-apimTemplateParameter.existsAction": "override",
          "x-ms-apimTemplate-policySection": "Request"
        }
      }
    ]
  }
}
```

The script reads `x-connector-username` and `x-connector-password` using `GetHeader`. These headers are stripped before the request leaves the platform — they are only visible within the script execution context.
