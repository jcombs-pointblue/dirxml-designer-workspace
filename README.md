# dirxml-designer-workspace

A Claude skill for reading and describing **NetIQ / OpenText Identity Manager Designer workspaces** on disk — DirXML drivers, policies, filters, entitlements, packages, Identity Vault schema, and User Application / RBPM provisioning.

It spares Claude from re-deriving the Designer file layout, object model, and policy grammar from first principles every session, so you can point it at a workspace and immediately ask things like *"what does this driver do?"*, *"summarize the Subscriber policies"*, *"which entitlements does it expose?"*

## What's in here

```
dirxml-designer-workspace/
├── SKILL.md                                      Entry point — loaded when the skill triggers
├── references/
│   ├── workspace-layout.md                       Directory map + every known .Type_ extension
│   ├── object-types.md                           Per-type details (Driver_, Subscriber_, Filter_, …)
│   ├── policies-and-rules.md                     DirXML Script grammar, XSLT, mapping, filter
│   ├── provisioning-userapp.md                   RBPM: PRDs, forms, roles, resources, DAL, SoD
│   └── identity-vault-and-packages.md            Schema, packages, GCVs, ECVs, named passwords
└── examples/
    └── cyberark-driver-walkthrough.md            One real driver traced end-to-end
```

## Scope

Read-and-describe only. The skill will help you document, summarize, or answer questions about a workspace. It does **not** modify Designer files — non-trivial edits really want to be done in Designer itself so IDs, BackReferences, and the Modeler graph stay consistent.

## Using the skill

Drop `dirxml-designer-workspace/` into a skills folder that your Claude surface loads from — Claude Code's `.claude/skills/`, a Cowork plugin `skills/` directory, or wherever your setup expects skill bundles. The skill auto-triggers on phrases like *"the CyberArk driver"*, *"our AD sync policies"*, paths under `Model/EdirOrphan/`, or filenames ending in `.Driver_` / `.ScriptPolicy_` / `.MappingPolicy_`.

## License

MIT — see `LICENSE`.
