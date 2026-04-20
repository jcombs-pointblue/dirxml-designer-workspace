# dirxml-designer-workspace

A Claude plugin that lets Claude read and describe **NetIQ / OpenText Identity Manager Designer workspaces** on disk — DirXML drivers, policies, filters, entitlements, packages, Identity Vault schema, and User Application / RBPM provisioning.

It spares Claude from re-deriving the Designer file layout, object model, and policy grammar from first principles every session, so you can point it at a workspace and immediately ask things like *"what does this driver do?"*, *"summarize the Subscriber policies"*, *"which entitlements does it expose?"*

## Install

### Claude Cowork mode

Download the latest `.plugin` file from [Releases](https://github.com/jcombs-pointblue/dirxml-designer-workspace/releases) and drag it into Cowork. Or install from this repo directly — Cowork recognizes a plugin by its `.claude-plugin/plugin.json`.

### Claude Code (CLI)

Install as a plugin with the `/plugin` command inside a `claude` session, pointing at this repo.

Or, to use just the skill without the plugin wrapper, clone and symlink (or copy) the inner skill directory:

```bash
git clone https://github.com/jcombs-pointblue/dirxml-designer-workspace.git /tmp/dxw
mkdir -p ~/.claude/skills
ln -s /tmp/dxw/skills/dirxml-designer-workspace ~/.claude/skills/dirxml-designer-workspace
```

Restart your `claude` session and ask it about a Designer workspace.

## What's in here

```
dirxml-designer-workspace/
├── .claude-plugin/
│   └── plugin.json                                 Plugin manifest
└── skills/
    └── dirxml-designer-workspace/
        ├── SKILL.md                                Entry point — loads when the skill triggers
        ├── references/
        │   ├── workspace-layout.md                 Directory map + every known .Type_ extension
        │   ├── object-types.md                     Per-type details (Driver_, Subscriber_, Filter_, …)
        │   ├── policies-and-rules.md               DirXML Script grammar, XSLT, mapping, filter
        │   ├── provisioning-userapp.md             RBPM: PRDs, forms, roles, resources, DAL, SoD
        │   └── identity-vault-and-packages.md      Schema, packages, GCVs, ECVs, named passwords
        └── examples/
            └── cyberark-driver-walkthrough.md      One real driver traced end-to-end
```

## Scope

Read-and-describe only. The skill will help you document, summarize, or answer questions about a workspace. It does **not** modify Designer files — non-trivial edits really want to be done in Designer itself so IDs, BackReferences, and the Modeler graph stay consistent.

## Triggers

The skill auto-triggers on phrases like *"the CyberArk driver"*, *"our AD sync policies"*, *"this IDM workspace"*, *"a PRD"*, paths under `Model/EdirOrphan/` or `Model/Provisioning/`, or filenames ending in `.Driver_` / `.ScriptPolicy_` / `.MappingPolicy_` / `.Filter_` / `.IdmPackage_`.

## License

MIT — see [`LICENSE`](LICENSE).
