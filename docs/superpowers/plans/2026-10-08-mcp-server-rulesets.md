# MCP Server Safety & Usability Rulesets Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build two tested MuleSoft API Governance rulesets for MCP server manifests, **MCP Server Safety** (4 rules) and **MCP Server Usability** (7 rules), each in its own public repo and ready for P4A submission. Submitting is not part of this plan.

**Architecture:** Each repo is self-contained.
- `ruleset.yaml` is an AMF Validation Profile 1.0 file with `specKind: mcp` targets.
- `scripts/fixtures.py` generates every fixture from one compliant `GOOD` manifest plus one mutation per bad fixture.
- `scripts/check.sh` is the test suite:
  1. P4A tier-1 lint mirror,
  2. `validate-authoring`,
  3. dialect `validate`,
  4. `governance:api:validate` over every fixture. A bad fixture must produce **exactly one** finding, for its own rule, at the expected severity.

Every rule follows a red/green cycle: add the mutation, see `check.sh` fail, add the rule, see it pass.

**Tech Stack:**
- anypoint-cli-v4 1.6.25 with `mulesoft-anypoint-cli-governance-plugin` 1.0.21
- bash (`#!/usr/bin/env bash`)
- awk
- Python 3 stdlib only (PyYAML is **not** installed)
- direnv + gh

**Spec:** `docs/superpowers/specs/2026-10-08-mcp-server-rulesets-design.md` (in this repo)

**Repos (already created, cloned, `.envrc` allowed, default branch `main`):**
- `S` = `/Users/tbolis/ClaudeProjects/p4a/ms-omni-governance-rulesets/mcp-server-safety-ruleset` (already has the spec commit pushed)
- `U` = `/Users/tbolis/ClaudeProjects/p4a/ms-omni-governance-rulesets/mcp-server-usability-ruleset` (empty, no commits)

## Global Constraints

- **Never publish to Exchange.** No `exchange:asset:upload` and no `governance:ruleset:publish`.
- **No P4A calls:** no `submit_ruleset`, `validate_ruleset` tier-2 or `deploy_ruleset`. The user must give an explicit go first, and each call needs its own approval.
- Every profile starts with the exact line `#%Validation Profile 1.0`.
- Every profile declares `prefixes:` → `  mcp: http://anypoint.com/vocabs/mcp#`. If it's missing, the validator panics at runtime and the static checks don't catch it.
- **Property paths use `core.*` only** (e.g. `core.description`). Never write a key like `mcp.description:`. `targetClass:` values are `mcp.Tool`, `mcp.Server`, `mcp.ToolInputSchema`, `mcp.Resource` and `mcp.Prompt`.
- **`ruleset.yaml` formatting is fixed, because `check.sh` parses it with awk:**
  - two-space indent;
  - severity lists are top-level `violation:` / `warning:` / `info:` with `  - <rule-id>` entries;
  - rule definitions are `  <rule-id>:` lines directly under `validations:`.
- Rule IDs are kebab-case `[a-z0-9-]+`. A bad fixture's directory name is `<rule-id>` or `<rule-id>.<variant>`.
- **Fixture project shape:**
  - `mcp-metadata.json` + `exchange.json`, where `exchange.json` is `{"main":"mcp-metadata.json","name":…,"groupId":"p4a-fixtures","assetId":…,"version":"1.0.0","classifier":"mcp-metadata","descriptorVersion":"1.0.0"}`.
  - Never use `classifier: mcp`.
  - Never hand-edit `fixtures/`. Edit `scripts/fixtures.py` and regenerate with `python3 -I scripts/fixtures.py`.
- **Ruleset `exchange.json` is exactly these keys:** `main`, `name`, `description`, `assetId`, `version`. No `classifier`, no `groupId`.
  - Safety: `mcp-server-safety` / "MCP Server Safety" / `1.0.0`.
  - Usability: `mcp-server-usability` / "MCP Server Usability" / `1.0.0`.
- **Severities.** Safety:
  - violation: `tool-side-effect-hint-declared`, `server-auth-declared`
  - warning: `tool-input-closed`, `tool-open-world-hint-declared`

  Usability:
  - violation: `tool-description-required`
  - warning: `tool-name-format`, `tool-description-substantive`, `resource-described`, `prompt-described`, `prompt-argument-described`
  - info: `tool-output-schema-declared`
- **Every rule has** `message` (problem + fix, one sentence), `documentation` (why it matters to agents/gateways), and `examples: {valid, invalid}` (minimal manifest snippets as YAML block strings).
- **Allowed `validate-authoring` warnings:**
  - Safety: exactly 1 (`in` on `core.additionalProperties`, a node-typed property).
  - Usability: 0.
- **CLI behaviors `check.sh` relies on (verified 2026-10-08):**
  - `governance:api:validate` exits 0 even on findings, and prints `Conforms: true` when only warnings or infos fire. Assert on the `Constraint:` / `Severity:` lines, not on `Conforms` or the exit code.
  - Each finding prints `Constraint: …#/encodes/validations/<rule-id>` followed by `Severity: Violation|Warning|Info`.
- Commits end with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Push with `direnv exec . git push`.
- Run commands from the repo root (`S` or `U`). Rule docs and READMEs never cite `mulesoft-emu/*` repos.

## Review Focus

These are the input classes the spec implies but no basic test exercises. Each line gets a fixture in the task named.

1. **Annotations present, but neither `readOnlyHint` nor `destructiveHint`** (only `title` and `openWorldHint`). An author might think "annotations exist" is enough, so `tool-side-effect-hint-declared` must still fire. Pinned by the variant `tool-side-effect-hint-declared.other-hints-only` (Task 2).
2. **`destructiveHint: false` with no `readOnlyHint`.** This is an explicit declaration and must **not** fire. It's pinned by the second tool in `GOOD`, `set_alert_threshold` (Task 1). Every bad fixture inherits it, so a false positive breaks every test.
3. **`additionalProperties` missing vs `true`.** Both must fire `tool-input-closed`, and `false` must not. Pinned by `tool-input-closed.missing` and `tool-input-closed.true` (Task 4); `false` is covered by `GOOD`.
4. **A defect in a non-first item** (the second tool, or the second prompt argument). It must still fire, which proves rules apply per item and aren't limited to the first one. Pinned by `tool-side-effect-hint-declared.second-tool` (Task 2) and `prompt-argument-described.second-argument` (Task 9).
5. **Empty `securitySchemes: {}`.** Verified **not** to fire `server-auth-declared`: the profile language can't count map entries. This is a known gap, documented in the README Limitations (Task 5), not a test. **Prompt with no `arguments` at all** must not fire `prompt-argument-described`. It's pinned by the second prompt in `GOOD`, `daily_summary` (Task 1).

