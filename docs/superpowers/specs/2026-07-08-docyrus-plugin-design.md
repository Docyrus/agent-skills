# Docyrus Plugin Design

## Goal

Create a local Codex plugin named `docyrus` that bundles the Docyrus skills from this repository and a vendored snapshot of `docyrus/design-skills`, registers itself in the default personal marketplace, and documents the `docyrus` CLI as a prerequisite without trying to install it.

## Scope

- Create the plugin at `~/plugins/docyrus`
- Add or update the personal marketplace entry at `~/.agents/plugins/marketplace.json`
- Bundle all skill directories from `/Users/anilbeyazoglu/Dev/agent-skills/skills`
- Bundle all skill directories from `https://github.com/docyrus/design-skills`
- Skip MCP server wiring
- Skip `.app.json`
- Document `npm install -g @docyrus/docyrus@latest` as a prerequisite

## Architecture

The plugin is a single self-contained local plugin. It uses a merged `skills/` directory under the plugin root and relies on standard Codex plugin discovery through `.codex-plugin/plugin.json`. The design-skills repository is vendored as a snapshot so the plugin remains reproducible and does not depend on external filesystem paths after creation.

## Plugin Structure

```text
~/plugins/docyrus/
  .codex-plugin/plugin.json
  skills/
    <all skill directories from this repo>
    <all skill directories from docyrus/design-skills>
```

There is no `.mcp.json` and no `.app.json` in this first version.

## Skill Merge Rules

- Copy the full contents of `/Users/anilbeyazoglu/Dev/agent-skills/skills` into the plugin `skills/` directory.
- Copy the full contents of `docyrus/design-skills/skills` into the same plugin `skills/` directory.
- Preserve each skill directory intact, including `SKILL.md`, `references/`, and `assets/`.
- Current inspection shows no top-level directory name collisions between the two sources, so no renaming layer is required.

## Manifest Rules

`plugin.json` should:

- use the normalized plugin name `docyrus`
- declare `skills: "./skills/"`
- omit `mcpServers`
- omit `apps`
- present Docyrus as a developer-tools plugin
- mention that some skills expect the Docyrus CLI to be installed separately

## CLI Prerequisite

The plugin must not try to install global software or run `npm install -g` as part of installation. Instead, the plugin metadata should state that users who need CLI-backed skills should install:

```bash
npm install -g @docyrus/docyrus@latest
```

## Marketplace Rules

- Use the default personal marketplace at `~/.agents/plugins/marketplace.json`
- Create it if it does not exist
- Add the plugin entry with:
  - `policy.installation: "AVAILABLE"`
  - `policy.authentication: "ON_INSTALL"`
  - an explicit category

## Validation

Validate the finished plugin with:

```bash
python3 /Users/anilbeyazoglu/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py ~/plugins/docyrus
```

## Non-Goals

- Installing the Docyrus CLI automatically
- Wiring any MCP server
- Declaring any Codex app/connector dependencies
- Publishing or committing the plugin to a remote repository
