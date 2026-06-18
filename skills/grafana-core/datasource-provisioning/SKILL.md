---
name: datasource-provisioning
license: Apache-2.0
description: Generate a copy-paste Grafana data source provisioning file (YAML or Terraform) for any plugin from its standardized settings schema on the plugins CDN. Use when the user wants to provision or configure a data source as code — e.g. "provision infinity", "datasource yaml for clickhouse", "terraform for the github datasource" — even when they only name the plugin and not the word "provisioning".
---

## Workflow

### 1. Ask the starting point: from scratch, or from an existing data source?

**Ask this before anything else** (skip only if the user already made it clear):

- **From scratch** — the user names a plugin type to provision → continue with step 2.
- **From an existing data source** in a running instance → jump to [Convert an existing data source](#convert-an-existing-data-source), then return to step 6.

### 2. Resolve the full plugin id

Provisioning needs the canonical plugin id (`<org>-<name>-datasource`), not the short name a user might say.

- Already canonical (contains `-datasource` or `-app`)? Use as-is: `yesoreyeram-infinity-datasource`.
- Short name only (e.g. `infinity`, `clickhouse`)? Search the catalog API with `filter=<keyword>`:
  ```bash
  curl -s "https://grafana.com/api/plugins?filter=infinity" \
    | jq -r '.items[] | "\(.slug)\t\(.name)"'
  # → yesoreyeram-infinity-datasource    Infinity
  ```
  Multiple matches → show the candidates and ask which one.

### 3. Resolve the latest version

```bash
curl -s "https://grafana.com/api/plugins/yesoreyeram-infinity-datasource" | jq -r '.version'
```

Never hardcode a version — the CDN path is version-pinned and a stale version 404s.

### 4. Ask the user: YAML or Terraform?

**Always ask before generating** — same fields, different output file and syntax:

| Choice               | Produces                                     |
| -------------------- | -------------------------------------------- |
| **YAML config file** | `provisioning/datasources/<name>.yaml`       |
| **Terraform**        | `<name>.tf` (`grafana_data_source` resource) |

Do not assume — a user who says "provision X" may want either. If they already named a format ("terraform for X"), skip the question.

### 5. Fetch the settings schema (primary structured source)

```
https://plugins-cdn.grafana.net/<PLUGIN_ID>/<VERSION>/public/plugins/<PLUGIN_ID>/schema/settings.schema.json
```

```bash
ID=yesoreyeram-infinity-datasource
VER=$(curl -s "https://grafana.com/api/plugins/$ID" | jq -r '.version')
curl -sf "https://plugins-cdn.grafana.net/$ID/$VER/public/plugins/$ID/schema/settings.schema.json"
```

Schema shape (`schemaVersion: "v1"`) — each field declares where it goes and its constraints:

```jsonc
{
  "pluginType": "yesoreyeram-infinity-datasource",
  "fields": [
    {
      "key": "auth_method", // the provisioning key
      "valueType": "string", // string | boolean | number
      "target": "jsonData", // root | jsonData | secureJsonData
      "validations": [
        {
          "type": "allowedValues",
          "values": [
            "none",
            "basicAuth",
            "apiKey",
            "bearerToken",
            "oauth2",
            "aws",
            "azureBlob",
          ],
        },
      ],
    },
  ],
}
```

Select only the fields relevant to what the user asked for (chosen auth method + connection), not all of them. Honor `validations.allowedValues` for selector fields like `auth_method`. Each field's `description` tells you which auth method it belongs to.

### 6. Map each field by its `target`, in the chosen format's syntax

| `target`         | YAML                                                                  | Terraform (`grafana_data_source`)                                                    |
| ---------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `root`           | top-level key on the datasource (`url`, `basicAuth`, `basicAuthUser`) | top-level argument (`url`) / inside `json_data_encoded`                              |
| `jsonData`       | under `jsonData:`                                                     | key inside `json_data_encoded = jsonencode({ … })`                                   |
| `secureJsonData` | under `secureJsonData:` as `${ENV_VAR}`                               | key inside `secure_json_data_encoded = jsonencode({ … })` via a `sensitive` variable |

Use each field's `valueType` for the scalar (`string` quoted in YAML, `boolean`→`true`/`false`, `number` bare). Never inline a real secret. Nested objects (`oauth2`, `aws`) and arrays (`allowedHosts`, `scopes`) map directly.

### 7. Emit the file

**YAML** → `provisioning/datasources/<name>.yaml`:

```yaml
apiVersion: 1
datasources:
  - name: Infinity
    type: yesoreyeram-infinity-datasource # = pluginType from the schema
    uid: infinity-ds # stable so dashboards can reference it
    jsonData:
      auth_method: apiKey # value from validations.allowedValues
      apiKeyKey: X-API-Key
      apiKeyType: header
      allowedHosts:
        - https://api.example.com
    secureJsonData:
      apiKeyValue: ${API_KEY} # env var ref, never a literal secret
    editable: false
```

**Terraform** → `<name>.tf`:

```hcl
variable "api_key" {
  type      = string
  sensitive = true
}

resource "grafana_data_source" "infinity" {
  type = "yesoreyeram-infinity-datasource"
  name = "Infinity"
  uid  = "infinity-ds"

  json_data_encoded = jsonencode({
    auth_method  = "apiKey"
    apiKeyKey    = "X-API-Key"
    apiKeyType   = "header"
    allowedHosts = ["https://api.example.com"]
  })

  secure_json_data_encoded = jsonencode({
    apiKeyValue = var.api_key
  })
}
```

### 8. Fallback when no schema is published

If `schema/settings.schema.json` 404s (older plugins):

- Try the `configuring-<PLUGIN_ID>/SKILL.md` prose skill (its provisioning + auth-method sections), then the CDN `README.md`, then the docs link from the catalog API.
- Last resort: the generic structure in [grafana-oss](../grafana-oss/SKILL.md) (§ Data source provisioning) — tell the user the field names are best-effort, not plugin-authoritative.

### 9. Return the file to the user

Present the complete file in a single code block for the user to copy and paste into their environment — note where it goes:

- **YAML** → `provisioning/datasources/<name>.yaml` (apply on Grafana start or a provisioning reload).
- **Terraform** → their Terraform config, applied with `terraform apply`.

Optionally, tell them how to confirm it worked once applied:

```bash
curl -s https://grafana.example.com/api/datasources/uid/<uid>/health \
  -H "Authorization: Bearer <token>"
# { "status": "OK" }    → working
# { "status": "ERROR" } → URL unreachable or auth misconfigured
```

## Convert an existing data source

To codify a data source already configured in a running instance, read its config through the **Grafana MCP server** ([grafana/mcp-grafana](https://github.com/grafana/mcp-grafana)).

**Precondition:** the Grafana MCP server is connected with its **Datasources** toolset enabled (it holds the instance credentials). **If it isn't available, do not support this path** — never ask the user to paste a Grafana token into chat. Fall back to the from-scratch Workflow instead.

1. Find the data source with the MCP tools — `list_datasources` to browse, then `get_datasource` (by `uid` or `name`) for the full config.
2. The result carries every **non-secret** field directly: `type`, `uid`, `url`, `access`, `basicAuth`, `basicAuthUser`, and the full `jsonData` object. Copy them as-is.
3. **Secrets are never returned.** The `secureJsonFields` map lists _which_ secret keys are set (e.g. `{"apiKeyValue": true}`) without their values. Emit an `${ENV_VAR}` placeholder for each key it reports `true`.
4. Cross-check against the schema (step 5) to confirm secret key names and `target` placement, then continue at **step 6** (map) and **step 7** (emit) as normal.

## Related

- [grafana-oss](../grafana-oss/SKILL.md) — generic data source / dashboard provisioning structure and provisioning paths.
