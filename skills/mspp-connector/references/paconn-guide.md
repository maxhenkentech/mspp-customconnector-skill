# paconn CLI Reference — Power Platform Custom Connectors

Source: https://learn.microsoft.com/en-us/connectors/custom-connectors/paconn-cli

---

## Installation

```bash
pip install paconn
# If access denied:
pip install paconn --user
```

Requirements:
- Python 3.5 or later
- Add Python to PATH during installation
- Verify installation: `paconn --help`

---

## Commands

### paconn login

Authenticates using device-code flow. Opens a browser prompt for interactive sign-in. No Service Principal support — use `pac` CLI for CI/CD pipelines.

```bash
paconn login
```

After login, credentials are cached for subsequent commands.

---

### paconn logout

Clears the cached credentials.

```bash
paconn logout
```

---

### paconn validate

**Run validate before every create or update.** Prints all errors, warnings, and a final success or failure message. Fix all errors before proceeding.

```bash
paconn validate --api-def apiDefinition.swagger.json
paconn validate -s settings.json
paconn validate --api-def apiDefinition.swagger.json --pau https://api.powerapps.com
```

| Argument | Short | Description |
|----------|-------|-------------|
| `--api-def` | | Path to the OpenAPI (swagger) definition file |
| `--pau` | `-u` | Power Apps URL. Default: `https://api.powerapps.com` |
| `--pav` | `-v` | Power Apps API version. Default: `2016-11-01` |
| `--settings` | `-s` | Path to settings.json. Overrides individual flags |

---

### paconn download

Downloads the connector files into a subdirectory named after the connector ID. Also writes a `settings.json` file into the download directory.

```bash
paconn download
paconn download --cid abc123 --env 00000000-0000-0000-0000-000000000000
paconn download -s settings.json --dest ./connectors
```

| Argument | Short | Description |
|----------|-------|-------------|
| `--cid` | `-c` | Connector ID (from Power Platform admin or portal) |
| `--dest` | `-d` | Destination directory for downloaded files. Default: current directory |
| `--env` | `-e` | Environment GUID. Always specify explicitly |
| `--overwrite` | `-w` | Overwrite existing files without prompting |
| `--pau` | `-u` | Power Apps URL |
| `--pav` | `-v` | Power Apps API version |
| `--settings` | `-s` | Path to settings.json |

---

### paconn create

Creates a new custom connector. Prints the new connector ID on success. After creation, add the printed connector ID to `settings.json` under `connectorId`.

```bash
# First deployment — settings.json does not exist yet, pass files explicitly
paconn create --api-def apiDefinition.swagger.json --api-prop apiProperties.json --icon icon.png
paconn create --api-def apiDefinition.swagger.json --api-prop apiProperties.json --script script.csx --secret YOUR_SECRET

# Subsequent use — after settings.json contains connectorId
paconn create -s settings.json
paconn create -s settings.json --env 00000000-0000-0000-0000-000000000000
```

| Argument | Short | Description |
|----------|-------|-------------|
| `--api-def` | | Path to the OpenAPI (swagger) definition file |
| `--api-prop` | | Path to apiProperties.json |
| `--env` | `-e` | Environment GUID |
| `--icon` | | Path to icon.png (optional) |
| `--script` | `-x` | Path to script.csx (optional) |
| `--pau` | `-u` | Power Apps URL |
| `--pav` | `-v` | Power Apps API version |
| `--secret` | `-r` | OAuth 2.0 client secret. Required when apiProperties defines OAuth |
| `--settings` | `-s` | Path to settings.json |

---

### paconn update

Updates an existing connector. Requires `--cid` or `connectorId` in settings.json.

```bash
paconn update --api-def apiDefinition.swagger.json --api-prop apiProperties.json --cid abc123
paconn update -s settings.json
paconn update -s settings.json --secret <oauth-client-secret>
```

| Argument | Short | Description |
|----------|-------|-------------|
| `--cid` | `-c` | Connector ID (required if not in settings.json) |
| `--api-def` | | Path to the OpenAPI (swagger) definition file |
| `--api-prop` | | Path to apiProperties.json |
| `--env` | `-e` | Environment GUID |
| `--icon` | | Path to icon.png (optional) |
| `--script` | `-x` | Path to script.csx (optional) |
| `--pau` | `-u` | Power Apps URL |
| `--pav` | `-v` | Power Apps API version |
| `--secret` | `-r` | OAuth 2.0 client secret |
| `--settings` | `-s` | Path to settings.json |

---

## settings.json Schema

`settings.json` consolidates all CLI arguments into a single file. Pass it with `-s settings.json` to any command.

