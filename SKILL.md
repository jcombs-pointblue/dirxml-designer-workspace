---
name: dirxml-designer-workspace
description: Read and describe NetIQ / OpenText Identity Manager Designer workspaces on disk — DirXML drivers, policies (DirXML Script, XSLT, mapping), filters, entitlements, packages, Identity Vault schema, User Application / RBPM provisioning (workflows, forms, roles, resources, PRDs). Use this skill whenever the user references a Designer workspace, a `designer_workspace*` folder, a `.Driver_` / `.ScriptPolicy_` / `.MappingPolicy_` / `.Filter_` / `.IdmPackage_` file, or asks about DirXML policies, IDM drivers, driver sets, GCVs, Subscriber/Publisher channels, eDirectory sync, User App workflows, role/resource provisioning, or IDM documentation. Trigger even if the user doesn't name the product — phrases like "the CyberArk driver", "our AD sync policies", "the IDM workspace", "a PRD", or a path under `Model/EdirOrphan/` or `Model/Provisioning/` mean this skill applies.
---

# NetIQ / OpenText IDM Designer Workspace

Use this skill when you need to understand, describe, document, or answer questions about a Designer workspace on disk. It spares you from re-deriving the file layout and object model from first principles each session.

## What a Designer workspace is

A Designer workspace is an **Eclipse workspace** (it contains a standard `.metadata/` directory) that holds one or more **IDM projects**. Each project is a Designer model of an Identity Manager deployment — DriverSets, Drivers, policies, Identity Vault schema, and optionally the User Application / RBPM provisioning model.

Designer stores every model object as **two files that share a short uppercase ID**:

- `<ID>.<Type>_` — metadata (XML, `com.novell.designer.model:CObject` root) with scalar attributes and `relations` to other objects
- `<ID>_contents.xml` — the actual payload (DirXML Script policy, XSLT, mapping table, filter XML, entitlement XML, ECMAScript, etc.)

Objects with children (Drivers, Subscribers, Publishers, DriverSets) also have a **sibling directory named with the same ID** that contains the child objects. So to walk the tree you chase both the `relations` keys and the matching directory.

## Orientation: the minimum you must know to work in one of these

Open the workspace and get your bearings in this order:

1. **Identify the project roots.** Every child directory of the workspace with a `.project` file whose nature is `com.novell.idm.DesignerProjectNature` is an IDM project. Ignore `.metadata/` (Eclipse bookkeeping) and `.tmp/`.
2. **Inside a project, the interesting trees are:**
   - `Model/IdentityManager/<DomainID>/` — the Modeler view: Domain, IdentityVault, Servers, Applications (app icons connecting to drivers). Schemas live under `<IdentityVaultID>/`.
   - `Model/EdirOrphan/` — **where DriverSets, Servers, Drivers, Libraries, and their policies actually live.** "Orphan" means not currently bound to a live eDirectory replica in the Modeler, but this is the normal storage location for the driver configuration source of truth.
   - `Model/Provisioning/AppConfig/` — User Application / RBPM: workflows (RequestDefs → `*.prd`), forms, roles, resources, SoD, attestations, directory abstraction layer.
   - `Model/Project/<ProjectID>/` — Catalog (package categories) and TreeData.
   - `Designer/Documents/Generated/` — Document Generator output, if the user has run it.
3. **The driver tree is a hub-and-spoke graph, not a simple nesting.** A `.Driver_` metadata file declares `relations` like `Idm:Subscriber`, `Idm:Publisher`, `Idm:Filter`, `Idm:Entitlements`, `Idm:Policies` pointing at sibling objects. Subscriber/Publisher then declare `Idm:EventPolicies`, `Idm:MatchingPolicies`, `Idm:CreatePolicies`, `Idm:CommandPolicies`, `Idm:InputPolicies`, `Idm:OutputPolicies`, `Idm:SchemaMappingPolicies`, `Idm:TransformPolicies` — each of these is an ordered policy set.
4. **Packages leave fingerprints.** Almost every driver is built from one or more `IdmPackage_` references — when you see `relations name="Idm:InstalledPackages"` on a DriverSet or Driver, that's tracing the package provenance.

## The canonical lookup: given X, how do I find it?

| You have | Where to look |
| --- | --- |
| Driver name (e.g. "CyberArk") | Grep `Model/EdirOrphan/*/*.Driver_` for `name="CyberArk"` — the file ID is the driver ID |
| Driver ID (e.g. `9HPNAS5Y`) | Metadata is `Model/EdirOrphan/<DriverSetID>/<ID>.Driver_`; children are the sibling directory `.../<ID>/` |
| Policy by name | Grep `*.ScriptPolicy_` / `*.StylesheetPolicy_` / `*.MappingPolicy_` for `name="..."` — the real content is the adjacent `_contents.xml` |
| The policies *attached* to a channel in execution order | Read the `.Subscriber_` / `.Publisher_` metadata file — policy-set relations (`Idm:EventPolicies`, etc.) are in document order, which is the order Designer runs them |
| A filter's classes/attrs | `<ID>_contents.xml` with root `<filter>` (sibling of `<ID>.Filter_`) |
| GCVs (global config values) | At DriverSet scope: `*.GlobalConfig_` under `Model/EdirOrphan/<DriverSetID>/`. At Driver scope: driver-embedded `DirXML-ConfigValues` CDATA in the `.Driver_` file. Per-server overrides: `<DriverID>_<ServerID>_DirXML-ConfigValues.xml` |
| eDirectory schema extensions | `Model/IdentityManager/<DomainID>/<IdentityVaultID>/<SchemaDefID>_schema.xml` |
| User App workflows (PRDs) | `Model/Provisioning/AppConfig/RequestDefs/*.prd` |
| Roles / Resources / SoD / Attestations | `Model/Provisioning/AppConfig/RoleConfig/{RoleDefs,ResourceDefs,SoDDefs,Attestations}/` |
| Workflow forms | `Model/Provisioning/AppConfig/WorkflowForms/{WorkflowRequestForms,WorkflowApprovalForms,WorkflowTemplateForms}/` |
| Directory Abstraction Layer (entities, queries, relationships, choice lists) | `Model/Provisioning/AppConfig/DirectoryModel/` |

