# Presentation formats — apply-universal-policy

Users say "APIs" when they mean **deployed instances**. Accept that phrasing
and resolve API → instances yourself, but every table titles and lists
**instances**. A protected instance must not hide an unprotected sibling of
the same API.

Reuse the **same** shape for the same kind of data everywhere it appears
(pre-apply preview and post-apply confirmation use the identical mapping
layout).

## 1. API-instance list (discovery)

Table titled **"API instances"**.

| API name | Instance name | Environment | Provider | Policy present? |
| --- | --- | --- | --- | --- |
| Payments API | prod | Production | apigee | ✗ |
| Payments API | sandbox | Sandbox | kong | ✓ |

Plus a summary counted **per instance**: "N instances found, M missing this
policy."

Fallback for small results: a bullet list that still shows API name +
instance name.

## 2. Policy configuration schema

Present the template's full `configurationSchema` (required **and** optional)
as one table, then ask for **the whole configuration in one shot** — never
field by field.

| Field | Description | Required? | Default / allowed values |
| --- | --- | --- | --- |
| `jwksUrl` | JWKS endpoint | yes | — |
| `skipClientIdValidation` | Skip client-id check | no | `false` |

For nested objects/arrays, indent the child rows under the parent field
(or flatten with dotted paths). Do not drop optional fields to "simplify".

## 3. Policy → API + instance mapping

Used for both the pre-apply confirmation and the post-apply result.

| Native policy | API name | Instance name | Environment | Provider |
| --- | --- | --- | --- | --- |
| Spike arrest | Payments API | prod | Production | apigee |
| Rate limiting | Orders API | prod | Production | kong |

Plain native names from `list_universal_policies` → `providerMapping` (not
IDs). Join each target instance's provider to that mapping.

## 4. Applied-policy list (audit)

Group by **API, then instance**. Under each instance list policy names (and
direction / enabled state when the read returns them).

```
Payments API
  prod (Production · apigee)
    - Spike arrest (inbound, enabled)
  sandbox (Sandbox · apigee)
    - (none)
```

## 5. Operation result

One short status line:

- **Succeeded** — every targeted instance completed.
- **Partially succeeded** — name the API instances that failed.
- **Failed** — retryable vs terminal, using the operation `error.retryable`
  flag when present. Never claim success while status is `RUNNING`.
