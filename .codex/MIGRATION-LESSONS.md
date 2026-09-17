# AEM Forms migration lessons

Codex agent routing is generated from the authoritative Claude agent prompts. The migration-agent
prompt lives at `../.claude/agents/migrate-form/AGENT.md`; do not duplicate it in generated TOML.

Apply these non-negotiable checks to every Foundation/on-premise to Core Components migration:

- Extract every source rule before implementing it. Preserve the source event, operators,
  operands, literal bounds, and script; never infer the logic from the field label or a message.
- Use a known-good, editor-authored Core Rule Editor AST as the structural template. Verify each
  deployed multi-value `fd:*` JCR property parses independently and that the Rule Editor displays a
  complete rule without console JSON/React errors.
- A legacy `css` property is not a Core Components DOM class. Target the actual rendered Core
  Components DOM beneath the specific form container. Explicitly delete obsolete live JCR children;
  a package update does not reliably remove nodes merely omitted from source.
- For PDF submit, verify selection/wiring, validation gating, and an actual valid POST whose body
  begins `%PDF-`; compile success is insufficient.
- Investigate a form-model/schema importer error through the DAM asset descriptor and SDK logs
  before removing a schema association. Remove it only if optional and bindings remain correct.
