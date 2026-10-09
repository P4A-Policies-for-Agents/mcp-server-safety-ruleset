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

## Requirements

Governance plugin 1.1.4 or later (`anypoint-cli-v4 plugins --core`). Plugin 1.1.x names MCP element fields `mcp.*` (for example `mcp.description`); 1.0.x named them `core.*`, so this version does not work on 1.0.x. `scripts/check.sh` checks the plugin version.

## Deploy it to your org

This ruleset is published through the P4A catalog (https://www.p4a.ai). Open its catalog entry and use **Publish to Exchange** (or the P4A MCP server's `deploy_ruleset`) to publish it into your own Anypoint organization, then add it to a governance profile that targets your MCP server assets.

## Limitations

- **Manifest-level rules key on `transport`.** `server-auth-declared` checks only documents with a `transport` field, which the MCP schema requires and A2A Agent Cards don't have. That keeps it off Agent Cards attached to the same profile (`fixtures/scope/`).
- **Per-parameter checks are not possible yet.** Bounded strings, described parameters and typed parameters all need access to individual input-schema properties. In governance plugin 1.0.21 the MCP model exposes input `properties` only as an opaque value, and custom Rego rules are not supported for MCP assets.
- **An empty `securitySchemes: {}` passes `server-auth-declared`.** Profiles can check that the key is present but not count its entries.
- **Hints are declarations, not proof.** The ruleset checks that annotations are declared, not that they are true. Runtime enforcement belongs in gateway policies.

## Known authoring error and warning

`governance:ruleset:validate-authoring` reports one error: `server-auth-declared` uses `targetClass: core.encodes`, which the linter calls invalid. The linter's MCP metadata is out of date. It lists an `mcp.Server` class, but plugin 1.1.x never types the manifest root as `mcp:Server`, so a rule on `mcp.Server` would never run. The root is typed `core:encodes`, and the rule fires correctly on it (see `fixtures/bad/server-auth-declared`).

It also reports one warning: `in` is used on `mcp.additionalProperties`. This is intentional. It's the only constraint that tells `false` apart from `true` or a missing value, and it works at validation time (see `fixtures/bad/tool-input-closed.*`).

## Test on Exchange assets

To try the ruleset on real assets, publish the fixtures to a test business group, then attach
the ruleset to them, for example with a draft governance profile. Copy `.env.example` to `.env`
and fill in a connected app and business group ID; `.env` is gitignored.

```bash
scripts/publish-examples.sh --dry-run   # list the assets
scripts/publish-examples.sh             # <prefix>-ok plus one <prefix>-<rule-id> per rule
scripts/publish-examples.sh --all       # also every bad variant and the scope fixtures
scripts/cleanup-examples.sh             # soft-delete them all after testing (--hard, --yes)
```

`<prefix>-ok` should give 0 findings, and each `<prefix>-<rule-id>` exactly the finding named in
its description. Publishing skips versions that already exist; to republish changed fixtures,
clean up first or set `EXAMPLES_VERSION`.

## Development

```bash
python3 -I scripts/fixtures.py   # regenerate fixtures/ from scripts/fixtures.py
scripts/check.sh                 # lint + authoring + dialect + every fixture
```

Bump `version` in `exchange.json` for every rule change; Exchange versions are immutable. Design: [`docs/superpowers/specs/2026-10-08-mcp-server-rulesets-design.md`](docs/superpowers/specs/2026-10-08-mcp-server-rulesets-design.md).

> Source Ref: [MuleSoft API Governance: custom rulesets](https://docs.mulesoft.com/api-governance/create-custom-rulesets), [MCP tool annotations](https://modelcontextprotocol.io/specification/2025-06-18/server/tools). Snapshot 2026-10-09; validated with anypoint-cli-v4 1.6.25 / governance plugin 1.1.4.
