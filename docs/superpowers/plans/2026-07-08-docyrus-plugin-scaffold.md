# Docyrus Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local `docyrus` Codex plugin that bundles both Docyrus skill sources, registers it in the personal marketplace, and validates the manifest.

**Architecture:** Scaffold a single plugin into `~/plugins/docyrus`, then populate its `skills/` directory from the two source repositories. Keep the plugin self-contained, skip MCP and apps, and document the CLI prerequisite in manifest text only.

**Tech Stack:** Codex plugin scaffold scripts, JSON manifests, filesystem copy operations, git clone, plugin validator

## Global Constraints

- Plugin root must be `/Users/anilbeyazoglu/plugins/docyrus`
- Marketplace path must be `/Users/anilbeyazoglu/.agents/plugins/marketplace.json`
- Include all directories from `/Users/anilbeyazoglu/Dev/agent-skills/skills`
- Include all directories from `https://github.com/docyrus/design-skills` under the plugin `skills/` directory
- Do not create `.mcp.json`
- Do not create `.app.json`
- Document `npm install -g @docyrus/docyrus@latest` as a prerequisite; do not execute it

---

### Task 1: Scaffold the plugin and marketplace entry

**Files:**
- Create: `/Users/anilbeyazoglu/plugins/docyrus/.codex-plugin/plugin.json`
- Create: `/Users/anilbeyazoglu/plugins/docyrus/skills/`
- Create: `/Users/anilbeyazoglu/.agents/plugins/marketplace.json`

**Interfaces:**
- Consumes: plugin creator scaffold script at `/Users/anilbeyazoglu/.codex/skills/.system/plugin-creator/scripts/create_basic_plugin.py`
- Produces: plugin root `/Users/anilbeyazoglu/plugins/docyrus`

- [ ] **Step 1: Run the scaffold command**

```bash
python3 /Users/anilbeyazoglu/.codex/skills/.system/plugin-creator/scripts/create_basic_plugin.py \
  docyrus \
  --with-skills \
  --with-marketplace \
  --category "Developer Tools"
```

- [ ] **Step 2: Verify scaffold output exists**

Run:

```bash
test -f /Users/anilbeyazoglu/plugins/docyrus/.codex-plugin/plugin.json
test -d /Users/anilbeyazoglu/plugins/docyrus/skills
test -f /Users/anilbeyazoglu/.agents/plugins/marketplace.json
```

Expected: all three checks succeed with exit code `0`.

- [ ] **Step 3: Inspect the generated manifest and marketplace entry**

Run:

```bash
sed -n '1,220p' /Users/anilbeyazoglu/plugins/docyrus/.codex-plugin/plugin.json
sed -n '1,220p' /Users/anilbeyazoglu/.agents/plugins/marketplace.json
```

Expected: plugin name is `docyrus`, `skills` points to `./skills/`, and the marketplace contains a `docyrus` entry.

### Task 2: Populate skills and update plugin metadata

**Files:**
- Modify: `/Users/anilbeyazoglu/plugins/docyrus/.codex-plugin/plugin.json`
- Modify: `/Users/anilbeyazoglu/plugins/docyrus/skills/*`
- Create: `/Users/anilbeyazoglu/plugins/docyrus/PLUGIN_NOTES.md`

**Interfaces:**
- Consumes: local skill source `/Users/anilbeyazoglu/Dev/agent-skills/skills`
- Consumes: remote skill source `https://github.com/docyrus/design-skills`
- Produces: merged plugin skill tree and finalized manifest metadata

- [ ] **Step 1: Copy the local skill tree into the plugin**

Run:

```bash
rsync -a /Users/anilbeyazoglu/Dev/agent-skills/skills/ /Users/anilbeyazoglu/plugins/docyrus/skills/
```

Expected: all local Docyrus skill directories appear under `/Users/anilbeyazoglu/plugins/docyrus/skills`.

- [ ] **Step 2: Clone the design-skills repository**

Run:

```bash
rm -rf /tmp/docyrus-design-skills
git clone --depth=1 https://github.com/docyrus/design-skills /tmp/docyrus-design-skills
```

Expected: `/tmp/docyrus-design-skills/skills` exists.

- [ ] **Step 3: Copy the design skills into the plugin**

Run:

```bash
rsync -a /tmp/docyrus-design-skills/skills/ /Users/anilbeyazoglu/plugins/docyrus/skills/
```

Expected: the plugin `skills/` directory contains both the local Docyrus skills and the design-skills directories.

- [ ] **Step 4: Update plugin manifest metadata**

Write this JSON to `/Users/anilbeyazoglu/plugins/docyrus/.codex-plugin/plugin.json`:

```json
{
  "name": "docyrus",
  "version": "0.1.0",
  "description": "Docyrus skills plugin bundling platform, design, CLI, and app-development guidance for Codex.",
  "author": {
    "name": "Docyrus",
    "url": "https://github.com/docyrus"
  },
  "homepage": "https://github.com/docyrus/agent-skills",
  "repository": "https://github.com/docyrus/agent-skills",
  "license": "MIT",
  "keywords": [
    "docyrus",
    "skills",
    "design",
    "cli",
    "api",
    "react"
  ],
  "skills": "./skills/",
  "interface": {
    "displayName": "Docyrus",
    "shortDescription": "Docyrus platform and design skills for Codex",
    "longDescription": "Use Docyrus skills in Codex for platform architecture, API and React app development, design workflows, data sources, automations, and related tasks. Some skills expect the Docyrus CLI to be installed separately with npm install -g @docyrus/docyrus@latest.",
    "developerName": "Docyrus",
    "category": "Developer Tools",
    "capabilities": [
      "Interactive",
      "Read",
      "Write"
    ],
    "websiteURL": "https://github.com/docyrus/agent-skills",
    "defaultPrompt": [
      "Help me build on the Docyrus platform.",
      "Use Docyrus skills to design a page or workflow.",
      "Guide me through a Docyrus CLI or API task."
    ],
    "brandColor": "#2563EB"
  }
}
```

- [ ] **Step 5: Add plugin notes with the CLI prerequisite**

Write this Markdown to `/Users/anilbeyazoglu/plugins/docyrus/PLUGIN_NOTES.md`:

~~~markdown
# Docyrus Plugin Notes

This plugin bundles Docyrus skills from:

- `https://github.com/docyrus/agent-skills`
- `https://github.com/docyrus/design-skills`

Some skills expect the Docyrus CLI to be installed separately:

```bash
npm install -g @docyrus/docyrus@latest
```
~~~

### Task 3: Validate the plugin

**Files:**
- Test: `/Users/anilbeyazoglu/plugins/docyrus/.codex-plugin/plugin.json`
- Test: `/Users/anilbeyazoglu/.agents/plugins/marketplace.json`

**Interfaces:**
- Consumes: validator `/Users/anilbeyazoglu/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py`
- Produces: a validation result for the finished plugin

- [ ] **Step 1: Run the validator**

Run:

```bash
python3 /Users/anilbeyazoglu/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py /Users/anilbeyazoglu/plugins/docyrus
```

Expected: success output with no validation errors.

- [ ] **Step 2: Inspect the final skill set**

Run:

```bash
find /Users/anilbeyazoglu/plugins/docyrus/skills -mindepth 1 -maxdepth 1 -type d -exec basename {} \; | sort
```

Expected: the list includes the local Docyrus skills plus:

```text
docyrus-crm-like-detail-page-design
docyrus-dashboard-design
docyrus-data-grid-page-design
docyrus-record-detail-form-design
docyrus-ui-react-components
```