---

## File Structure (each repo)

| File | Responsibility |
|---|---|
| `ruleset.yaml` | The Validation Profile. Repo-specific. |
| `exchange.json` | Prefill metadata for P4A's deploy dialog. Repo-specific. |
| `scripts/fixtures.py` | `GOOD` manifest (identical in both repos) + `BAD` mutations (repo-specific). Writes `fixtures/`. |
| `fixtures/good/`, `fixtures/bad/<rule-id>[.<variant>]/` | Generated fixture projects. Committed, because they serve as the catalog's `examplesUrl`. |
| `scripts/check.sh` | The test suite. Identical in both repos except the 3 config lines at the top. |
| `README.md`, `CHANGELOG.md` | Adopter-facing docs. |
| `.envrc` | direnv: `GH_TOKEN` computed from `gh auth token --user tbolis-at-mulesoft` (no secret stored). Committed. |

---

### Task 1: Safety harness + good fixture + `server-auth-declared`

**Files:**
- Create: `S/scripts/fixtures.py`, `S/scripts/check.sh`, `S/ruleset.yaml`, `S/exchange.json`
- Commit: `S/.envrc` (already present, not yet committed)
- Generated: `S/fixtures/good/`, `S/fixtures/bad/server-auth-declared/`

**Interfaces:**
- Produces:
  - `scripts/fixtures.py`: module-level `GOOD: dict`, `BAD: dict[str, Callable[[dict], object]]` (fixture name → in-place mutation of a deep copy of `GOOD`), and helper `tool(d, i=0) -> dict`. Later tasks only add `BAD` entries.
  - `scripts/check.sh`: config vars `EXPECTED_AUTHORING_WARNINGS`, `SIBLING_GOOD`. Prints `PASS: …` lines, exits 1 with `FAIL: …` on stderr.

- [ ] **Step 1: Write the fixture generator**

`S/scripts/fixtures.py`:

```python
"""Generate fixtures/ from GOOD plus one mutation per bad fixture.

Run from the repo root:  python3 -I scripts/fixtures.py
This deletes and rewrites fixtures/. Never edit fixtures/ by hand.

Bad fixture names are <rule-id> or <rule-id>.<variant>. scripts/check.sh expects
each one to produce exactly one finding, for <rule-id>.
"""
import copy
import json
import pathlib
import shutil

ROOT = pathlib.Path(__file__).resolve().parent.parent / "fixtures"

# Compliant with BOTH the MCP Server Safety and MCP Server Usability rulesets.
# Keep this dict identical in both repos.
GOOD = {
    "protocolVersion": "2025-06-18",
    "transport": {"kind": "streamableHttp", "path": "/mcp"},
    "capabilities": {"tools": {}, "resources": {}, "prompts": {}},
    "securitySchemes": {"bearer": {"type": "http", "scheme": "bearer"}},
    "tools": [
        {
            "name": "get_weather",
            "description": "Gets the current weather for a city.",
            "annotations": {"readOnlyHint": True, "openWorldHint": True},
            "inputSchema": {
                "type": "object",
                "additionalProperties": False,
                "properties": {
                    "location": {"type": "string", "description": "City name", "maxLength": 100}
                },
                "required": ["location"],
            },
            "outputSchema": {
                "type": "object",
                "properties": {
                    "temperatureC": {"type": "number", "description": "Temperature in Celsius"}
                },
            },
        },
        {
            # destructiveHint: false (no readOnlyHint) is an explicit declaration.
            "name": "set_alert_threshold",
            "description": "Sets the temperature that triggers a weather alert for a city.",
            "annotations": {"destructiveHint": False, "openWorldHint": False},
            "inputSchema": {
                "type": "object",
                "additionalProperties": False,
                "properties": {
                    "location": {"type": "string", "description": "City name", "maxLength": 100},
                    "thresholdC": {"type": "number", "description": "Alert threshold in Celsius"},
                },
                "required": ["location", "thresholdC"],
            },
            "outputSchema": {
                "type": "object",
                "properties": {"ok": {"type": "boolean", "description": "Whether the threshold was saved"}},
            },
        },
    ],
    "resources": [
        {
            "uri": "weather://cities",
            "name": "cities",
            "description": "List of supported cities.",
            "mimeType": "application/json",
        }
    ],
    "prompts": [
        {
            "name": "weather_report",
            "description": "Writes a short weather report for a city.",
            "arguments": [
                {"name": "location", "description": "City name", "required": True},
                {"name": "tone", "description": "Writing tone, e.g. formal or casual", "required": False},
            ],
        },
        {
            # A prompt without arguments must not trip prompt-argument-described.
            "name": "daily_summary",
            "description": "Summarizes today's weather for all supported cities.",
        },
    ],
}


def tool(d, i=0):
    return d["tools"][i]


BAD = {
    "server-auth-declared": lambda d: d.pop("securitySchemes"),
}


def write(name, doc):
    path = ROOT / name
    path.mkdir(parents=True)
    (path / "mcp-metadata.json").write_text(json.dumps(doc, indent=2) + "\n")
    asset = name.replace("/", "-").replace(".", "-")
    exchange = {
        "main": "mcp-metadata.json",
        "name": asset,
        "groupId": "p4a-fixtures",
        "assetId": asset,
        "version": "1.0.0",
        "classifier": "mcp-metadata",
        "descriptorVersion": "1.0.0",
    }
    (path / "exchange.json").write_text(json.dumps(exchange, indent=2) + "\n")


def main():
    shutil.rmtree(ROOT, ignore_errors=True)
    write("good", GOOD)
    for name, mutate in BAD.items():
        doc = copy.deepcopy(GOOD)
        mutate(doc)
        write(f"bad/{name}", doc)


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Write the test suite**

`S/scripts/check.sh` (then `chmod +x scripts/check.sh`):

```bash
#!/usr/bin/env bash
# Test suite for this ruleset. Run from anywhere: scripts/check.sh
# Prints PASS lines; exits 1 with a FAIL line on the first failed assertion.
set -euo pipefail
cd "$(dirname "$0")/.."

# --- per-repo config ---
EXPECTED_AUTHORING_WARNINGS=0   # Task 4 raises this to 1 (README "Known authoring warning")
SIBLING_GOOD=../mcp-server-usability-ruleset/fixtures/good
# -----------------------

RULESET=ruleset.yaml
CLI=anypoint-cli-v4

