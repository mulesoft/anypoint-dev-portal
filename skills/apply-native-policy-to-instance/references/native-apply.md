# Native single-instance apply — routing

## Which tool path

| Instance | `provider` | Catalog / apply | Wait for completion |
| --- | --- | --- | --- |
| Kong / Apigee / Azure | `kong` / `apigee` / `azure` | Universal API via `prepare_policy_creation` → `get_policy_template_form` → `apply_policy_to_instance` (`asset` coordinates required) | If `httpStatus` is 202 or `operationStatus=accepted`, poll `list_instance_policy_operations` (`organization_id`, `api_instance_id`, `provider`) until that instance's latest APPLY is `COMPLETED` or `FAILED`. ~20 polls, a few seconds apart. A 202 is acceptance, not success. |
| MuleSoft / Anypoint | omit, or `mulesoft` | API Manager via the same three tools (`environment_id` required) | 2xx without `operationStatus` is terminal. Do not poll `list_instance_policy_operations` (external-only). |

AWS is not on this apply surface — say so and stop.

## `apply_policy_to_instance` vs `apply_universal_policy`

- Native plugin / template id on **one** instance → `apply_policy_to_instance`.
- Canonical kebab name from `list_universal_policies` on **many** instances
  → skill apply-universal-policy / `apply_universal_policy`.

Never send a Kong plugin name to `apply_universal_policy`. Never loop
`apply_policy_to_instance` to fake a Universal fan-out.

## Required arguments (do not guess)

External: `organization_id`, `api_instance_id`, `provider`,
`asset.group_id`, `asset.asset_id`, `asset.asset_version`,
`configuration_data`.

MuleSoft: `organization_id`, `environment_id`, `api_instance_id`,
`configuration_data`, plus template identity (`asset` or
`policy_template_id`) from `get_policy_template_form`.
