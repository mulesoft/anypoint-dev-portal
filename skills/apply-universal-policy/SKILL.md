---
name: apply-universal-policy
description: |
  Protect a portfolio slice by applying one Universal (canonical) policy across
  API instances on any gateway (Kong, Apigee, Azure, AWS, MuleSoft), or audit
  and edit already-applied native policies. Use when the user wants JWT
  validation on tagged APIs, rate limiting everywhere, IP filtering on Apigee,
  "what protection do I have", tighten an existing policy, or apply one config
  across providers. DO NOT TRIGGER when applying a policy to a single already
  chosen API Manager instance — use skill apply-policy-to-api-instance. Do not
  use to remove, detach, enable, or disable a policy (those tools do not exist).
license: Apache-2.0
compatibility: >
  Requires the MuleSoft Platform MCP Server (urn:mcp:mulesoft-platform). The
  Universal-policy tools this skill calls land in the portal catalog via
  mulesoft-dx PR #222 — if they are missing, stop (see Prerequisites).
metadata:
  author: mulesoft-omni
  version: "1.0.0"
---

# Apply Universal Policy

Protect APIs across gateways with one Universal policy: discover what is
missing, collect configuration once, apply in one call, and wait until the
work has actually finished.

## When to Use This Skill

**Use this skill when the user asks to:**

- "Ensure all my finance-tagged APIs have JWT validation"
- "What protection do I have on my Apigee APIs?"
- "Add 1.1.1.1 to the IP filtering on those APIs"
- Apply one canonical policy (JWT, rate limiting, CORS, IP allow/deny, …)
  across many instances or many providers

**Trigger keywords:** universal policy · canonical policy · JWT validation ·
rate limiting · spike arrest · IP filtering · IP allowlist · protect my APIs ·
portfolio · missing this policy · Kong · Apigee · Azure · AWS.

**Do NOT use this skill when:**

- The user already has **one** API Manager instance and wants a single native
  policy on it → **skill apply-policy-to-api-instance**
- They need to **create** an instance or deploy a gateway first → **skill secure-api**
- They want to **remove**, detach, or pause (enable/disable) a policy — say so
  and stop; those actions are not on the tool surface

## Prerequisites

You must be connected to the MuleSoft Platform MCP Server
(`urn:mcp:mulesoft-platform`). Probe availability with a read-only call:

