---
name: datasource-provisioning
license: Apache-2.0
description: Generate a copy-paste Grafana data source provisioning config for any plugin by fetching that plugin's standardized settings schema from the Grafana plugins CDN. Given a data source type (e.g. "infinity", "yesoreyeram-infinity-datasource", "clickhouse", "github"), resolve its latest version, ASK whether the user wants a YAML config file or Terraform, map each schema field to jsonData/secureJsonData/root by its declared target, and emit a ready-to-use provisioning/datasources/*.yaml or *.tf. Use when the user asks to provision a data source, set up a datasource YAML, configure a plugin data source as code, generate Terraform for a data source, or says "provision <plugin>", "datasource yaml for <plugin>", "terraform for <plugin>", "configure <plugin> as code" — even when they only name the plugin and not the word "provisioning".
---

# Data Source Provisioning from Plugin Settings Schemas

Grafana data source plugins publish a standardized, machine-readable settings schema on the plugins CDN. That schema is the source of truth for every field and whether it's secret. This skill maps it into a provisioning file in the format the user wants — YAML or Terraform — instead of guessing field names from memory.

## Workflow

### 1. Resolve the full plugin id

Provisioning needs the canonical plugin id (`<org>-<name>-datasource`), not the short name a user might say.

- Already canonical (contains `-datasource` or `-app`)? Use as-is: `yesoreyeram-infinity-datasource`.
- Short name only (e.g. `infinity`, `clickhouse`)? Resolve via the catalog API:
  ```bash
  curl -s "https://grafana.com/api/plugins?typeCodes=datasource&keyword=infinity" \
    | jq -r '.items[] | "\(.slug)\t\(.name)"'
  # pick the slug whose name matches what the user meant → yesoreyeram-infinity-datasource
  ```
  Multiple matches → show the candidates and ask which one.

### 2. Resolve the latest version

```bash
curl -s "https://grafana.com/api/plugins/yesoreyeram-infinity-datasource" | jq -r '.version'
```
Never hardcode a version — the CDN path is version-pinned and a stale version 404s.

### 3. Ask the user: YAML or Terraform?

**Always ask before generating** — same fields, different output file and syntax:

| Choice | Produces |
|--------|----------|
| **YAML config file** | `provisioning/datasources/<name>.yaml` |
| **Terraform** | `<name>.tf` (`grafana_data_source` resource) |

Do not assume — a user who says "provision X" may want either. If they already named a format ("terraform for X"), skip the question.

### 4. Fetch the settings schema (primary structured source)

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
      "key": "auth_method",          // the provisioning key
      "valueType": "string",          // string | boolean | number
      "target": "jsonData",           // root | jsonData | secureJsonData
      "validations": [ { "type": "allowedValues", "values": ["none","basicAuth","apiKey","bearerToken","oauth2","aws","azureBlob"] } ]
    }
  ]
}
```

Select only the fields relevant to what the user asked for (chosen auth method + connection), not all of them. Honor `validations.allowedValues` for selector fields like `auth_method`. Each field's `description` tells you which auth method it belongs to.

### 5. Map each field by its `target`, in the chosen format's syntax

| `target` | YAML | Terraform (`grafana_data_source`) |
|----------|------|-----------------------------------|
| `root` | top-level key on the datasource (`url`, `basicAuth`, `basicAuthUser`) | top-level argument (`url`) / inside `json_data_encoded` |
| `jsonData` | under `jsonData:` | key inside `json_data_encoded = jsonencode({ … })` |
| `secureJsonData` | under `secureJsonData:` as `${ENV_VAR}` | key inside `secure_json_data_encoded = jsonencode({ … })` via a `sensitive` variable |

Use each field's `valueType` for the scalar (`string` quoted in YAML, `boolean`→`true`/`false`, `number` bare). Never inline a real secret. Nested objects (`oauth2`, `aws`) and arrays (`allowedHosts`, `scopes`) map directly.

### 6. Emit the file

**YAML** → `provisioning/datasources/<name>.yaml`:
```yaml
apiVersion: 1
datasources:
  - name: Infinity
    type: yesoreyeram-infinity-datasource   # = pluginType from the schema
    uid: infinity-ds                          # stable so dashboards can reference it
    jsonData:
      auth_method: apiKey                     # value from validations.allowedValues
      apiKeyKey: X-API-Key
      apiKeyType: header
      allowedHosts:
        - https://api.example.com
    secureJsonData:
      apiKeyValue: ${API_KEY}                 # env var ref, never a literal secret
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

### 7. Fallback when no schema is published

If `schema/settings.schema.json` 404s (older plugins):
- Try the `configuring-<PLUGIN_ID>/SKILL.md` prose skill (its provisioning + auth-method sections), then the CDN `README.md`, then the docs link from the catalog API.
- Last resort: the generic structure in [grafana-oss](../grafana-oss/SKILL.md) (§ Data source provisioning) — tell the user the field names are best-effort, not plugin-authoritative.

### 8. Return the file to the user

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

## Related

- [grafana-oss](../grafana-oss/SKILL.md) — generic data source / dashboard provisioning structure and provisioning paths.
