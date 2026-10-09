# Changelog

## 1.0.0 (2026-10-08)

- Initial release: `server-auth-declared`, `tool-side-effect-hint-declared` (violation); `tool-open-world-hint-declared`, `tool-input-closed` (warning).
- Fix false findings on governance plugin 1.1.x, which hosted Anypoint governance runs: element paths now use `mcp.*` (plugin 1.1.x renamed them from `core.*`). Needs plugin 1.1.4 or later.
- `server-auth-declared` now targets `core.encodes`, the type plugin 1.1.x gives the manifest root. On `mcp.Server` it never ran.