fail() { printf 'FAIL: %s\n' "$*" >&2; exit 1; }
pass() { printf 'PASS: %s\n' "$*"; }

# Rule IDs listed under a top-level severity key (violation|warning|info).
severity_ids() {
  awk -v key="$1:" '$0 == key { on = 1; next } /^[^ ]/ { on = 0 } on && /^  - / { print $2 }' "$RULESET"
}

# Rule IDs defined directly under `validations:`.
defined_ids() {
  awk '/^validations:/ { on = 1; next } /^[^ ]/ { on = 0 } on && /^  [a-z0-9-]+:$/ { sub(":", "", $1); print $1 }' "$RULESET"
}

# CLI severity label for a rule ID (Violation|Warning|Info); fails if unlisted.
expected_severity() {
  local sev
  for sev in violation warning info; do
    if severity_ids "$sev" | grep -qx "$1"; then
      case $sev in violation) echo Violation ;; warning) echo Warning ;; info) echo Info ;; esac
      return 0
    fi
  done
  return 1
}

# Sorted "<rule-id>:<Severity>" lines for one fixture project.
findings() {
  local out
  out=$("$CLI" governance:api:validate "$1" --rulesets "$RULESET" 2>&1) || true
  grep -q '^Conforms:' <<<"$out" || fail "$1: validator did not run:"$'\n'"$out"
  if grep -q 'example-validation-error' <<<"$out"; then
    fail "$1: manifest fails the MCP schema (example-validation-error):"$'\n'"$out"
  fi
  awk '/^Constraint: / { n = split($2, a, "/"); id = a[n] } /^Severity: / { print id ":" $2 }' <<<"$out" | sort
}

lint() {
  local listed dup
  [ "$(grep -m1 -v '^[[:space:]]*$' "$RULESET")" = '#%Validation Profile 1.0' ] \
    || fail "first non-blank line must be '#%Validation Profile 1.0'"
  grep -qE '^profile: .+' "$RULESET" || fail "missing non-empty 'profile:' name"
  grep -qx '  mcp: http://anypoint.com/vocabs/mcp#' "$RULESET" \
    || fail "missing 'prefixes: mcp: http://anypoint.com/vocabs/mcp#' (validator panics without it)"
  if grep -nE '^ +mcp\.[A-Za-z]+:' "$RULESET"; then
    fail "property paths must use core.*, not mcp.* (lines above)"
  fi
  listed=$({ severity_ids violation; severity_ids warning; severity_ids info; } | sort)
  [ -n "$listed" ] || fail "no rules listed under violation/warning/info"
  dup=$(uniq -d <<<"$listed")
  [ -z "$dup" ] || fail "rules listed more than once: $dup"
  [ "$listed" = "$(defined_ids | sort)" ] || fail "severity lists and validations: keys differ"
  [ "$(wc -c <"$RULESET")" -lt 524288 ] || fail "$RULESET is larger than 512 KB"
  pass "tier-1 lint"
}

echo "CLI: $("$CLI" --version) / $("$CLI" plugins --core | grep governance-plugin)"

lint

out=$("$CLI" governance:ruleset:validate-authoring "$RULESET" 2>&1) || true
summary=$(grep -E '^[0-9]+ error\(s\), [0-9]+ warning\(s\)' <<<"$out") || fail "validate-authoring did not run:"$'\n'"$out"
[ "$summary" = "0 error(s), $EXPECTED_AUTHORING_WARNINGS warning(s)" ] \
  || fail "validate-authoring: $summary (expected 0 errors, $EXPECTED_AUTHORING_WARNINGS warnings)"$'\n'"$out"
pass "validate-authoring ($summary)"

out=$("$CLI" governance:ruleset:validate "$RULESET" 2>&1) || true
grep -q 'Ruleset conforms with Dialect' <<<"$out" || fail "dialect validation:"$'\n'"$out"
pass "dialect validate"

got=$(findings fixtures/good)
[ -z "$got" ] || fail "fixtures/good should have 0 findings, got:"$'\n'"$got"
pass "fixtures/good: 0 findings"

if [ -d "$SIBLING_GOOD" ]; then
  got=$(findings "$SIBLING_GOOD")
  [ -z "$got" ] || fail "$SIBLING_GOOD should have 0 findings, got:"$'\n'"$got"
  pass "$SIBLING_GOOD: 0 findings"
fi

for id in $(defined_ids); do
  [ -d "fixtures/bad/$id" ] || compgen -G "fixtures/bad/$id.*" >/dev/null \
    || fail "rule $id has no fixtures/bad/$id[.<variant>] fixture"
done

