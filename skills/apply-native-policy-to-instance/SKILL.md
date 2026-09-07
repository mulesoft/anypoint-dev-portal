---
name: apply-native-policy-to-instance
description: |
  Apply one provider-native policy to a SINGLE already-chosen API instance
  (Kong plugin, Apigee template, Azure policy, or one MuleSoft/Anypoint
  instance) via the MuleSoft Platform MCP Server. Use when the user has one
  instance in context and wants a native plugin — "add ip-restriction on this
  Kong service", "apply that Apigee quota to prod", "rate limiting on this API
  Manager instance". DO NOT TRIGGER for a Universal/canonical policy across
  many instances or providers — use skill apply-universal-policy. DO NOT
  TRIGGER to remove, detach, enable, or disable a policy.
license: Apache-2.0
compatibility: Requires the MuleSoft Platform MCP Server (urn:mcp:mulesoft-platform) with prepare_policy_creation, get_policy_template_form, apply_policy_to_instance. If those tools are missing, stop.
metadata:
  author: mulesoft-omni
  version: "1.0.0"
---

# Apply Native Policy to an Instance

Apply one native policy to one instance: pick the instance, load the
provider catalog, collect configuration once, apply, and wait until it
has actually finished.

## When to Use This Skill

**Use this skill when the user asks to:**

- "Add ip-restriction to this Kong instance"
- "Apply quota to that Apigee proxy"
- "Put JWT validation on this API Manager instance"
- Apply a **native** plugin/template to **one** already-identified instance

**Trigger keywords:** this instance · this Kong service · this Apigee
proxy · native policy · Kong plugin · apply to this API.

**Do NOT use this skill when:**

- They want the **same canonical policy on many instances / providers**
  → **skill apply-universal-policy** (`apply_universal_policy`)
- They want to **audit or edit** an already-applied policy → **skill apply-universal-policy**
  (Audit / Edit paths)
- They are driving **Anypoint REST** (OpenAPI `urn:api:*`), not MCP →
  **skill apply-policy-to-api-instance**
- They need to **create** the instance first → **skill secure-api**
- They want to **remove / disable** a policy — say so and stop

## Prerequisites

Connected to `urn:mcp:mulesoft-platform`. Probe with
`prepare_policy_creation` for the target instance. If the tool is missing,
stop. On 403 / gate-closed (`enabled: false`), say they are not entitled
and stop — never silently switch provider.

## Reference files

- **`references/native-apply.md`** — provider routing, required arguments,
  and how to wait for completion. Read it before the apply call.

## Workflow

### Rules that always apply

1. **One instance.** If more than one instance matches, ask which one.
   Do not silently apply to a set — that is skill apply-universal-policy.
2. **Native catalog, not Universal.** `prepare_policy_creation` lists
   plugins/templates for **this** provider. Never send a Kong plugin name
   to `apply_universal_policy`.
3. **Confirm before acting.** Show instance + native policy name + schema,
   then wait for an explicit yes.
4. **One-shot config.** Show the schema table (required-only when it is
   longer than about 12 flattened rows, disclosing hidden optionals and
   offering to expand — same rule as skill apply-universal-policy). Collect
   the visible fields in one reply. Do not invent optionals.
5. **Wait for completion.** External apply `httpStatus` 202 /
   `operationStatus=accepted` is not success. Poll
   `list_instance_policy_operations`. MuleSoft apply is typically
   synchronous (2xx without `operationStatus`).

### Step 1: Identify the instance

If the user already named an instance, use that `api_instance_id` (and
`environment_id` for MuleSoft). Otherwise `find_assets` (`asset_type=api`,
`user_query` verbatim) and pick **one**. If several remain, list them
(API name · instance · environment · provider) and wait.

Resolve `provider`: `kong` / `apigee` / `azure` for external; omit (or
`mulesoft`) for Anypoint. Do not guess.

### Step 2: List native policies that are not yet applied

Call `prepare_policy_creation` with `organization_id`, `api_instance_id`,
`provider` (external), and `policy_name_hint` when they named a policy.
MuleSoft also needs `environment_id`.

If more than one template matches the hint, list `name`s and ask. If
none match, say so and stop.

### Step 3: Load the schema and collect configuration

Call `get_policy_template_form` with the selected template's Exchange
coordinates (`group_id`, `asset_id`, `asset_version` — required on the
external path) plus `provider` / `environment_id` as in Step 2.

Present the schema table. **[GATE] Wait for the whole config in one
reply, then for okay to apply.**

### Step 4: Apply once

Call `apply_policy_to_instance` **once** with:

- `api_instance_id`
- `configuration_data` — the collected object
- `asset.{group_id,asset_id,asset_version}` from Step 2 (required external)
- `provider` for Kong / Apigee / Azure
- `environment_id` for MuleSoft
- `injection_point` only if the template requires it and the user set it

Do not loop this tool. Do not call `apply_universal_policy`.

### Step 5: Wait until it is actually done

See `references/native-apply.md`. Report the native policy name → this
API instance (environment / provider). On failure, say whether it is
retryable when `error.retryable` is present.

## Troubleshooting

**Tools missing:** the Platform MCP catalog on this host is stale. Stop.
Do not fall back to Anypoint REST unless the user is on
skill apply-policy-to-api-instance.

**Gate closed / 403:** say they are not entitled for that provider. Stop.

**More than one instance:** ask. Do not fan out.

**User named a Universal/canonical template for many APIs:** switch to
skill apply-universal-policy.

## Related Skills

- **skill apply-universal-policy**: one canonical policy across many
  instances or providers; also audit/edit of already-applied policies.
- **skill apply-policy-to-api-instance**: same job over Anypoint REST
  (`urn:api:*`), not MCP.
- **skill secure-api**: create/deploy the instance first.
