# Tool payloads — apply-universal-policy

Read this when building tables, choosing `instance_ids`, or calling
`edit_applied_policy`. Field names below are the structured tool output,
not the chat rendering.

## `find_assets` join (Protect + Audit)

`include_applied_policies=true` annotates each **userOrgAssets** row. Public
catalog hits have no applied-policy state — ignore them for this skill.

Per user-org asset you get:

- API display name from the row (`name` / title).
- `appliedPolicies.instances[]` — `{instanceId, environment, policyNames}`
  for the **sampled** instances.
- When `policy_name_filter` was set: `policyCoverage`, `hasPolicy`, and
  `instancesMissingPolicy[]` — `{instanceId, environment}` only.

`instancesMissingPolicy` does **not** carry instance name or provider.
Build each table row by:

1. Taking the API name from the parent asset.
2. Joining `instanceId` to `appliedPolicies.instances` for `environment`
   (and `policyNames` on the audit path).
3. Taking provider from the asset/version if present. If it is missing,
   call `view_api_instance_policies` for that `instanceId` before applying
   or before filtering to "my Apigee APIs". Do not invent a provider.
4. Using `environment` as the instance-name fallback when the payload has
   no separate instance name.

Apply targets = the joined `instancesMissingPolicy` rows whose
`policyCoverage` is `partial` or `none`. Skip `all`. Re-check `unknown`.

`appliedPolicyEnrichment.assetCapApplied` (or `assetsDropped` > 0) means
the list is truncated. Fan out `view_api_instance_policies` for remaining
user-org assets rather than treating the truncated set as complete.

## Provider slice ("my Apigee APIs")

`find_assets` has no provider parameter. Put the provider word in `query`
(and keep `user_query` verbatim), then **filter client-side**: keep
instances whose provider matches (case-insensitive: `apigee`, `kong`,
`azure`, `aws`, `mulesoft` / `anypoint`). If provider is still unknown
after the join, resolve it with `view_api_instance_policies` before
including or excluding the row.

## `apply_universal_policy` responses

Branch on `status` — do not assume async:

| `status` | Meaning | Next |
| --- | --- | --- |
| `accepted` | HTTP 202. Work is in flight. `operationId` is set. | Poll `get_policy_operation_status` |
| `ok` | Synchronous success. `appliedCount` / `results` are final. | Report **Succeeded** + mapping. Do not poll. |
| `partial` | HTTP 207. Some targets failed. | Report **Partially succeeded**; name failures from `results`. Do not poll. |

Poll only the `accepted` branch. Same `operationId` covers the whole fan-out
(not one id per instance). Poll every few seconds; stop at `COMPLETED` or
`FAILED`, or after ~20 attempts — then report **Still running**, never
success. On `FAILED`, read `error.retryable`.

`not_found` from the status tool means the id is unknown or aged out — say
so; do not retry with a guessed id.

## `edit_applied_policy` (external only)

This tool edits native policies on **Kong / Apigee / Azure** only. It
rejects MuleSoft/Anypoint instances. If the target is MuleSoft, say you
cannot edit it with the current tools and stop.

Required on every call (do not guess):

- `organization_id`
- `api_instance_id`
- `provider` — `kong` / `apigee` / `azure`
- `policy_id` — from `view_api_instance_policies`
- `asset.group_id`, `asset.asset_id`, `asset.asset_version` — Exchange
  coordinates of the applied template, from the same read
- `configuration_data` — **full** object. Omitted keys are dropped.

There is no bulk-edit. One user request across many instances = one
`edit_applied_policy` per instance, same merged config.

A `success` / 202 from the edit is **acceptance**, not completion. Poll
`list_instance_policy_operations` per instance (`organization_id`,
`api_instance_id`, `provider`) until that instance's latest update is
`COMPLETED` or `FAILED`.
