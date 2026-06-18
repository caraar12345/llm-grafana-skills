---
name: datasource-provisioning
license: Apache-2.0
description: Generate a copy-paste Grafana data source provisioning YAML for any plugin by fetching that plugin's standardized settings schema from the Grafana plugins CDN. Given a data source type (e.g. "infinity", "yesoreyeram-infinity-datasource", "clickhouse", "github"), resolve its latest version, fetch schema/settings.schema.json from plugins-cdn.grafana.net, map each field to jsonData/secureJsonData/root by its declared target, and emit a provisioning/datasources/*.yaml the user can drop into their environment. Use when the user asks to provision a data source, set up a datasource YAML, configure a plugin data source as code, generate provisioning config for a specific plugin, or says "provision <plugin>", "datasource yaml for <plugin>", "how do I configure <plugin> as code" — even when they only name the plugin and not the word "provisioning".
---

# Data Source Provisioning from Plugin Settings Schemas

Grafana data source plugins publish a standardized, machine-readable settings schema on the plugins CDN. This skill fetches that version-matched schema and maps it directly into a provisioning file — instead of guessing field names from memory.

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
  If the keyword returns multiple plugins, show the candidates and ask which one.

### 2. Resolve the latest version

```bash
curl -s "https://grafana.com/api/plugins/yesoreyeram-infinity-datasource" | jq -r '.version'
# → 3.9.0
```

Never hardcode a version — the CDN path is version-pinned and a stale version 404s.

### 3. Fetch the settings schema (primary, standardized source)

URL pattern:

```
https://plugins-cdn.grafana.net/<PLUGIN_ID>/<VERSION>/public/plugins/<PLUGIN_ID>/schema/settings.schema.json
```

Example:

```bash
ID=yesoreyeram-infinity-datasource
VER=$(curl -s "https://grafana.com/api/plugins/$ID" | jq -r '.version')
curl -sf "https://plugins-cdn.grafana.net/$ID/$VER/public/plugins/$ID/schema/settings.schema.json"
```

Schema shape (`schemaVersion: "v1"`):

```jsonc
{
  "pluginType": "yesoreyeram-infinity-datasource",
  "pluginName": "Infinity",
  "docURL": "https://grafana.com/docs/plugins/yesoreyeram-infinity-datasource/",
  "fields": [
    {
      "id": "jsonData.auth_method", // unique id
      "key": "auth_method", // YAML key under its target
      "label": "Authentication method",
      "description": "...",
      "valueType": "string", // string | boolean | number
      "semanticType": "url", // optional hint (url, etc.)
      "target": "jsonData", // root | jsonData | secureJsonData
      "validations": [
        // optional constraints
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

### 4. Map each field to a provisioning slot by its `target`

| `target`         | Goes in YAML at                                                             | Notes                                                         |
| ---------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `root`           | top-level on the datasource object (`url`, `basicAuth`, `basicAuthUser`, …) | non-secret connection settings                                |
| `jsonData`       | under `jsonData:`                                                           | non-secret config                                             |
| `secureJsonData` | under `secureJsonData:`                                                     | **secrets — emit as `${ENV_VAR}`, never inline a real value** |

- Use each field's `key` as the YAML key and `valueType` for the scalar type (`string` → quoted, `boolean` → `true`/`false`, `number` → bare).
- Honor `validations.allowedValues` — set selector fields (e.g. `auth_method`) to a value from that list.
- **Include only the fields relevant to what the user asked for** (chosen auth method + connection), not all 42 fields. A field's `description` tells you which auth method it belongs to.

### 5. (Optional) Enrich with the plugin's prose skill

Some plugins also ship a human-oriented skill at the standardized path `configuring-<PLUGIN_ID>`:

```
https://plugins-cdn.grafana.net/<PLUGIN_ID>/<VERSION>/public/plugins/<PLUGIN_ID>/skills/configuring-<PLUGIN_ID>/SKILL.md
```

Fetch it for worked examples and auth-method narrative. It may 404 (not all versions/plugins ship it) — that's fine, the schema in step 3 is authoritative on its own.

### 6. Emit the provisioning YAML

Write `provisioning/datasources/<name>.yaml`:

```yaml
apiVersion: 1
datasources:
  - name: <human-readable name>
    type: yesoreyeram-infinity-datasource # = pluginType from the schema
    uid: <stable-uid> # stable so dashboards can reference it
    access: proxy
    url: https://api.example.com # target: root field
    basicAuth: true # target: root field
    basicAuthUser: api-user # target: root field
    jsonData: # target: jsonData fields
      auth_method: basicAuth # value from allowedValues
    secureJsonData: # target: secureJsonData fields
      basicAuthPassword: ${INFINITY_PASSWORD} # env var ref, never a literal secret
    editable: false
```

### 7. Fallback when no schema is published

If `schema/settings.schema.json` 404s (older plugins):

- Try the prose skill (step 5), then the CDN `README.md` (`.../public/plugins/<PLUGIN_ID>/README.md`), then the `docURL`/docs link from the catalog API.
- Fall back to the generic structure in [grafana-oss](../grafana-oss/SKILL.md) (§ Data source provisioning) and tell the user the field names are best-effort, not plugin-authoritative.

### 8. Validate

Drop the file in the provisioning path, restart Grafana, then health-check:

```bash
curl -s https://grafana.example.com/api/datasources/uid/<uid>/health \
  -H "Authorization: Bearer <token>"
# { "status": "OK" }    → working
# { "status": "ERROR" } → URL unreachable or auth misconfigured
```

## Related

- [grafana-oss](../grafana-oss/SKILL.md) — generic data source / dashboard provisioning structure and provisioning paths.
