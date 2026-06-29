# Component Registry

Auto-maintained registry of components.

The flow generator's assembler checks this registry **before** building any component (reuse-first):
an exact match is reused (and its usage counters bump); otherwise the component is created, stored
here, and committed — so the next flow that needs the same thing reuses it.

> This file is written automatically by the assembler. You can edit it freely; it is only
> created if it does not already exist.

## Repository layout

```
registry/
  index.json                                 # fast lookup catalog — metadata + counters, NO code
  components/
    cap.kinto.api_external_lead.7924cf.json   # full record incl. the component source code
    age.agenttoolcallingagent.8c27e9.json
    ski.skilltoolcallingagent.362826.json
    orc.orchestratortranscript_e.6f8069.json
```

**Two layers, on purpose:**
- `index.json` is read on *every* generation to match components quickly, so it omits code to stay small.
- The full source lives only in `components/<id>.json`, fetched **only** when an exact match is reused.

## Component id scheme

`<layer3>.<identity>.<sig6>` where `sig6` is the first 6 chars of the signature (guarantees uniqueness):

| Layer | Pattern | Example |
|---|---|---|
| Orchestrator | `orc.<type>.<sig6>` | `orc.orchestratortranscript_e.6f8069` |
| Agent | `age.<type>.<sig6>` | `age.agenttoolcallingagent.8c27e9` |
| Skill | `ski.<type>.<sig6>` | `ski.skilltoolcallingagent.362826` |
| Capability | `cap.<system>.<operation>.<sig6>` | `cap.kinto.api_external_lead.7924cf` |

## Component record fields

| Field | Meaning |
|---|---|
| `id` | stable unique id (see scheme above) |
| `layer` | `orchestrator` \| `agent` \| `skill` \| `capability` |
| `kind` | `rest` \| `guard` \| `audit` \| `smart_fhir` \| `mtls` \| `edi_parse` \| `synthetic` \| `llm` |
| `system` | external system name (capabilities only) |
| `display_name` | human-readable name |
| `signature` | sha256[:16] of the identity fields — drives matching |
| `config` | connection config (base_url / path / method / auth_type / body_type) |
| `input_schema` | request payload field shape |
| `component_type` | shell type for agents/skills (e.g. `ToolCallingAgent`) |
| `code` | full component source — **only in `components/<id>.json`, never in the index** |
| **Tracking** | |
| `created_at`, `created_by_flow` | provenance |
| `version` | record version |
| `reused` | bool — has it ever been reused |
| `reuse_count` | how many times reused |
| `used_in_flows` | flow ids that use it |
| `last_used_at` | most recent use timestamp |
| `instances` | per-flow configs applied to a shared shell (role, system_prompt, tools) |

## Matching (how "the same component" is decided)

A `signature` is computed from the identity fields and compared:

- **Capability** — `layer + kind + system + method + path + auth + body`. Same API operation = exact reuse.
- **Agent / skill / orchestrator** — `layer + component_type`. The shell is the reusable unit; the
  **prompt, role, and tools are per-flow configuration**, recorded under `instances[]`, not part of identity.
  So a "Lead Processor" and a "Refund Processor" reuse the **same** agent shell with different prompts.

Match outcomes: **exact** → reuse automatically; **close** (same system / same layer) → surfaced as
candidates; **none** → create + register.

## Identity model: capabilities vs agents/skills

- **Capabilities** are genuinely distinct components (a Kinto `/lead` connector ≠ a SAP `/so` connector —
  different code/config). On an exact match the stored `code` is pulled from here and built into the new
  flow. The registry **is** the source of the component.
- **Agents / skills / orchestrators** share an identical shell. Reuse bumps counters and appends the
  per-flow prompt/role to `instances[]`; the node itself is built from the shell and re-prompted.

## Configuration (on the assembler node)

Set these fields on the `cap.coloki.flow.assemble` component (or as env vars):

| Node field | Env var | Value |
|---|---|---|
| Registry GitHub Repo (owner/name) | `REGISTRY_GITHUB_REPO` | `owner/name` (a full URL also works) |
| Registry GitHub Token | `REGISTRY_GITHUB_TOKEN` | GitHub token with **Contents: Read and write** |
| Registry Branch | `REGISTRY_GITHUB_BRANCH` | branch to read/commit (e.g. `main`) |

Leave the repo/token blank to disable the registry (flows still generate normally).

## Notes

- Writes use the GitHub Contents API (no git binary needed); the index is written **once per flow**
  (batched) to minimize API calls.
- Concurrent generations are last-write-wins on `index.json`.
- The token must have write access to this repo, or registrations fail with HTTP 403.
