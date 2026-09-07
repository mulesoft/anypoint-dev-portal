---
name: apply-universal-policy
description: |
  Protect a portfolio slice by applying one Universal (canonical) policy across
  API instances on any gateway (Kong, Apigee, Azure, AWS, MuleSoft), or audit
  and edit already-applied native policies. Use when the user says all / many /
  my APIs / a provider name / a portfolio slice — JWT validation, rate limiting,
  spike arrest, IP filtering, "what protection do I have", tighten an existing
  policy, or apply one config across providers. DO NOT TRIGGER for a single
  already-chosen API Manager instance — use skill apply-policy-to-api-instance.
  Do not use to remove, detach, enable, or disable a policy (those tools do not
  exist).
license: Apache-2.0
compatibility: Requires the MuleSoft Platform MCP Server (urn:mcp:mulesoft-platform) with list_universal_policies, apply_universal_policy, get_policy_operation_status, and find_assets (include_applied_policies). If those tools are missing, stop.
metadata:
  author: mulesoft-omni
  version: "1.1.0"
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
- Apply one canonical policy (JWT, rate limiting, spike arrest, IP allow/deny)
  across many instances or many providers

**Trigger keywords:** all my APIs · portfolio · missing this policy · universal
policy · canonical policy · JWT validation · rate limiting · spike arrest ·
IP filtering · IP allowlist · Kong · Apigee · Azure · AWS.

**Do NOT use this skill when:**

- The user already has **one** API Manager instance in context and wants a
  single native policy on it → **skill apply-policy-to-api-instance**
- They need to **create** an instance or deploy a gateway first → **skill secure-api**
- They want to **remove**, detach, or pause (enable/disable) a policy — say so
  and stop; those actions are not on the tool surface

## Prerequisites

You must be connected to the MuleSoft Platform MCP Server
(`urn:mcp:mulesoft-platform`). Probe with a read-only call:

1. Call `list_universal_policies`. If the tool is **not in the catalog**, STOP
   and say the Platform MCP catalog on this host does not include the Universal
   policy tools yet. Do not substitute Anypoint REST calls.
2. If the tool returns the catalog is unavailable (`status=not_configured` or
   an empty `policies` list with that message), say so and stop — do not invent
   a fallback apply path.
3. On a 403 / entitlement / gate-closed error, say they are not entitled and
   **stop**. Never silently switch provider or org.

## Reference files

- **`references/presentation-format.md`** — pinned tables. Read it when you
  are about to render a user-facing list, schema, mapping, or result.
- **`references/payloads.md`** — how to join `find_assets` rows, branch on
  apply `status`, and call `edit_applied_policy`. Read it before the first
  apply or edit.

## Workflow

### Pick the path first

Read the latest user turn and choose **one** path. Do not run Protect steps
for an Audit or Edit request. Re-pick on every turn (a conversation can
Protect, then Audit, then Edit).

| Path | When | Go to |
| --- | --- | --- |
| **Protect** | A filter + a protection to apply ("JWT on finance APIs") | Steps 1–4 |
| **Audit** | "What protection / what's applied" with no change yet | Step 5 |
| **Edit** | Change an already-applied policy ("add 1.1.1.1") | Step 6 |

### Rules that always apply

1. **Confirm before acting.** Report found/missing (or the edit preview) and
   wait for an explicit yes before `apply_universal_policy` or
   `edit_applied_policy`. A missing confirmation writes policies the user did
   not approve.
2. **Instances, not APIs.** Users say "APIs"; policies attach to **deployed
   instances**. Gate apply on `policyCoverage` + `instancesMissingPolicy`,
   never on `hasPolicy` alone (`hasPolicy` is any-instance and would skip
   unprotected siblings).
3. **Join before you talk.** `instancesMissingPolicy` is `{instanceId,
   environment}` only. Build display rows and `instance_ids` using
   `references/payloads.md` — do not invent names or providers.
4. **Apply: one config, one shot.** Show the schema table (full surface, or
   required-only when it is too long — see `references/presentation-format.md`),
   then collect the **entire** configuration for the fields you showed in one
   reply. Reuse it across every targeted instance. Never walk field by field.
5. **Wait for completion.** Acceptance (`accepted` / HTTP 202 / edit
   `success`) is not success. Poll until `COMPLETED` or `FAILED`. Distinguish
   retryable vs terminal from `error.retryable`.
6. **Plain names.** Mappings use human-readable native policy names from
   `providerMapping`, not IDs.

### Step 1: Discover the slice and what's missing (Protect)

Call `find_assets` with:

- `query` — search terms from the request (names, "finance", "payments").
  Include tag-like words when the user said "tagged", but do not claim a
  tag filter the backend did not apply — if hits look unrelated, say you
  searched by those terms.
- `asset_type` — `api` when they mean APIs (expands to the API Exchange types)
- `user_query` — the verbatim user message
- `include_applied_policies` — `true`
- `policy_name_filter` — the protection they named ("JWT validation",
  "rate-limiting"; matching is case- and separator-insensitive)
- `max_results` — raise it when they asked for "all" / a large slice
  (default is 20)

If `userOrgAssets` is empty, say you found no matching API instances and
stop. Do not search the public catalog for apply targets.

Present the **API-instance list** (`references/presentation-format.md`)
using the join in `references/payloads.md`. Summarize **per instance**:
"N instances found, M missing this policy, K unknown."

Coverage rules:

| `policyCoverage` | Meaning | Apply to |
| --- | --- | --- |
| `all` | Every readable instance has it | skip (already protected) |
| `partial` / `none` | Some or none have it | exactly the joined `instancesMissingPolicy` |
| `unknown` | Incomplete read (error or cap) | re-check with `view_api_instance_policies`. Never count as missing. |