| Field | Required | Description |
|-------|----------|-------------|
| `connectorId` | Required for download/update. Absent for create. | Connector ID from Power Platform |
| `environment` | Recommended | Environment GUID from Power Platform admin center |
| `apiProperties` | Required | Path to apiProperties.json |
| `apiDefinition` | Required | Path to the swagger (OpenAPI 2.0) file |
| `icon` | Optional | Path to icon.png |
| `script` | Optional | Path to script.csx |
| `powerAppsApiVersion` | Optional | API version. Default: `"2016-11-01"` |
| `powerAppsUrl` | Optional | Power Apps base URL. Default: `"https://api.powerapps.com"` |

### Complete settings.json Example

```json
{
  "connectorId": "shared_myconnector-5fabc123",
  "environment": "00000000-0000-0000-0000-000000000000",
  "apiProperties": "apiProperties.json",
  "apiDefinition": "apiDefinition.swagger.json",
  "icon": "icon.png",
  "script": "script.csx",
  "powerAppsApiVersion": "2016-11-01",
  "powerAppsUrl": "https://api.powerapps.com"
}
```

For a new connector (before creation), omit `connectorId`:

```json
{
  "environment": "00000000-0000-0000-0000-000000000000",
  "apiProperties": "apiProperties.json",
  "apiDefinition": "apiDefinition.swagger.json",
  "icon": "icon.png",
  "powerAppsApiVersion": "2016-11-01",
  "powerAppsUrl": "https://api.powerapps.com"
}
```

---

## Guided Workflows

### Workflow A: New Connector

Use when creating a connector for the first time. `settings.json` does not exist yet — pass all file paths explicitly.

```bash
# Step 1: Authenticate
paconn login

# Step 2: Validate before creating
paconn validate --api-def apiDefinition.swagger.json

# Step 3: Create the connector — pass each file explicitly
# Omit --script if there is no script.csx
# Omit --icon if there is no icon.png
# For OAuth 2.0 connectors, --secret is required.
# Pass --secret "dummy" if the secret is not yet known or is user-supplied.
paconn create \
  --api-def apiDefinition.swagger.json \
  --api-prop apiProperties.json \
  --script script.csx \
  --icon icon.png \
  --secret YOUR_OAUTH_CLIENT_SECRET

# Step 4: Copy the printed connector ID into settings.json under "connectorId"
# Example output: Connector ID: shared_myconnector-5fabc123
```

After step 4, create `settings.json` with the returned connector ID so all future updates use `-s settings.json`:

```json
{
  "connectorId": "shared_myconnector-5fabc123",
  "environment": "00000000-0000-0000-0000-000000000000",
  "apiProperties": "apiProperties.json",
  "apiDefinition": "apiDefinition.swagger.json",
  "icon": "icon.png",
  "script": "script.csx"
}
```

---

### Workflow B: Update Existing Connector

Use when modifying a connector that already exists in an environment.

```bash
# Step 1: Validate all changes
paconn validate -s settings.json

# Step 2: Update — include --secret only if connector uses OAuth
paconn update -s settings.json
paconn update -s settings.json --secret <oauth-client-secret>
```

Ensure `connectorId` is set in `settings.json` before running update.

---

### Workflow C: Download for Editing

Use when starting work on an existing connector or syncing local files from the platform.

```bash
# Step 1: Download connector files
paconn download --cid shared_myconnector-5fabc123 --env 00000000-0000-0000-0000-000000000000 --dest ./connector

# Step 2: Edit the downloaded files in ./connector/shared_myconnector-5fabc123/

# Step 3: Validate before pushing changes
paconn validate -s ./connector/shared_myconnector-5fabc123/settings.json

# Step 4: Push updated files
paconn update -s ./connector/shared_myconnector-5fabc123/settings.json
```

---

## Known Limitations

### stackOwner Blocks paconn update

When `apiProperties.json` includes a `stackOwner` field (required for ISV certification submission), `paconn update` fails. Maintain two versions of `apiProperties.json`:

| File | Purpose | Contains stackOwner |
|------|---------|---------------------|
| `apiProperties.json` | Development and paconn update | No |
| `apiProperties.certification.json` | Certification submission only | Yes |

Never use the certification version with `paconn update`.

### No Service Principal Authentication

`paconn login` only supports interactive device-code flow. For CI/CD pipelines and automated deployments, use the `pac` CLI (`pac connector create`, `pac connector update`) which supports Service Principal authentication.

### Environment Selection

If `--env` is omitted and no `environment` is set in `settings.json`, `paconn` prompts interactively and may default to a Power Automate environment rather than the intended target. Always set `environment` in `settings.json` or pass `-e [ENV-GUID]` explicitly.

---

## Finding the Environment GUID

1. Open the Power Platform admin center: https://admin.powerplatform.microsoft.com
2. Select **Environments** from the left navigation.
3. Click the target environment name.
4. Copy the **Environment ID** from the details panel.

The GUID format is `00000000-0000-0000-0000-000000000000`. Use this value for the `--env` flag and the `environment` field in `settings.json`.