for dir in fixtures/bad/*/; do
  name=$(basename "$dir")
  id=${name%%.*}
  sev=$(expected_severity "$id") || fail "fixtures/bad/$name: no rule named $id in $RULESET"
  got=$(findings "$dir")
  [ "$got" = "$id:$sev" ] || fail "fixtures/bad/$name: expected exactly '$id:$sev', got:"$'\n'"${got:-<none>}"
  pass "fixtures/bad/$name -> $id ($sev)"
done

echo "ALL CHECKS PASSED"
```

- [ ] **Step 3: Write `exchange.json` and a ruleset with no rules (red)**

`S/exchange.json`:

```json
{
  "main": "ruleset.yaml",
  "name": "MCP Server Safety",
  "description": "Safety checks for MCP server manifests: declared authentication, declared tool side effects and reach, and closed tool inputs.",
  "assetId": "mcp-server-safety",
  "version": "1.0.0"
}
```

`S/ruleset.yaml` starts as a header and prefixes only, with no rules yet:

```yaml
#%Validation Profile 1.0
profile: MCP Server Safety
description: >-
  Safety checks for MCP server manifests: the server declares how clients authenticate,
  every tool declares its side effects and whether it reaches outside systems, and tool
  inputs reject undeclared fields. Pairs with MCP Server Usability and with MuleSoft's
  Agent Network Best Practices (transport and protocol version).
prefixes:
  mcp: http://anypoint.com/vocabs/mcp#
validations: {}
```

- [ ] **Step 4: Generate fixtures and run the suite to see it fail**

Run: `python3 -I scripts/fixtures.py && scripts/check.sh`

Expected: `FAIL: no rules listed under violation/warning/info`.

- [ ] **Step 5: Add `server-auth-declared`**

Replace `validations: {}` in `S/ruleset.yaml` with:

```yaml
violation:
  - server-auth-declared
validations:
  server-auth-declared:
    message: The MCP server declares no security scheme. Add at least one entry under securitySchemes (for example an HTTP bearer or OAuth 2.0 scheme).
    documentation: |
      Agents and gateways decide how to call a server from its manifest. A server that declares
      no security scheme is either open to anyone or protected in a way clients cannot discover,
      and both lead to unauthenticated tool calls or failed connections. Declare every scheme the
      server accepts so gateways can enforce it and agents can obtain the right credentials.
      Note: an empty securitySchemes map ({}) is not detected; see the README.
    examples:
      valid: |
        securitySchemes:
          bearer:
            type: http
            scheme: bearer
      invalid: |
        tools:
          - name: get_weather
        # no securitySchemes
    targetClass: mcp.Server
    propertyConstraints:
      core.securitySchemes:
        minCount: 1
```

- [ ] **Step 6: Run the suite to see it pass**

Run: `scripts/check.sh`

Expected:
- `PASS: tier-1 lint`
- `PASS: validate-authoring (0 error(s), 0 warning(s))`
- `PASS: dialect validate`
- `PASS: fixtures/good: 0 findings`
- `PASS: fixtures/bad/server-auth-declared -> server-auth-declared (Violation)`
- `ALL CHECKS PASSED`

If `fixtures/good` reports `example-validation-error`, the `GOOD` manifest violates the MCP schema. Fix `GOOD` (spike-verified shapes: `securitySchemes` entries are OpenAPI style) and don't touch the rule.

- [ ] **Step 7: Commit**

```bash
git add .envrc exchange.json ruleset.yaml scripts fixtures
git commit -m "feat: test harness, good fixture and server-auth-declared

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: `tool-side-effect-hint-declared`

**Files:**
- Modify: `S/scripts/fixtures.py` (`BAD`), `S/ruleset.yaml`
- Generated: `S/fixtures/bad/tool-side-effect-hint-declared*`

**Interfaces:**
- Consumes: `GOOD`, `BAD` and `tool(d, i)` from Task 1, plus `check.sh` unchanged.

- [ ] **Step 1: Add the failing fixtures**

Add these entries to `BAD` in `S/scripts/fixtures.py`:

```python
    "tool-side-effect-hint-declared": lambda d: tool(d).pop("annotations"),
    # Review Focus 1: annotations exist but neither side-effect hint is set.
    "tool-side-effect-hint-declared.other-hints-only": lambda d: tool(d).__setitem__(
        "annotations", {"title": "Weather", "openWorldHint": True}
    ),
    # Review Focus 4: only the second tool is missing its hint.
    "tool-side-effect-hint-declared.second-tool": lambda d: tool(d, 1)["annotations"].pop("destructiveHint"),
```

The base fixture drops `annotations` entirely. That makes `openWorldHint` absent too, but `tool-open-world-hint-declared` (Task 3) uses `nested` without `minCount`, so it stays silent when `annotations` is missing. Each fixture still produces one finding.

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I scripts/fixtures.py && scripts/check.sh`

Expected: `FAIL: fixtures/bad/tool-side-effect-hint-declared: no rule named tool-side-effect-hint-declared in ruleset.yaml`.

- [ ] **Step 3: Add the rule**

In `S/ruleset.yaml`, add `  - tool-side-effect-hint-declared` under `violation:` after `server-auth-declared`, and add this under `validations:`:

```yaml
  tool-side-effect-hint-declared:
    message: The tool does not declare its side effects. Set annotations.readOnlyHint (true for read-only tools) or annotations.destructiveHint (true or false for tools that change state).
    documentation: |
      MCP clients and gateways use tool annotations to decide whether a call needs human
      confirmation, can be retried, or may run unattended. When neither readOnlyHint nor
      destructiveHint is set, clients must assume the worst (MCP's defaults are "not read-only"
      and "destructive"), or worse, guess. Declaring one explicitly lets safe tools run
      without prompts and makes destructive tools visible to approval policies.
    examples:
      valid: |
        tools:
          - name: get_weather
            annotations:
              readOnlyHint: true
          - name: set_alert_threshold
            annotations:
              destructiveHint: false
      invalid: |
        tools:
          - name: delete_city
            annotations:
              title: Delete city
    targetClass: mcp.Tool
    propertyConstraints:
      core.annotations:
        minCount: 1
        nested:
          or:
            - propertyConstraints:
                core.readOnlyHint:
                  minCount: 1
            - propertyConstraints:
                core.destructiveHint:
                  minCount: 1
```

- [ ] **Step 4: Run to verify it passes**

Run: `scripts/check.sh`

Expected: three new `PASS: fixtures/bad/tool-side-effect-hint-declared… -> tool-side-effect-hint-declared (Violation)` lines and `ALL CHECKS PASSED`.

Pay particular attention to these two:
- `.second-tool` must pass. If it reports `<none>`, the rule only checks the first tool. Stop and report; don't weaken the fixture.
- `fixtures/good` must stay clean. That proves `destructiveHint: false` counts as declared (Review Focus 2).

- [ ] **Step 5: Commit**

```bash
git add ruleset.yaml scripts/fixtures.py fixtures
git commit -m "feat: tool-side-effect-hint-declared

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: `tool-open-world-hint-declared`

**Files:**
- Modify: `S/scripts/fixtures.py` (`BAD`), `S/ruleset.yaml`

- [ ] **Step 1: Add the failing fixture**

Add this entry to `BAD`:

```python
    "tool-open-world-hint-declared": lambda d: tool(d)["annotations"].pop("openWorldHint"),
```

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I scripts/fixtures.py && scripts/check.sh`

Expected: `FAIL: fixtures/bad/tool-open-world-hint-declared: no rule named tool-open-world-hint-declared in ruleset.yaml`.

- [ ] **Step 3: Add the rule**

Add a `warning:` list after the `violation:` list in `S/ruleset.yaml`:

```yaml
warning:
  - tool-open-world-hint-declared
```

Add this under `validations:`:

```yaml
  tool-open-world-hint-declared:
    message: The tool does not say whether it reaches outside systems. Set annotations.openWorldHint (true if it talks to the internet or third-party services, false if it only touches the server's own data).
    documentation: |
      openWorldHint tells clients whether a tool's effects stay inside a closed domain or reach
      an open one (web search, email, third-party APIs). Gateways and agents use it to apply
      stricter data-loss and prompt-injection controls to open-world tools, whose inputs and
      outputs can carry untrusted content. MCP defaults it to true when it is absent, so tools
      that only touch local data are over-restricted and real open-world tools go unnoticed.
      This rule only checks tools that have an annotations object; tools without one are
      reported by tool-side-effect-hint-declared.
    examples:
      valid: |
        tools:
          - name: get_weather
            annotations:
              readOnlyHint: true
              openWorldHint: true
      invalid: |
        tools:
          - name: get_weather
            annotations:
              readOnlyHint: true
    targetClass: mcp.Tool
    propertyConstraints:
      core.annotations:
        nested:
          propertyConstraints:
            core.openWorldHint:
              minCount: 1
```

- [ ] **Step 4: Run to verify it passes**

Run: `scripts/check.sh`

Expected:
- `PASS: fixtures/bad/tool-open-world-hint-declared -> tool-open-world-hint-declared (Warning)`
- every `tool-side-effect-hint-declared*` fixture still reports exactly one finding;
- `ALL CHECKS PASSED`.

If `fixtures/bad/tool-side-effect-hint-declared` now reports two findings, then `nested` fires on a missing `annotations`. In that case change its mutation to `tool(d).__setitem__("annotations", {"openWorldHint": True})`, regenerate, and re-run. Don't change the rule.

- [ ] **Step 5: Commit**

```bash
git add ruleset.yaml scripts/fixtures.py fixtures
git commit -m "feat: tool-open-world-hint-declared

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: `tool-input-closed`

**Files:**
- Modify: `S/scripts/fixtures.py` (`BAD`), `S/ruleset.yaml`, `S/scripts/check.sh` (restore `EXPECTED_AUTHORING_WARNINGS=1`)

- [ ] **Step 1: Add the failing fixtures (Review Focus 3)**

Add these entries to `BAD`:

```python
    "tool-input-closed.missing": lambda d: tool(d)["inputSchema"].pop("additionalProperties"),
    "tool-input-closed.true": lambda d: tool(d)["inputSchema"].__setitem__("additionalProperties", True),
```

In `S/scripts/check.sh`, change the config line to `EXPECTED_AUTHORING_WARNINGS=1   # README "Known authoring warning"`.

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I scripts/fixtures.py && scripts/check.sh`

Expected: `FAIL: validate-authoring: 0 error(s), 0 warning(s) (expected 0 errors, 1 warnings)`.

- [ ] **Step 3: Add the rule**

Add `  - tool-input-closed` under `warning:`, and add this under `validations:`:

```yaml
  tool-input-closed:
    message: The tool's input schema accepts undeclared fields. Set additionalProperties to false in inputSchema.
    documentation: |
      A tool input schema that allows additional properties lets a model (or an injected prompt)
      send fields the server never declared. Servers that pass arguments through to downstream
      APIs can then be steered into behavior the tool's contract does not describe, and gateways
      cannot validate what they cannot see. additionalProperties: false makes the declared schema
      the whole contract. This rule reports a missing additionalProperties as well as true.
    examples:
      valid: |
        tools:
          - name: get_weather
            inputSchema:
              type: object
              additionalProperties: false
              properties:
                location: { type: string }
      invalid: |
        tools:
          - name: get_weather
            inputSchema:
              type: object
              properties:
                location: { type: string }
    targetClass: mcp.ToolInputSchema
    propertyConstraints:
      core.additionalProperties:
        minCount: 1
        in: [false]
```

- [ ] **Step 4: Run to verify it passes**

Run: `scripts/check.sh`

Expected:
- `PASS: validate-authoring (0 error(s), 1 warning(s))`
- `PASS: fixtures/bad/tool-input-closed.missing -> tool-input-closed (Warning)`
- `PASS: fixtures/bad/tool-input-closed.true -> tool-input-closed (Warning)`
- `ALL CHECKS PASSED`

- [ ] **Step 5: Commit**

```bash
git add ruleset.yaml scripts fixtures
git commit -m "feat: tool-input-closed

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Safety simplify, docs, push

**Files:**
- Create: `S/README.md`, `S/CHANGELOG.md`
- Modify: `S/ruleset.yaml` (only if `simplify` finds a real simplification)

- [ ] **Step 1: Run simplify and review**

Run: `anypoint-cli-v4 governance:ruleset:simplify ruleset.yaml > /tmp/safety-simplified.yaml; diff ruleset.yaml /tmp/safety-simplified.yaml`

Apply a suggestion only if it keeps:
- `core.*` paths,
- the `prefixes` block,
- the `documentation` and `examples` keys,
- the awk-parseable layout.

The tool may strip docs or reorder keys; never accept those changes. Re-run `scripts/check.sh` after any change. Expected: `ALL CHECKS PASSED`.

- [ ] **Step 2: Write `S/README.md`**

````markdown
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
````

- [ ] **Step 3: Write `S/CHANGELOG.md`**

```markdown
# Changelog

## 1.0.0 (2026-10-08)

- Initial release: `server-auth-declared`, `tool-side-effect-hint-declared` (violation); `tool-open-world-hint-declared`, `tool-input-closed` (warning).
```

- [ ] **Step 4: Verify the README links resolve, then run the suite**

Run:

```bash
for u in https://docs.mulesoft.com/api-governance/create-custom-rulesets https://modelcontextprotocol.io/specification/2025-06-18/server/tools; do curl -s -o /dev/null -w "%{http_code} $u\n" -L "$u"; done; scripts/check.sh
```

Expected: both URLs return `200` and the suite prints `ALL CHECKS PASSED`. If a URL doesn't return 200, find the current docs page with the built-in browser, update the Source Ref, and re-run.

- [ ] **Step 5: Commit and push**

```bash
git add README.md CHANGELOG.md ruleset.yaml
git commit -m "docs: README and changelog for 1.0.0

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
direnv exec . git push
```

---

### Task 6: Usability harness + `tool-description-required`

**Files:**
- Create: `U/scripts/fixtures.py`, `U/scripts/check.sh`, `U/ruleset.yaml`, `U/exchange.json`
- Commit: `U/.envrc` (already present)

**Interfaces:**
- Consumes:
  - `S/scripts/fixtures.py` as the starting copy. `GOOD`, `tool()`, `write()` and `main()` are unchanged; replace only `BAD`.
  - `S/scripts/check.sh` as the starting copy. Only the 3 config lines change.

- [ ] **Step 1: Copy the harness and set the repo config**

Run from `U`:

```bash
mkdir -p scripts && cp ../mcp-server-safety-ruleset/scripts/fixtures.py ../mcp-server-safety-ruleset/scripts/check.sh scripts/
```

In `U/scripts/check.sh`, set the config block to:

```bash
EXPECTED_AUTHORING_WARNINGS=0
SIBLING_GOOD=../mcp-server-safety-ruleset/fixtures/good
```

and make sure the `EXPECTED_AUTHORING_WARNINGS` comment reads `# no known authoring warnings`.

In `U/scripts/fixtures.py`, replace the whole `BAD = {...}` dict with:

```python
BAD = {
    "tool-description-required": lambda d: tool(d).pop("description"),
}
```

- [ ] **Step 2: Write `exchange.json` and a ruleset with no rules (red)**

`U/exchange.json`:

```json
{
  "main": "ruleset.yaml",
  "name": "MCP Server Usability",
  "description": "Usability checks for MCP server manifests: tools, resources and prompts are named and described well enough for agents to choose and call them correctly.",
  "assetId": "mcp-server-usability",
  "version": "1.0.0"
}
```

`U/ruleset.yaml`:

```yaml
#%Validation Profile 1.0
profile: MCP Server Usability
description: >-
  Usability checks for MCP server manifests: tools have consistent names, real descriptions
  and declared outputs, and resources, prompts and prompt arguments are described, so agents
  can choose and call them correctly. Pairs with MCP Server Safety and with MuleSoft's
  Agent Network Best Practices (transport and protocol version).
prefixes:
  mcp: http://anypoint.com/vocabs/mcp#
validations: {}
```

Run: `python3 -I scripts/fixtures.py && scripts/check.sh`

Expected: `FAIL: no rules listed under violation/warning/info`.

- [ ] **Step 3: Add the rule**

Replace `validations: {}` with:

```yaml
violation:
  - tool-description-required
validations:
  tool-description-required:
    message: The tool has no description. Add one that tells an agent what the tool does and when to call it.
    documentation: |
      A model chooses tools almost entirely from their descriptions. A tool without one is
      invisible to tool selection at best, and called for the wrong job at worst. The
      description is the tool's contract with the agent: what it does, when to use it, and
      what it returns.
    examples:
      valid: |
        tools:
          - name: get_weather
            description: Gets the current weather for a city.
      invalid: |
        tools:
          - name: get_weather
    targetClass: mcp.Tool
    propertyConstraints:
      core.description:
        minCount: 1
```

- [ ] **Step 4: Run to verify it passes**

Run: `scripts/check.sh`

Expected:
- `PASS: validate-authoring (0 error(s), 0 warning(s))`
- `PASS: fixtures/good: 0 findings`
- `PASS: ../mcp-server-safety-ruleset/fixtures/good: 0 findings`
- `PASS: fixtures/bad/tool-description-required -> tool-description-required (Violation)`
- `ALL CHECKS PASSED`

- [ ] **Step 5: Commit (first commit in U)**

```bash
git add .envrc exchange.json ruleset.yaml scripts fixtures
git commit -m "feat: test harness, good fixture and tool-description-required

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: `tool-name-format` + `tool-description-substantive`

**Files:**
- Modify: `U/scripts/fixtures.py` (`BAD`), `U/ruleset.yaml`

- [ ] **Step 1: Add the failing fixtures**

Add these entries to `BAD`:

```python
    "tool-name-format": lambda d: tool(d).__setitem__("name", "GetWeather"),
    "tool-name-format.too-long": lambda d: tool(d).__setitem__("name", "a" * 65),
    "tool-description-substantive": lambda d: tool(d).__setitem__("description", "Weather."),
    # 19 characters: one below the minimum.
    "tool-description-substantive.19-chars": lambda d: tool(d).__setitem__("description", "Gets city weather!!"),
```

Check: `len("Gets city weather!!") == 19`.

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I scripts/fixtures.py && scripts/check.sh`

Expected: `FAIL: fixtures/bad/tool-description-substantive: no rule named tool-description-substantive in ruleset.yaml`. Fixtures are checked in glob order, so `tool-description-required` passes first.

- [ ] **Step 3: Add the rules**

Add a `warning:` list after `violation:`:

```yaml
warning:
  - tool-name-format
  - tool-description-substantive
```

Add these under `validations:`:

```yaml
  tool-name-format:
    message: The tool name is not lower snake_case. Rename it to start with a lowercase letter and use only lowercase letters, digits and underscores (at most 64 characters).
    documentation: |
      Tool names are identifiers that models emit verbatim in tool calls, and many clients and
      LLM APIs restrict them (64 characters, a limited character set). Lower snake_case avoids
      rejected calls, keeps names stable when gateways merge tools from several servers, and
      matches the convention most MCP servers use.
    examples:
      valid: |
        tools:
          - name: get_weather
      invalid: |
        tools:
          - name: GetWeather
    targetClass: mcp.Tool
    propertyConstraints:
      core.name:
        pattern: "^[a-z][a-z0-9_]{0,63}$"
  tool-description-substantive:
    message: The tool description is shorter than 20 characters. Expand it to say what the tool does and when an agent should call it.
    documentation: |
      One-word descriptions ("Weather.", "Search") give a model nothing to choose between
      similar tools. Twenty characters is a deliberately low floor: enough for a verb, an
      object and a qualifier. Missing descriptions are reported by tool-description-required,
      not by this rule.
    examples:
      valid: |
        tools:
          - name: get_weather
            description: Gets the current weather for a city.
      invalid: |
        tools:
          - name: get_weather
            description: Weather.
    targetClass: mcp.Tool
    propertyConstraints:
      core.description:
        minLength: 20
```

- [ ] **Step 4: Run to verify it passes**

Run: `scripts/check.sh`

Expected:
- four new PASS lines, each `(Warning)`;
- `fixtures/bad/tool-description-required` still reports only `tool-description-required` (verified in the spike: `minLength` doesn't fire on a missing value);
- `ALL CHECKS PASSED`.

- [ ] **Step 5: Commit**

```bash
git add ruleset.yaml scripts/fixtures.py fixtures
git commit -m "feat: tool-name-format and tool-description-substantive

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: `resource-described` + `prompt-described`

**Files:**
- Modify: `U/scripts/fixtures.py` (`BAD`), `U/ruleset.yaml`

- [ ] **Step 1: Add the failing fixtures**

Add these entries to `BAD`:

```python
    "resource-described.no-description": lambda d: d["resources"][0].pop("description"),
    "resource-described.no-mime-type": lambda d: d["resources"][0].pop("mimeType"),
    "prompt-described": lambda d: d["prompts"][1].pop("description"),
```

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I scripts/fixtures.py && scripts/check.sh`

Expected: `FAIL: fixtures/bad/prompt-described: no rule named prompt-described in ruleset.yaml`.

- [ ] **Step 3: Add the rules**

Append under `warning:`:

```yaml
  - resource-described
  - prompt-described
```

Add these under `validations:`:

```yaml
  resource-described:
    message: The resource is missing a description or a mimeType. Add both so agents know what the resource contains and how to read it.
    documentation: |
      Agents list resources and decide which to read from their descriptions, and they need
      the mimeType to parse what comes back. A resource without either is effectively
      undiscoverable, or is read and then misinterpreted.
    examples:
      valid: |
        resources:
          - uri: weather://cities
            name: cities
            description: List of supported cities.
            mimeType: application/json
      invalid: |
        resources:
          - uri: weather://cities
            name: cities
    targetClass: mcp.Resource
    propertyConstraints:
      core.description:
        minCount: 1
      core.mimeType:
        minCount: 1
  prompt-described:
    message: The prompt has no description. Add one that says what the prompt produces and when to use it.
    documentation: |
      Prompts are surfaced to users and agents as reusable templates. Without a description,
      a client can only show the prompt's name, and nobody can tell what it does without running it.
    examples:
      valid: |
        prompts:
          - name: weather_report
            description: Writes a short weather report for a city.
      invalid: |
        prompts:
          - name: weather_report
    targetClass: mcp.Prompt
    propertyConstraints:
      core.description:
        minCount: 1
```

- [ ] **Step 4: Run to verify it passes**

Run: `scripts/check.sh`

Expected: `resource-described.no-description`, `resource-described.no-mime-type` and `prompt-described` each pass with `(Warning)`, then `ALL CHECKS PASSED`.

Each `resource-described` variant must report exactly **one** finding. If a single resource missing one field reports two, the CLI is reporting one finding per property; stop and report.

- [ ] **Step 5: Commit**

```bash
git add ruleset.yaml scripts/fixtures.py fixtures
git commit -m "feat: resource-described and prompt-described

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: `prompt-argument-described` + `tool-output-schema-declared`

**Files:**
- Modify: `U/scripts/fixtures.py` (`BAD`), `U/ruleset.yaml`

- [ ] **Step 1: Add the failing fixtures**

Add these entries to `BAD`:

```python
    "prompt-argument-described": lambda d: d["prompts"][0]["arguments"][0].pop("description"),
    # Review Focus 4: only the second argument is undescribed.
    "prompt-argument-described.second-argument": lambda d: d["prompts"][0]["arguments"][1].pop("description"),
    "tool-output-schema-declared": lambda d: tool(d).pop("outputSchema"),
```

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I scripts/fixtures.py && scripts/check.sh`

Expected: `FAIL: fixtures/bad/prompt-argument-described: no rule named prompt-argument-described in ruleset.yaml`.

- [ ] **Step 3: Add the rules**

Append `  - prompt-argument-described` under `warning:`, then add an `info:` list after `warning:`:

```yaml
info:
  - tool-output-schema-declared
```

Add these under `validations:`:

```yaml
  prompt-argument-described:
    message: A prompt argument has no description. Describe every argument so users and agents know what value to supply.
    documentation: |
      Clients render prompt arguments as a form, and agents fill them in from context.
      An argument with only a name ("text", "id") leaves both guessing at format and meaning.
      Prompts without arguments are not affected by this rule.
    examples:
      valid: |
        prompts:
          - name: weather_report
            description: Writes a short weather report for a city.
            arguments:
              - name: location
                description: City name
      invalid: |
        prompts:
          - name: weather_report
            description: Writes a short weather report for a city.
            arguments:
              - name: location
    targetClass: mcp.Prompt
    propertyConstraints:
      core.arguments:
        nested:
          propertyConstraints:
            core.description:
              minCount: 1
  tool-output-schema-declared:
    message: The tool declares no outputSchema. Add one so agents and gateways know the shape of its structured result.
    documentation: |
      Since MCP 2025-06-18, tools can declare an outputSchema for their structuredContent.
      With it, agents can chain tool results without guessing their shape, and gateways can
      validate or redact responses. It is optional in the protocol, so this is informational:
      prefer it for any tool that returns data rather than prose.
    examples:
      valid: |
        tools:
          - name: get_weather
            outputSchema:
              type: object
              properties:
                temperatureC: { type: number }
      invalid: |
        tools:
          - name: get_weather
    targetClass: mcp.Tool
    propertyConstraints:
      core.outputSchema:
        minCount: 1
```

- [ ] **Step 4: Run to verify it passes**

Run: `scripts/check.sh`

Expected:
- `PASS: fixtures/bad/prompt-argument-described -> prompt-argument-described (Warning)`
- `PASS: fixtures/bad/prompt-argument-described.second-argument -> prompt-argument-described (Warning)`
- `PASS: fixtures/bad/tool-output-schema-declared -> tool-output-schema-declared (Info)`
- `ALL CHECKS PASSED`

If the info fixture fails with `got: tool-output-schema-declared:<Label>` and `<Label>` isn't `Info`, change the `info) echo Info ;;` arm of `expected_severity` in **both** repos' `check.sh` to `<Label>`. Commit the Safety-side change separately in `S`.

- [ ] **Step 5: Commit**

```bash
git add ruleset.yaml scripts fixtures
git commit -m "feat: prompt-argument-described and tool-output-schema-declared

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Usability simplify, docs, push

**Files:**
- Create: `U/README.md`, `U/CHANGELOG.md`
- Modify: `U/ruleset.yaml` (only if `simplify` finds a real simplification)

- [ ] **Step 1: Run simplify and review**

Run: `anypoint-cli-v4 governance:ruleset:simplify ruleset.yaml > /tmp/usability-simplified.yaml; diff ruleset.yaml /tmp/usability-simplified.yaml`

Apply the same acceptance rule as Task 5 Step 1, then re-run `scripts/check.sh`. Expected: `ALL CHECKS PASSED`.

- [ ] **Step 2: Write `U/README.md`**

````markdown
# MCP Server Usability

A MuleSoft API Governance ruleset (AMF Validation Profile 1.0) for **MCP server** assets. It checks that an MCP server's tools, resources and prompts are named and described well enough for an agent to pick the right one and call it correctly.

**Pairs with:**
- [MCP Server Safety](https://github.com/P4A-Policies-for-Agents/mcp-server-safety-ruleset) covers declared authentication, tool side effects and closed tool inputs.
- MuleSoft's **Agent Network Best Practices** ruleset covers MCP transport and protocol version, which this ruleset deliberately leaves out.

## Rules

| Rule | Severity | Why | Fix |
|---|---|---|---|
| `tool-description-required` | violation | Models choose tools from their descriptions. | Add a `description`. |
| `tool-name-format` | warning | Clients and LLM APIs reject or mangle unusual names. | Use lower snake_case, ≤ 64 chars. |
| `tool-description-substantive` | warning | One-word descriptions can't be told apart. | Write at least 20 characters. |
| `resource-described` | warning | Agents can't find or parse undescribed resources. | Add `description` and `mimeType`. |
| `prompt-described` | warning | Users can't tell what a prompt does. | Add a `description`. |
| `prompt-argument-described` | warning | Users and agents guess argument values. | Describe every argument. |
| `tool-output-schema-declared` | info | Agents can't chain results of unknown shape. | Add an `outputSchema`. |

Each rule's `documentation` and `examples` in [`ruleset.yaml`](ruleset.yaml) explain it in full. [`fixtures/`](fixtures) holds a compliant manifest (`good/`) and one failing manifest per rule (`bad/<rule>/`).

## Deploy it to your org

This ruleset is published through the P4A catalog (https://www.p4a.ai). Open its catalog entry and use **Publish to Exchange** (or the P4A MCP server's `deploy_ruleset`) to publish it into your own Anypoint organization, then add it to a governance profile that targets your MCP server assets.

## Limitations

- **Tool parameters are not checked.** Rules like "every parameter has a description" or "every parameter has a type" need access to individual input-schema properties. In governance-ruleset-tools 1.0.21 the MCP model exposes input `properties` only as an opaque value, and custom Rego rules are not supported for MCP assets.
- **Description quality is a length floor.** 20 characters can still be unhelpful; review descriptions as part of your API review.

## Development

```bash
python3 -I scripts/fixtures.py   # regenerate fixtures/ from scripts/fixtures.py
scripts/check.sh                 # lint + authoring + dialect + every fixture
```

Bump `version` in `exchange.json` for every rule change; Exchange versions are immutable. Design: [design spec](https://github.com/P4A-Policies-for-Agents/mcp-server-safety-ruleset/blob/main/docs/superpowers/specs/2026-10-08-mcp-server-rulesets-design.md) (in the Safety repo).

> Source Ref: [MuleSoft API Governance: custom rulesets](https://docs.mulesoft.com/api-governance/create-custom-rulesets), [MCP tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools), [MCP resources](https://modelcontextprotocol.io/specification/2025-06-18/server/resources), [MCP prompts](https://modelcontextprotocol.io/specification/2025-06-18/server/prompts). Snapshot 2026-10-08; validated with anypoint-cli-v4 1.6.25 / governance plugin 1.0.21.
````

- [ ] **Step 3: Write `U/CHANGELOG.md`**

```markdown
# Changelog

## 1.0.0 (2026-10-08)

- Initial release: `tool-description-required` (violation); `tool-name-format`, `tool-description-substantive`, `resource-described`, `prompt-described`, `prompt-argument-described` (warning); `tool-output-schema-declared` (info).
```

- [ ] **Step 4: Verify the README links resolve, then run the suite**

Run:

```bash
for u in https://docs.mulesoft.com/api-governance/create-custom-rulesets https://modelcontextprotocol.io/specification/2025-06-18/server/tools https://modelcontextprotocol.io/specification/2025-06-18/server/resources https://modelcontextprotocol.io/specification/2025-06-18/server/prompts; do curl -s -o /dev/null -w "%{http_code} $u\n" -L "$u"; done; scripts/check.sh
```

Expected: all four URLs return `200` and the suite prints `ALL CHECKS PASSED`.

- [ ] **Step 5: Commit and push**

```bash
git add README.md CHANGELOG.md ruleset.yaml
git commit -m "docs: README and changelog for 1.0.0

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
direnv exec . git push -u origin main
```

---

### Task 11: Cross-check, realistic manifest, report (no P4A)

**Files:** none committed. Work happens in `/tmp/mcp-realistic/`.

- [ ] **Step 1: Re-run both suites (Safety now sees the Usability sibling)**

Run: `(cd S && scripts/check.sh) && (cd U && scripts/check.sh)`

Expected:
- both print `ALL CHECKS PASSED`;
- Safety now includes `PASS: ../mcp-server-usability-ruleset/fixtures/good: 0 findings`;
- the two `GOOD` dicts are identical: `diff <(sed -n '/^GOOD = {/,/^}/p' S/scripts/fixtures.py) <(sed -n '/^GOOD = {/,/^}/p' U/scripts/fixtures.py)` prints nothing.

- [ ] **Step 2: Build a realistic manifest**

Write `/tmp/mcp-realistic/mcp-metadata.json`. It describes a plausible third-party-style server ("issue tracker") as a typical author would write it, with no ruleset in mind:
- 5 tools: `search_issues`, `get_issue`, `create_issue`, `update_issue`, `delete_issue`, with mixed annotations (some missing), realistic descriptions, one tool with `additionalProperties` omitted, and no `outputSchema` on 3 tools;
- 1 resource without `mimeType`;
- 1 prompt with 2 arguments;
- `securitySchemes` with OAuth 2.0 `{"type":"oauth2","flows":{"clientCredentials":{"tokenUrl":"https://example.com/token","scopes":{}}}}`. If this yields `example-validation-error`, fall back to `{"type":"http","scheme":"bearer"}` and note it.

Copy `exchange.json` from `S/fixtures/good/` and change `name`/`assetId` to `realistic`.

- [ ] **Step 3: Run both rulesets on it and judge noise**

Run:

```bash
for r in S U; do anypoint-cli-v4 governance:api:validate /tmp/mcp-realistic --rulesets $r/ruleset.yaml 2>&1 | rg '^(Conforms|Number|Constraint|Severity|Target)'; done
```

Expected:
- every finding maps to a defect deliberately put into the manifest in Step 2;
- no `example-validation-error`;
- no finding on a compliant element.

Any finding on a compliant element is a noisy rule. Stop and report it with the manifest excerpt; don't patch it silently.

- [ ] **Step 4: Report to the user and HOLD**

Report:
- both `check.sh` outputs (PASS line counts, CLI version);
- the realistic-manifest findings table;
- the `simplify` outcomes;
- the known gaps (empty `securitySchemes`; per-parameter rules).

Then **stop**. Remaining after the user's explicit go, each step needing separate approval:
1. tag `v1.0.0` in both repos;
2. P4A submission ×2 (`targetScopes: ["agent-network"]`, tags `mcp`, `agent` + `security`/`usability`, `examplesUrl` → `fixtures/`, `sourceRef` `v1.0.0`);
3. dogfood `deploy_ruleset` + draft console profile.