If `appliedPolicyEnrichment.assetCapApplied` is true, fan out
`view_api_instance_policies` for the remaining instances rather than
treating the truncated set as complete.

**[GATE] Wait for the user to confirm they want you to fix the gap.**

### Step 2: Choose the Universal policy and show its schema (Protect)

Call `list_universal_policies` with `policy_name_hint` from the user's
protection. If more than one template matches, list their `label`s and ask
which to use — do not pick silently.

Each entry has `configurationSchema`, `supportedProviders`, and
`providerMapping` (what it becomes on Kong / Apigee / …).

Drop any target instance whose provider is not in `supportedProviders`. If
that leaves none, say so and stop. Keep the surviving list — that is the
apply set for Steps 3–4, **not** the raw Step 1 missing list.

Present the **policy configuration schema** table
(`references/presentation-format.md`). If the flattened schema is too long
to scan, show required fields only, say how many optionals are hidden, and
offer to expand. Then ask the user to fill the **visible** fields in one
reply. Do not invent optional values; omitted hidden fields keep schema
defaults.

### Step 3: Preview the native mapping, then apply (Protect)

Build the **policy → API + instance mapping** by joining each **remaining**
instance's provider to `providerMapping`. Show it. **[GATE] Wait for okay.**

Call `apply_universal_policy` once with:

- `policy_name` — the canonical kebab-case template `name`
- `instance_ids` — remaining missing instances from Step 2 (after the
  provider drop), not the raw Step 1 list
- `configuration` — the collected config, reused as-is

Do **not** loop `apply_policy_to_instance` for a Universal/canonical template.
Use that tool only for a provider-native plugin on a single instance.

### Step 4: Wait until it is actually done (Protect)

Branch on the apply `status` (`references/payloads.md`):

- `accepted` — poll `get_policy_operation_status` with `operationId` until
  `COMPLETED` or `FAILED` (not `RUNNING`). A few seconds between polls;
  give up after ~20 attempts and report **Still running**.
- `ok` — synchronous success. Report immediately. Do not poll.
- `partial` — some targets failed. Report **Partially succeeded** from
  `results`. Do not poll.

Then the same **policy → API + instance mapping** as the preview (now as
the result). On `FAILED`, say whether `error.retryable` is true.

### Step 5: Audit an existing slice

Skip Steps 1–4. Do **not** pass `policy_name_filter`. Do **not** offer to
apply unless the user asks.

1. `find_assets` with `include_applied_policies=true`, `asset_type=api`,
   `user_query` verbatim. Put a named provider in `query`, then filter
   client-side (`references/payloads.md`).
2. Present the **applied-policy list** (grouped by API, then instance),
   only the requested provider's instances.
3. Ask whether they want to change anything. If they do, go to Step 6.

### Step 6: Edit an already-applied native policy

Skip Steps 1–4 unless you still need the instance list from Step 5.

`edit_applied_policy` is **Kong / Apigee / Azure only**. If the instance is
MuleSoft/Anypoint, say you cannot edit it with the current tools and stop.

There is no bulk-edit: loop **once per instance**.

1. Identify `api_instance_id` + `policy_id` + Exchange
   `asset.{group_id,asset_id,asset_version}` + `provider` from
   `view_api_instance_policies` (do not guess).
2. Load the schema with `get_policy_template_form` using those coordinates.
3. Read current `configurationData`. Merge the requested **delta** into that
   full object (`configuration_data` replaces wholesale — omitted keys are
   dropped). Prompt only for required fields the merge left empty — do not
   re-ask the whole schema when the delta is complete.
4. Show the **policy → API + instance mapping** of the edit (every instance
   you will loop). **[GATE] Wait for okay.**
5. Call `edit_applied_policy` per instance. A `success` / 202 is acceptance.
   Poll `list_instance_policy_operations` per instance until `COMPLETED` /
   `FAILED`. Confirm per API instance.

A `readOnly: true` policy cannot be edited — say so and stop.

## Best Practices

- **Route first.** ✅ Audit stays on Step 5. ❌ running Step 1's
  `policy_name_filter` + apply GATE because the file is numbered 1–6.
- **One apply call for Universal.** ✅ `apply_universal_policy` with the
  Step-2 remaining ids. ❌ looping native apply; ❌ sending the pre-drop
  Step 1 list.
- **Don't invent display fields.** ✅ join `instanceId` as in
  `references/payloads.md`. ❌ filling provider/name from memory.

## Troubleshooting

**Universal policy tools are missing from the catalog:** say so and stop.
Do not substitute Anypoint REST calls.

**`policyCoverage` is `unknown`, or the table shows `?`:** re-read those
instances with `view_api_instance_policies`. Do not apply as if they were
missing.

**403 / not entitled / gate closed:** the org cannot manage that provider's
policies. Say so plainly and stop.

**Apply returned `ok` or `partial` and you started polling:** those statuses
are already terminal. Report from `results`; only `accepted` has an
`operationId` to poll.

**Edit dropped fields the user did not mention:** `configuration_data` is a
full replace. Merge onto the latest `configurationData` immediately before
the edit.

**Edit failed on a MuleSoft instance:** expected — the tool is external-only.
Stop and say so.

**User asks to undo / remove / disable the policy:** not supported. There is
no detach tool and no enable/disable parameter on the agent surface. Explain
and stop.

## Related Skills

- **skill apply-policy-to-api-instance**: apply one catalog policy to a single
  already-chosen API Manager instance (Anypoint REST, not Universal fan-out).
- **skill secure-api**: create/deploy an instance and then apply a policy —
  use that when the API is not yet an instance.
- **skill secure-mcp-server**: protect an MCP server, not an API instance.
