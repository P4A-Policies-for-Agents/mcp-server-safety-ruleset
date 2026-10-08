# MCP Server Safety & MCP Server Usability rulesets (design)

Date: 2026-10-08. Status: draft, awaiting review.

## Goal

Ship two MuleSoft API Governance rulesets (AMF Validation Profile 1.0, `specKind: mcp`) as public
P4A catalog entries. Organizations deploy them from P4A into their own Anypoint orgs and apply
them through governance profiles to the MCP server assets they publish to Exchange.

- **MCP Server Safety** (`mcp-server-safety-ruleset`): an MCP server declares how it authenticates,
  what side effects its tools have, and that tool inputs are closed.
- **MCP Server Usability** (`mcp-server-usability-ruleset`): tools, resources and prompts are
  described well enough for an agent to pick and call them correctly.

The two are split so an org can adopt them independently. Each one complements MuleSoft's
**Agent Network Best Practices** (`68ef9520-24e9-4cf2-b2f5-620025690913/agent-network-best-practices`),
which covers agent cards, brokers, connections, MCP transport and protocol version. We don't
duplicate those rules, and both READMEs recommend applying MuleSoft's ruleset alongside ours.

**Out of scope:**
- Publishing to Exchange ourselves. P4A does that for adopters.
- Runtime enforcement. That belongs to gateway policies.
- Per-parameter schema rules. See Limitations.

## Target document

Each ruleset validates the Exchange MCP asset manifest, `mcp-metadata.json` (`classifier: mcp-metadata`).
The manifest contains `protocolVersion`, `transport`, `capabilities`, `securitySchemes`,
`tools[]` (`name`, `title`, `description`, `annotations`, `inputSchema`, `outputSchema`), `resources[]`
and `prompts[]` (with `arguments[]`).

## Rule catalog

Severity tiers:
- `violation`: unsafe, or breaks agent use.
- `warning`: strong recommendation.
- `info`: nice to have.

All paths use the canonical `core.*` prefix, and every profile declares
`prefixes: { mcp: http://anypoint.com/vocabs/mcp# }`.

### MCP Server Safety (Exchange asset `mcp-server-safety`, 1.0.0)

| Rule ID | Target | Constraint | Severity |
|---|---|---|---|
| `tool-side-effect-hint-declared` | `mcp.Tool` | `core.annotations` minCount 1, nested `or(core.readOnlyHint minCount 1, core.destructiveHint minCount 1)` | violation |
| `server-auth-declared` | `mcp.Server` | `core.securitySchemes` minCount 1 | violation |
| `tool-input-closed` | `mcp.ToolInputSchema` | `core.additionalProperties` minCount 1 + `in: [false]` | warning |
| `tool-open-world-hint-declared` | `mcp.Tool` | `core.annotations` nested `core.openWorldHint` minCount 1 | warning |

`tool-input-closed` triggers one known `validate-authoring` **warning**, because `in` is used on a node-typed property. It isn't an error. The spike showed this is the only way to tell `false` apart from `true` or a missing value, and the README documents it.

### MCP Server Usability (Exchange asset `mcp-server-usability`, 1.0.0)

| Rule ID | Target | Constraint | Severity |
|---|---|---|---|
| `tool-description-required` | `mcp.Tool` | `core.description` minCount 1 | violation |
| `tool-name-format` | `mcp.Tool` | `core.name` pattern `^[a-z][a-z0-9_]{0,63}$` | warning |
| `tool-description-substantive` | `mcp.Tool` | `core.description` minLength 20 | warning |
| `resource-described` | `mcp.Resource` | `core.description` and `core.mimeType` minCount 1 | warning |
| `prompt-described` | `mcp.Prompt` | `core.description` minCount 1 | warning |
| `prompt-argument-described` | `mcp.Prompt` | `core.arguments` nested `core.description` minCount 1 | warning |
| `tool-output-schema-declared` | `mcp.Tool` | `core.outputSchema` minCount 1 | info |

### Every rule carries

- `message`: one line that names the problem and the fix. Example: "Tool has no description. Add one that tells an agent when to call this tool."
- `documentation`: why it matters for agents and gateways.
- `examples.valid` / `examples.invalid`: minimal manifests, the same pattern MuleSoft's rulesets use.

## Repository layout (each repo)

```
ruleset.yaml               # the profile; P4A auto-discovers it at the repo root
exchange.json              # assetId, name, description, main: ruleset.yaml, version. P4A's deploy dialog prefills from it
fixtures/good/             # fully compliant manifest project; 0 findings
fixtures/bad/<rule-id>/    # violates exactly one rule; produces exactly that one finding
scripts/check.sh           # the test suite (below)
README.md                  # purpose, pairing note, rule table (id/severity/why/fix), limitations, deploy via P4A, Source Ref + date
CHANGELOG.md
.envrc                     # GH_TOKEN for tbolis-at-mulesoft
```

Each fixture folder has `mcp-metadata.json` and an `exchange.json` with `classifier: mcp-metadata`,
`descriptorVersion: 1.0.0` and `main: mcp-metadata.json`.

## Testing (`scripts/check.sh`)

1. `governance:ruleset:validate-authoring ruleset.yaml` reports no errors. Only the documented warning is allowed.
2. `governance:ruleset:validate ruleset.yaml` reports that it conforms with the dialect.
3. `governance:api:validate fixtures/good --rulesets ruleset.yaml` reports 0 findings and no `example-validation-error`.
4. For each `fixtures/bad/<id>`, the findings are exactly `{<id>}` with the expected severity.
5. Every rule ID in `ruleset.yaml` has a bad fixture, and every bad fixture maps to a rule.
6. The CLI version is printed. The script exits non-zero on any mismatch.

**Cross-checks:**
- Run each ruleset against the sibling repo's `fixtures/good`. Each ruleset's good fixture is compliant with both rulesets.
- Run each ruleset against one realistic manifest. This catches noisy rules.

## Limitations (documented in the READMEs)

- **No per-parameter rules.** Examples: bounded strings, described params, typed params. In governance-ruleset-tools 1.0.21, `mcp.JsonSchemaProperty` is never instantiated, input `properties` is an opaque `dynamic` value, and inline `rego` breaks MCP validation. We'll revisit this when the MCP model exposes parameters.
- **Transport and protocol version** are left to MuleSoft's Agent Network Best Practices.

## Delivery

1. Implement and test both repos. Safety comes first.
2. Report the results.
3. **⏸ Hold.** P4A submission waits for explicit approval. When approved:
   - `targetScopes: ["agent-network"]` (inference never sets it)
   - tags `mcp`, `agent`, plus `security` or `usability`
   - `examplesUrl` pointing to `fixtures/`
   - `sourceRef` `v1.0.0`
4. After approval and review, dogfood with `deploy_ruleset` into our own test org (approved separately) plus a draft profile in the console. This also settles how a profile targets MCP assets, since the CLI and P4A scopes list none.