## Reading policy content

The `.<Type>_` file is thin metadata and relations — the interesting stuff is almost always in `<ID>_contents.xml`. Three content dialects dominate:

- **DirXML Script** — root `<policy>`, inside are `<rule>` → `<description>` / `<conditions>` / `<actions>`. Conditions use `<if-*>` (e.g. `if-class-name`, `if-operation`, `if-global-variable`, `if-xpath`). Actions use `<do-*>` (`do-veto`, `do-set-dest-attr-value`, `do-status`, `do-add-association`). Tokens like `<token-attr>`, `<token-xpath>`, `<token-src-dn>` appear inside arg-* elements. This is the primary dialect — most ScriptPolicy files are DirXML Script.
- **XSLT stylesheet** — root `<xsl:stylesheet>`. These are `.StylesheetPolicy_` and run via the XSLT engine; they receive the XDS event document and emit transformed XML. Watch for the magic `<xsl:param>` names (`srcQueryProcessor`, `destQueryProcessor`, `dnConverter`) that Designer/IDM injects.
- **Mapping policy** — root `<attr-name-map>` with `<class-name>` and `<attr-name>` children mapping between `<nds-name>` (eDirectory) and `<app-name>` (connected system).

Filters are their own dialect: root `<filter>`, with `<filter-class>` (per eDirectory class) and `<filter-attr>` (per attribute) carrying `publisher=`, `subscriber=`, `merge-authority=` attributes.

## Describing a driver: the recipe

When the user asks "what does the X driver do?" or "document this driver":

1. Find the `.Driver_` metadata file by name.
2. From it, collect: shim (`DirXML-JavaModule`), target app (`DirXML-ShimAuthServer` etc.), driver-level config (`DirXML-ShimConfigInfo` CDATA — a nested `<driver-config>` with driver-options / subscriber-options / publisher-options), trace settings.
3. List the policies attached to each channel (read `.Subscriber_` and `.Publisher_` and walk their relations in order). For each, read its `_contents.xml` and summarize the rules — a DirXML Script rule's `<description>` is almost always a real sentence written by the developer and is the best single source of summary text.
4. Read the Filter to describe what classes/attributes actually flow and in which direction.
5. Read the Schema Mapping (MappingPolicy) to show the eDir↔app attribute naming.
6. Read any Entitlement files to list what the driver can grant.
7. Check `DirXML-ConfigValues` CDATA and separate `*_DirXML-ConfigValues.xml` for GCV values that parameterize the policies.
8. Mention package provenance: `Idm:InstalledPackages` relations tell you which `IdmPackage_` objects contributed.

## What to ignore

- `.metadata/` anywhere — Eclipse plugin state; irrelevant to IDM content.
- `.tmp/` — transient.
- `.DS_Store`, `.lock`, `_icon.gif`, `_icon.png` — cosmetic/housekeeping.
- History under `.metadata/.plugins/org.eclipse.core.resources/.history/` — Eclipse's local file history cache; not the IDM source of truth.

## Reference files (read on demand)

- `references/workspace-layout.md` — full directory map, every known type extension, and what each top-level tree is for.
- `references/object-types.md` — every `.<Type>_` file in this ecosystem, what it represents, and what its `_contents.xml` holds (if any).
- `references/policies-and-rules.md` — DirXML Script grammar, XSLT conventions, mapping/filter payloads, and how policy sets attach to channels.
- `references/provisioning-userapp.md` — UserApp / RBPM layout: RequestDefs, forms, roles, resources, SoD, attestations, DAL.
- `references/identity-vault-and-packages.md` — Identity Vault schema, packages, global configs, server bindings, GCV override files.
- `examples/cyberark-driver-walkthrough.md` — a fully traced example driver from the `ig-idm` project showing how the metadata, relations, policies, filter, schema map, and entitlements connect. Read this first when learning the model; then generalize.

## Tone and format for output

The user is an IDM consultant, so use IDM vocabulary directly (subscriber/publisher channel, GCV, NDS vs app name, merge authority, association, entitlement). Prefer concise prose; reach for bullet lists only when enumerating policies, rules, filter attrs, or similar. Quote short rule descriptions from the XML — they usually explain intent better than any paraphrase.

Modification is out of scope for this skill (it's read-and-describe only). If the user asks to edit, flag that editing Designer files byte-for-byte is viable for small script/filter tweaks, but non-trivial changes (adding policies, renaming drivers, rewiring relations) really want to be done in Designer itself because IDs, BackReferences, and the Modeler graph all have to stay consistent.
