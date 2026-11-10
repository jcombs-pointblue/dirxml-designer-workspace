# dirxml-designer-workspace

A Claude plugin that lets Claude read and describe **NetIQ / OpenText Identity Manager Designer workspaces** on disk — DirXML drivers, policies, filters, entitlements, packages, Identity Vault schema, and User Application / RBPM provisioning.

It spares Claude from re-deriving the Designer file layout, object model, and policy grammar from first principles every session, so you can point it at a workspace and immediately ask things like *"what does this driver do?"*, *"summarize the Subscriber policies"*, *"which entitlements does it expose?"*

## The headline use case: semantic comparison of driver or project versions

Text-based `diff` on a Designer workspace is nearly useless. Object IDs, relation orderings, and timestamps churn between exports, so a diff buries the real changes under noise. Dropping two workspaces (or two drivers) in front of an agent that understands the object model inverts that: instead of *"line 47 changed,"* you get *"the Subscriber Event Transformation in dev has an extra rule that vetoes User modifies when the IG sync mode GCV is enabled,"* *"the prod filter still notifies on `DirXML-EntitlementRef` but dev was switched to sync,"* or *"the `SafePermission` entitlement's query was changed from multi-value union to multi-value merge."*

This is the kind of comparison IDM consultants do constantly — non-prod vs prod before a promotion, two customer tenants looking for configuration drift, a pre-upgrade snapshot against the post-upgrade workspace, a package version compared to what's actually deployed. Previously that meant eyeballing Designer side-by-side for hours. An agent with this skill can do it in one pass and tell you what matters.

When you need the comparison to be exact and repeatable, not just readable, see [Comparing, validating or changing a project](#comparing-validating-or-changing-a-project-dirxmldev) below.

## Install

### Claude Cowork mode

Download the latest `.plugin` file from [Releases](https://github.com/jcombs-pointblue/dirxml-designer-workspace/releases) and drag it into Cowork. Or install from this repo directly — Cowork recognizes a plugin by its `.claude-plugin/plugin.json`.

### Claude Code (CLI)

The repo ships a marketplace manifest (`.claude-plugin/marketplace.json`), so you can add it as a plugin marketplace and install from there inside a `claude` session:

```
/plugin marketplace add jcombs-pointblue/dirxml-designer-workspace
/plugin install dirxml-designer-workspace@pointblue
```

Or install the plugin directly from the repo with the `/plugin` command.

To use just the skill without the plugin wrapper, clone and symlink (or copy) the inner skill directory:

```bash
git clone https://github.com/jcombs-pointblue/dirxml-designer-workspace.git /tmp/dxw
mkdir -p ~/.claude/skills
ln -s /tmp/dxw/skills/dirxml-designer-workspace ~/.claude/skills/dirxml-designer-workspace
```

Restart your `claude` session and ask it about a Designer workspace.

### Other LLMs (GPT, Gemini, local models, anything with a file-reading agent)

The bare skill is just Markdown — `skills/dirxml-designer-workspace/SKILL.md` plus the `references/` and `examples/` files it points at. Any model with access to read files can use it: point your agent framework at that directory, load `SKILL.md` into the system prompt or as retrievable context, and let the model open the reference files on demand. There's nothing Claude-specific in the content itself; the frontmatter is ignored by non-Claude runtimes.

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

## Comparing, validating or changing a project: DirXMLDev

This skill reads and explains. For work that has to be exact, or that changes the project, use [DirXMLDev](https://github.com/PointBlueTechnology/DirXMLDev). It's an open-source command-line tool (MIT) that turns a Designer project into plain files, an IDM-as-code tree, and works on those:

- **Compare two projects.** Import each one, then diff the trees. The diff lists every difference driver by driver, and gives the same answer every time:
  ```bash
  bin/idm import-project ~/designer_workspace/ProjectQA qa
  bin/idm import-project ~/designer_workspace/ProjectPROD prod
  bin/idm tree.diff prod qa
  ```
- **Validate.** `bin/idm validate qa` checks links, GCVs and mapping tables, and compiles each policy with the engine's own compilers. That last check needs the engine jars from your Identity Manager installation.
- **Change it.** Edit the tree, then deploy it to an Identity Vault with `bin/idm vault.deploy`, or write the change back into the Designer project with `bin/idm export-project qa ~/designer_workspace/ProjectQA`. `export-project` writes only the files the change needs. It handles drivers that weren't installed from packages; for a packaged driver, deploy to a vault and use Designer's *Import from the Identity Vault*.

The two work well together. Let the agent use this skill to explain what a driver does, and DirXMLDev for the diff, the checks and the edits. DirXMLDev needs JDK 21; see its [install guide](https://github.com/PointBlueTechnology/DirXMLDev/blob/master/docs/install.md). [DirXMLDevWeb](https://github.com/PointBlueTechnology/DirXMLDevWeb) puts the same tools in a browser.

## Triggers

The skill auto-triggers on phrases like *"the CyberArk driver"*, *"our AD sync policies"*, *"this IDM workspace"*, *"a PRD"*, paths under `Model/EdirOrphan/` or `Model/Provisioning/`, or filenames ending in `.Driver_` / `.ScriptPolicy_` / `.MappingPolicy_` / `.Filter_` / `.IdmPackage_`.

## License

MIT — see [`LICENSE`](LICENSE).
