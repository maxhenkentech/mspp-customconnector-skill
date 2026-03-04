# Custom Code (script.csx) — Power Platform Custom Connectors

Source: https://learn.microsoft.com/en-us/connectors/custom-connectors/write-code

> `script.csx` is the last resort. Exhaust all policy template options before writing custom code.

---

## When to Use

Use `script.csx` only when ALL of these conditions are true:

1. Policy templates cannot solve the requirement.
2. The transformation requires conditional branching on request or response content, OR the API requires SOAP/XML envelopes.
3. The logic exceeds what `setvaluefromurl` can handle for secondary HTTP calls.

---

## Decision Checklist Before Writing Custom Code

Work through this list before creating a `script.csx` file. If any item can be answered YES, use the policy template instead.

- [ ] Can `setheader` or `setqueryparameter` handle auth or header injection?
- [ ] Can `setproperty`, `convertarraytoobject`, or `convertobjecttoarray` handle the transform?
- [ ] Can `routerequesttoendpoint` handle the routing?
- [ ] Can `setvaluefromurl` handle the secondary API call?

If all four answers are NO, `script.csx` is appropriate.

---

## Constraints

| Constraint | Value |
|-----------|-------|
| Max execution time | 2 minutes |
| Max file size | 1 MB |
| Script files per connector | 1 |
| Runtime | .NET Standard 2.0 |
| On-premises gateway | Not supported |
| `HttpClient` (direct) | Deprecated — use `Context.SendAsync` |

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
