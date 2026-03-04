# Microsoft Power Platform Custom Connector Skill for Claude Code

> **GitHub:** [https://github.com/maxhenkentech/mspp-customconnector-skill](https://github.com/maxhenkentech/mspp-customconnector-skill)

A [Claude Code](https://claude.ai/code) plugin that gives Claude deep, production-ready expertise in building and maintaining **Microsoft Power Platform custom connectors** — from a blank folder to a deployed, validated connector with dynamic UI, policy transforms, and optional C# custom code.

Whether you are building your first connector or tuning a complex multi-instance OAuth connector with dynamic schemas, this skill knows the full stack.

---

## What This Skill Covers

### Full Connector Lifecycle
- Scaffold the correct file structure (`apiDefinition.swagger.json`, `apiProperties.json`, `icon.png`, `script.csx`, `settings.json`)
- Author or import OpenAPI 2.0 definitions with all required fields
- Validate with `paconn validate` before every deployment — no exceptions
- Deploy with `paconn create` or `paconn update`, with guided interactive prompts and `settings.json` management

### Quality Enforcement
Every operation, parameter, and response property is checked against the Power Platform quality rules:
- `operationId` — unique, PascalCase, alphanumeric only
- `summary` — sentence case, ≤ 120 characters, no trailing period
- `description` — full sentence with trailing period
- `x-ms-summary` — title case label for the Power Automate designer
- `x-ms-visibility` — `important`, `advanced`, or `internal` with correct semantics

### All 10 Policy Templates
Covers every policy template available in the platform — with templateId, parameters, expression syntax, and copy-paste JSON examples:

| Template | Status |
|----------|--------|
| `setheader` — Set HTTP Header | GA |
| `setqueryparameter` — Set Query String Parameter | GA |
| `dynamichosturl` — Dynamic Host URL | GA |
| `routerequesttoendpoint` — Route Request to Endpoint | GA |
| `setproperty` — Set Property | Preview |
| `convertarraytoobject` — Convert Array to Object | Preview |
| `convertobjecttoarray` — Convert Object to Array | Preview |
| `stringtoarray` — Split Delimited String into Array | Preview |
| `setvaluefromurl` — Set Header/Query from URL | Preview |
| `setconnectionstatustounauthenticated` — Force Reconnect on Auth Failure | Preview |

### Dynamic UI Patterns
- `x-ms-dynamic-values` + `x-ms-dynamic-list` — populate dropdowns from a live API call
- `x-ms-dynamic-schema` + `x-ms-dynamic-properties` — change the input/output schema based on prior selections
- `parameterReference` for body-level parameter wiring
- Covers both legacy and new extension variants for forward compatibility

### Webhook and Trigger Connectors
- `x-ms-trigger: single` and `x-ms-trigger: batch`
- `x-ms-notification-url` and `x-ms-notification-content` patterns
- Paired register/deregister (POST + DELETE) webhook setup
- Internal visibility rules for platform-managed operations

### x-ms Extension Catalog
Full reference for every supported `x-ms-*` extension: visibility, summaries, connector metadata, versioning/deprecation, URL encoding, enumerations, trigger hints, capability flags, and certification metadata.

### Supported Swagger Properties
Explicit guidance on what Power Platform supports and what it rejects — including unsupported patterns like OpenAPI 3.0, circular `$ref`, body parameters in GET requests, and duplicate `operationId` values.

### Custom Code (`script.csx`) — When Everything Else Falls Short
The skill positions `script.csx` as a last resort and works through a checklist to make sure a policy template cannot solve the problem first. When a script is appropriate, it provides:
- Full `ScriptBase` / `IScriptContext` API reference
- All supported .NET namespaces
- Pattern: Forward and transform a response
- Pattern: SOAP/XML request with envelope construction and XML-to-JSON parsing
- Pattern: Generate a dynamic schema from a proprietary metadata endpoint (enabling dynamic field tokens in Power Automate without an OpenAPI schema endpoint)
- Cross-region Base64 `OperationId` decoding
- All platform constraints: 2-minute timeout (and the edge case for existing connectors), 1 MB file limit, VNet restriction, `HttpClient` deprecation, no on-premises gateway support, compilation error behavior

### paconn CLI — Full Reference
- Installation and authentication
- All commands: `login`, `logout`, `list`, `download`, `validate`, `create`, `update`
- `settings.json` schema for storing environment and connector IDs locally
- Guided workflows for creating a new connector, updating an existing one, and downloading for editing
- Known limitations: the `stackOwner` field that blocks `paconn update`, Service Principal limitations, environment GUID requirements

---

## Who This Is For

- **Beginners** who are building their first Power Platform connector and need step-by-step guidance without reading dozens of documentation pages
- **Intermediate developers** who know the basics but need the `x-ms-*` extension catalog, policy template parameters, or dynamic schema patterns
- **Advanced developers** maintaining production connectors who need accurate, complete reference material for complex scenarios like multi-instance OAuth, SOAP backends, or dynamic property schemas
- **Anyone** who has wasted time debugging a connector deployment only to find out the `operationId` had an underscore, the `summary` was too long, or the `swagger` field said `3.0`

---

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- Python 3.8+ (for `paconn`)

---

## Installation

### macOS

```bash
# 1. Install the paconn CLI (required to deploy connectors)
pip install paconn

# 2. Create the Claude Code plugins directory if it does not exist
mkdir -p ~/.claude/plugins

# 3. Clone this repository into the plugins directory
git clone https://github.com/maxhenkentech/mspp-customconnector-skill.git \
  ~/.claude/plugins/mspp-custom-connector-builder

# 4. Restart Claude Code — the skill loads automatically
```

### Windows

Open **PowerShell** (or Windows Terminal):

```powershell
# 1. Install the paconn CLI (required to deploy connectors)
pip install paconn

# 2. Create the Claude Code plugins directory if it does not exist
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\plugins"

# 3. Clone this repository into the plugins directory
git clone https://github.com/maxhenkentech/mspp-customconnector-skill.git `
  "$env:USERPROFILE\.claude\plugins\mspp-custom-connector-builder"

# 4. Restart Claude Code — the skill loads automatically
```

> **No manual configuration required.** Claude Code discovers the plugin automatically from the `plugins` directory.

---

## Usage

Once installed, the skill activates automatically when you ask Claude about anything related to Power Platform custom connectors. You do not need to invoke it manually.

### Example prompts that trigger the skill

```
Build a custom connector for the Acme REST API
```
```
Add x-ms-dynamic-values to this dropdown parameter
```
```
Validate and deploy my connector with paconn
```
```
Add a policy template to inject my API key as a header
```
```
Write script.csx to talk to a SOAP API
```
```
Add a webhook trigger to my connector
```
```
My connector has a dynamic schema based on the record type selected
```
```
What swagger properties are not supported in Power Platform?
```

---

## Plugin Structure

```
mspp-custom-connector-builder/
├── plugin.json                          # Plugin manifest
└── skills/
    └── mspp-connector/
        ├── SKILL.md                     # Core workflow guide (~1 800 words)
        └── references/
            ├── paconn-guide.md          # Full paconn CLI reference
            ├── policy-templates.md      # All 10 policy templates with examples
            ├── x-ms-extensions.md       # Complete x-ms-* extension catalog
            ├── dynamic-patterns.md      # Dynamic values, lists, schema, properties
            ├── swagger-support.md       # Supported vs unsupported Swagger 2.0
            └── script-csx.md            # Custom code patterns and constraints
```

Reference files are loaded on demand — only when the specific topic is needed. This keeps the context window lean for simple tasks and comprehensive for complex ones.

---

## Contributing

Issues and pull requests are welcome. If a policy template parameter is wrong, a swagger rule has changed, or a paconn flag is missing, please open an issue or submit a fix.

---

## License

MIT
