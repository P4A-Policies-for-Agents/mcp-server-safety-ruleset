# MCP Server Safety

A MuleSoft API Governance ruleset (AMF Validation Profile 1.0) for **MCP server** assets. It checks that an MCP server's manifest declares how clients authenticate, what each tool does to the world, and that tool inputs reject undeclared fields: the things gateways and agents need to call the server safely.

**Pairs with:**
- [MCP Server Usability](https://github.com/P4A-Policies-for-Agents/mcp-server-usability-ruleset) checks that tools, resources and prompts are described well enough for agents.
- MuleSoft's **Agent Network Best Practices** ruleset covers MCP transport and protocol version, which this ruleset deliberately leaves out.

## Rules

| Rule | Severity | Why | Fix |
|---|---|---|---|
| `server-auth-declared` | violation | Clients can't discover how to authenticate, and gateways can't enforce it. | Add an entry under `securitySchemes`. |
| `tool-side-effect-hint-declared` | violation | Without it, approval policies can't tell safe tools from destructive ones. | Set `annotations.readOnlyHint` or `annotations.destructiveHint`. |
| `tool-open-world-hint-declared` | warning | Tools that reach outside systems need stricter data and injection controls. | Set `annotations.openWorldHint`. |
| `tool-input-closed` | warning | Undeclared input fields bypass the tool's contract. | Set `inputSchema.additionalProperties: false`. |

Each rule's `documentation` and `examples` in [`ruleset.yaml`](ruleset.yaml) explain it in full. [`fixtures/`](fixtures) holds a compliant manifest (`good/`) and one failing manifest per rule (`bad/<rule>/`).

## Deploy it to your org

This ruleset is published through the P4A catalog (https://www.p4a.ai). Open its catalog entry and use **Publish to Exchange** (or the P4A MCP server's `deploy_ruleset`) to publish it into your own Anypoint organization, then add it to a governance profile that targets your MCP server assets.

## Limitations

- **Per-parameter checks are not possible yet.** Bounded strings, described parameters and typed parameters all need access to individual input-schema properties. In governance-ruleset-tools 1.0.21 the MCP model exposes input `properties` only as an opaque value, and custom Rego rules are not supported for MCP assets.
- **An empty `securitySchemes: {}` passes `server-auth-declared`.** Profiles can check that the key is present but not count its entries.
- **Hints are declarations, not proof.** The ruleset checks that annotations are declared, not that they are true. Runtime enforcement belongs in gateway policies.

## Known authoring warning

`governance:ruleset:validate-authoring` reports one warning: `in` is used on `core.additionalProperties`. This is intentional. It's the only constraint that tells `false` apart from `true` or a missing value, and it works at validation time (see `fixtures/bad/tool-input-closed.*`).

## Development

```bash
python3 -I scripts/fixtures.py   # regenerate fixtures/ from scripts/fixtures.py
scripts/check.sh                 # lint + authoring + dialect + every fixture
```

Bump `version` in `exchange.json` for every rule change; Exchange versions are immutable. Design: [`docs/superpowers/specs/2026-10-08-mcp-server-rulesets-design.md`](docs/superpowers/specs/2026-10-08-mcp-server-rulesets-design.md).

> Source Ref: [MuleSoft API Governance: custom rulesets](https://docs.mulesoft.com/api-governance/create-custom-rulesets), [MCP tool annotations](https://modelcontextprotocol.io/specification/2025-06-18/server/tools). Snapshot 2026-10-08; validated with anypoint-cli-v4 1.6.25 / governance plugin 1.0.21.