1. Call `list_universal_policies`. If the tool is **not in the catalog**, STOP.
   Tell the user the portal catalog does not yet include these tools — they
   ship in [mulesoft-dx PR #222](https://github.com/mulesoft/mulesoft-dx/pull/222)
   and this skill cannot execute until that PR is merged and published.
2. If the tool returns that the Universal catalog is unavailable, say so and
   stop — do not invent a fallback apply path.
3. The caller needs policy-manage permission on the target org. On a 403 /
   entitlement error, say they are not entitled and **stop**. Never silently
   switch provider or org.

## Reference files

- **`references/presentation-format.md`** — pinned tables for every output
  (API-instance list, config schema, policy→API+instance mapping, applied-policy
  list, operation result). Read it before the first user-facing table and reuse
  those layouts for the rest of the run.

## Workflow

### Rules that always apply

1. **Confirm before acting.** Report found/missing (or the edit preview) and
   wait for an explicit yes before `apply_universal_policy` or
   `edit_applied_policy`. A missing confirmation is the most expensive mistake —
   it writes policies the user did not approve.
2. **Instances, not APIs.** Users say "APIs"; policies attach to **deployed
   instances**. Resolve that yourself. Gate apply on `policyCoverage` +
   `instancesMissingPolicy`, never on `hasPolicy` alone (`hasPolicy` is
   any-instance and would skip unprotected siblings).
3. **One config, whole surface.** Show every field the template schema exposes
   (required and optional), then collect the **entire** configuration in one
   shot. Reuse it across every targeted instance. Never walk field by field.
4. **Wait for completion.** A 202/`status=accepted` is not success. Poll
   `get_policy_operation_status` (or `list_instance_policy_operations` for a
   native edit) until `COMPLETED` or `FAILED`. Distinguish retryable vs
   terminal from `error.retryable`.
5. **Plain names.** Pre-apply and post-apply mappings use human-readable
   native policy names from `providerMapping`, not IDs.

### Step 1: Discover the slice and what's missing

Call `find_assets` with:

- `query` — the user's filter ("finance", "payments", a tag, a name)
- `asset_type` — `rest-api` when they mean APIs
- `user_query` — the verbatim user message
- `include_applied_policies` — `true`
- `policy_name_filter` — the protection they named ("JWT validation",
  "rate-limiting"; matching is case- and separator-insensitive)

Present the result with the **API-instance list** in
`references/presentation-format.md`. Summarize **per instance**:
"N instances found, M missing this policy."

Coverage rules:

| `policyCoverage` | Meaning | Apply to |
| --- | --- | --- |
| `all` | Every readable instance has it | skip (already protected) |
| `partial` / `none` | Some or none have it | exactly `instancesMissingPolicy` |
| `unknown` | Incomplete read (error or cap) | re-check those instances with `view_api_instance_policies` before acting. Never count `unknown` as missing. |

Enrichment is capped (about 12 assets, 3 production-first instances each). If
`appliedPolicyEnrichment.assetCapApplied` is true, fan out
`view_api_instance_policies` for the remaining instances rather than treating
the truncated set as complete.

**[GATE] Wait for the user to confirm they want you to fix the gap.**

### Step 2: Choose the Universal policy and show its schema

Call `list_universal_policies` (optional `policy_name_hint`). Pick the
canonical template that matches the user's protection. Each entry has
`configurationSchema`, `supportedProviders`, and `providerMapping` (what it
becomes on Kong / Apigee / …).

Drop any target instance whose provider is not in `supportedProviders`. If
that leaves none, say so and stop.

Present the **policy configuration schema** table (full surface). Then ask
the user to fill **all** fields in one reply.

### Step 3: Preview the native mapping, then apply

Build the **policy → API + instance mapping** by joining each remaining
instance's provider to `providerMapping`. Show it. **[GATE] Wait for okay.**

Call `apply_universal_policy` once with:

- `policy_name` — the canonical kebab-case template name
- `instance_ids` — every instance still missing the policy (from Step 1)
- `configuration` — the collected config, reused as-is

Do **not** loop `apply_policy_to_instance` for a Universal/canonical template
across many instances or providers — that is what `apply_universal_policy` is
for. Use `apply_policy_to_instance` only for a provider-native plugin on a
single instance.

### Step 4: Poll until it is actually done

If the apply returns `status=accepted` + `operationId`, poll
`get_policy_operation_status` until `COMPLETED` or `FAILED` (not `RUNNING`).

Report with the **operation result** layout, then the same **policy → API +
instance mapping** as the preview (now as the result). On partial failure,
name the instances that failed. On `FAILED`, say whether `error.retryable`
is true.

### Step 5: Audit an existing slice (optional path)

When the user asks "what protection do I have on my \<provider\> APIs?":

1. `find_assets` with `include_applied_policies=true` (no name filter needed).
2. Present the **applied-policy list** (grouped by API, then instance).
3. Ask whether they want to change anything.

### Step 6: Edit an already-applied native policy

When they ask to change an existing policy (e.g. add `1.1.1.1` to IP
filtering):

1. Identify the instance + `policy_id` + Exchange coordinates from
   `view_api_instance_policies` (do not guess IDs).
2. Load the schema with `get_policy_template_form` using those coordinates.
3. Read current `configurationData`, merge the requested delta into the
   **full** object (`edit_applied_policy` replaces wholesale — omitted keys
   are dropped).
4. Show the schema (only extra fields if the delta needs them) and the
   **policy → API + instance mapping** of the edit. **[GATE] Wait for okay.**
5. Call `edit_applied_policy`. Poll `list_instance_policy_operations` until
   `COMPLETED` / `FAILED`. Confirm per API instance.

A `readOnly: true` policy cannot be edited — say so and stop.

## Best Practices

- **One apply call for Universal.** ✅ `apply_universal_policy` with every
  missing `instance_id`. ❌ looping native apply for a canonical template.
- **Don't trust `hasPolicy`.** ✅ `policyCoverage` + `instancesMissingPolicy`.
  ❌ treating a True `hasPolicy` as "this API is done".
- **Don't claim success early.** ✅ poll to `COMPLETED`. ❌ reporting the 202
  as done.
- **Don't invent config.** ✅ full schema table, then one user-provided
  object. ❌ filling optional fields with guessed values.

## Troubleshooting

**Tool `list_universal_policies` / `apply_universal_policy` / `find_assets`
with `include_applied_policies` is missing:** the portal catalog is stale.
These tools are published by [PR #222](https://github.com/mulesoft/mulesoft-dx/pull/222).
Stop and tell the user; do not substitute Anypoint REST calls.

**`policyCoverage` is `unknown`:** an instance read failed or sampling was
capped. Re-read those instances with `view_api_instance_policies`. Do not
apply as if they were missing.

**403 / not entitled / gate closed:** the org cannot manage that provider's
policies. Say so plainly and stop.

**Apply returns 202 / `accepted`:** poll `get_policy_operation_status`. The
fan-out is one Temporal operation id for the whole apply, not per instance.

**Edit dropped fields the user did not mention:** `configuration_data` is a
full replace. Merge onto the latest `configurationData` from
`view_api_instance_policies` immediately before the edit.

**User asks to undo / remove / disable the policy:** not supported. There is
no detach tool and no enable/disable parameter on the agent surface. Explain
and stop.

## Related Skills

- **skill apply-policy-to-api-instance**: apply one catalog policy to a single
  already-chosen API Manager instance (Anypoint REST, not Universal fan-out).
- **skill secure-api**: create/deploy an instance and then apply a policy —
  use that when the API is not yet an instance.
- **skill secure-mcp-server**: protect an MCP server, not an API instance.
