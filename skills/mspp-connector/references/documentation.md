# Connector Documentation — Formats and Generation

When the user requests documentation, generate both files into the connector folder alongside the connector files.

| File | Format | Purpose |
|------|--------|---------|
| `README.md` | Markdown | Human-readable reference for developers and makers |
| `swagger-ui.html` | HTML (Swagger UI) | Interactive browser-based API explorer |

---

## Format 1: README.md

Use this exact structure. Populate every section from the connector's `apiDefinition.swagger.json` and `apiProperties.json`.

````markdown
# [Connector Title]

> **Version:** [info.version]
> **Publisher:** [info.contact.name or publisher name]
> **Description:** [info.description]

---

## Authentication

**Type:** [Auth type: OAuth 2.0 / API Key / Basic / Windows / None]

[One paragraph describing what credentials the maker needs and where to find them. For OAuth 2.0, include scopes. For API Key, state the header or query parameter name. For Basic, state the username/password source.]

---

## Connection Setup

To create a connection in Power Automate or Power Apps:

1. [Step — e.g., "Open the connector and click 'New connection'"]
2. [Step — e.g., "Enter your API key from the service's account settings"]
3. [Step — e.g., "Click 'Create'"]

---

## Operations

Operations are grouped by visibility. **Important** operations are shown first; **Advanced** operations follow.

---

### [Operation Summary]

**Operation ID:** `[operationId]`
**Method:** `[HTTP method]`
**Path:** `[path]`
**Visibility:** Important / Advanced

[operation.description — full sentence(s)]

#### Parameters

| Name | Location | Type | Required | Description |
|------|----------|------|----------|-------------|
| `[name]` | [query/path/header/body] | [type] | Yes / No | [description] |

> Omit this table if the operation has no parameters.

#### Request Body

> Omit this section if the operation has no request body.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `[property]` | [type] | Yes / No | [description] |

#### Responses

| Code | Description |
|------|-------------|
| [code] | [description] |

---

[Repeat the operation block for each operation in the connector]

---

## Policy Notes

[If policyTemplateInstances are configured, briefly describe what they do in plain English. Example: "All requests automatically include an Accept: application/json header." Omit this section if there are no policy templates.]

---

## Known Limitations

[List any platform constraints relevant to this connector — e.g., "This connector uses script.csx; the 2-minute execution timeout applies." Or "Sandbox and production require separate connections." Omit if none apply.]
````

---

## Format 2: swagger-ui.html

Generate a standalone HTML file that references the connector's `apiDefinition.swagger.json` by relative path. Place it in the same folder as the connector files. No build step or server is required — open the file in any browser.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>[Connector Title] — API Reference</title>
  <link rel="stylesheet" href="https://unpkg.com/swagger-ui-dist@5/swagger-ui.css" />
  <style>
    body { margin: 0; }
    .topbar { display: none; }
  </style>
</head>
<body>
  <div id="swagger-ui"></div>
  <script src="https://unpkg.com/swagger-ui-dist@5/swagger-ui-bundle.js"></script>
  <script>
    SwaggerUIBundle({
      url: "./apiDefinition.swagger.json",
      dom_id: "#swagger-ui",
      presets: [
        SwaggerUIBundle.presets.apis,
        SwaggerUIBundle.SwaggerUIStandalonePreset
      ],
      layout: "BaseLayout",
      deepLinking: true,
      defaultModelsExpandDepth: 1,
      defaultModelExpandDepth: 1
    });
  </script>
</body>
</html>
```

Replace `[Connector Title]` in the `<title>` tag with the connector's display name.

**To view:** Open `swagger-ui.html` in a browser. If the browser blocks the local file fetch due to CORS, serve it with any local static file server:

```bash
# Python (macOS / Linux / Windows)
python -m http.server 8080
# Then open: http://localhost:8080/swagger-ui.html
```

---

## What to Tell the User

After generating both files, tell the user:

- `README.md` — human-readable reference, suitable for a GitHub repo or internal wiki
- `swagger-ui.html` — open in a browser for the interactive Swagger UI explorer; if it shows a blank page due to local file restrictions, serve with `python -m http.server 8080` and visit `http://localhost:8080/swagger-ui.html`
