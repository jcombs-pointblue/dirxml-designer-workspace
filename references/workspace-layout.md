# Designer workspace layout

A Designer workspace is an Eclipse workspace (`.metadata/`) containing one or more IDM projects. Each project is its own directory, with a `.project` file declaring nature `com.novell.idm.DesignerProjectNature`.

```
<workspace_root>/
├── .metadata/                     Eclipse bookkeeping — IGNORE for IDM content
├── <project-A>/
│   ├── .project                   Eclipse project descriptor (IDM nature)
│   ├── .tmp/                      Transient — IGNORE
│   ├── Model/
│   │   ├── IdentityManager/       Modeler view: Domain, IdentityVault, Applications, Schema
│   │   ├── EdirOrphan/            *** Driver source of truth: DriverSets, Drivers, policies ***
│   │   ├── Provisioning/          User App / RBPM model (if the project has UserApp)
│   │   └── Project/               Catalog + TreeData for the project
│   └── Designer/
│       ├── Toolbox/               DocumentGenerator, Simulator state
│       └── Documents/Generated/   Generated documentation output (HTML, etc.)
└── <project-B>/                   Same structure; a workspace can hold several
```

## Model/IdentityManager — the Modeler view

This is what you see in Designer's Modeler tab: a tree of Domain → IdentityVault → Servers and Applications (app icons connected to drivers).

```
Model/IdentityManager/
├── <DomainID>.Domain_             Top of the tree
├── <DomainID>/
│   ├── <IdentityVaultID>.IdentityVault_
│   ├── <IdentityVaultID>/
│   │   ├── <SchemaDefID>.SchemaDef_
│   │   └── <SchemaDefID>_schema.xml        eDirectory schema extensions
│   ├── <AppID>.Application_                One per app icon in the Modeler
│   ├── <AppID>_icon.{png,gif}              App icon bitmap
│   └── ...
└── <ModelerID>.ModelerNodes_               Node positions/layout for the Modeler canvas
```

Application objects are just Modeler-layer icons. Their `relations` back-reference the actual Driver under `EdirOrphan/` (`Idm:Application` relation on the Driver, `Idm:Drivers` relation on the Application).

## Model/EdirOrphan — where drivers actually live

"Orphan" means not currently associated with a live eDirectory replica in the Modeler, but this is the normal on-disk home of driver configuration. A workspace almost always has one DriverSet directory here per deployment.

```
Model/EdirOrphan/
├── <ServerID>.Server_                       The logical IDM engine/server
├── <DriverSetID>.DriverSet_                 DriverSet metadata (JVM params, log events, named passwords)
├── <DriverSetID>/                           DriverSet contents
│   ├── <DriverID>.Driver_                   One per driver — shim config, trace, auth
│   ├── <DriverID>/                          Driver contents (see below)
│   ├── <DriverID>_icon.{gif,png}            Driver icon
│   ├── <DriverID>_<ServerID>_DirXML-ConfigValues.xml   Per-server GCV overrides
│   ├── <LibraryID>.Library_                 Shared policies / resources at DriverSet scope
│   ├── <LibraryID>/                         Library contents (ECMAScript resources, etc.)
│   ├── <GlobalConfigID>.GlobalConfig_       GCV definitions (DriverSet-level)
│   ├── <GlobalConfigID>_initial_state.xml   Initial GCV values
│   ├── <GlobalConfigID>_<ServerID>_DirXML-ConfigValues.xml
│   └── <JobID>.Job_ + <JobID>_contents.xml  Scheduled jobs
└── <NotfCollectionID>.NotfTemplateCollection_
    <NotfCollectionID>/*.NotfTemplate_       Notification templates (email)
```

A Driver directory looks like:

```
<DriverID>/
├── <SubscriberID>.Subscriber_               Subscriber channel (eDir → app)
├── <SubscriberID>/*.ScriptPolicy_ + _contents.xml
├── <SubscriberID>/*.StylesheetPolicy_ + _contents.xml
├── <PublisherID>.Publisher_                 Publisher channel (app → eDir)
├── <PublisherID>/*.ScriptPolicy_ + _contents.xml
├── <FilterID>.Filter_ + _contents.xml       The driver's class/attribute filter
├── <MappingPolicyID>.MappingPolicy_ + _contents.xml   Schema mapping (nds ↔ app names)
├── <ECMAScriptID>.ECMAScriptResource_ + _contents.xml  Driver-scoped JS helpers
├── <EntitlementID>.Entitlement_ + _contents.xml        Each entitlement the driver grants
└── <PolicyID>.ScriptPolicy_ + _contents.xml            Driver-level policies (e.g. Input Transform)
```

Note: the same policy file can appear both as a `Child` under the Driver and as a `Reference` from the Subscriber/Publisher — that's normal. Policies are stored once; channels reference them in execution order.

## Model/Provisioning — User Application / RBPM

Only present in projects that include a User Application (identity portal, request/approval workflows, roles, resources). See `provisioning-userapp.md` for the full breakdown; at a glance:

```
Model/Provisioning/
└── AppConfig/
    ├── AppConfig.digest
    ├── AppDefs/                             Application-wide definitions + locale config
    ├── AuthTypes/                           Authentication type configuration
    ├── UIConfig/                            Portal UI, nav items
    ├── DirectoryModel/                      Directory Abstraction Layer (DAL)
    │   ├── EntityDefs/*.entity              Entity definitions (user, group, container, ...)
    │   ├── RelationshipDefs/*.relation
    │   ├── QueryDefs/*.query                Named LDAP queries
    │   └── ChoiceDefs/*.choice              Lookup/choice lists
    ├── RequestDefs/*.prd                    Provisioning Request Definitions (workflows)
    ├── RoleConfig/
    │   ├── RoleDefs/                        Role definitions
    │   ├── ResourceDefs/                    Resource definitions
    │   ├── ResourceAssociations/            Role ↔ Resource bindings
    │   ├── ResourceRequests/                PRD wrappers for resource requests
    │   ├── CprsRequests/                    CPRS-based request definitions
    │   ├── SoDDefs/                         Separation-of-Duties constraints
    │   ├── Attestations/                    Attestation campaigns
    │   └── ReportDefs/                      Report definitions
    ├── TeamDefs/                            Team / delegation definitions
    └── WorkflowForms/
        ├── WorkflowRequestForms/*.formRequest
        ├── WorkflowApprovalForms/*.formApproval
        └── WorkflowTemplateForms/*.formTemplate
```

Every `.<ext>` here has a matching `.digest` sibling — Designer's change-detection hash. Ignore `.digest` files when reading content.

## Model/Project

```
Model/Project/<ProjectID>/
├── <ProjectID>.ProjectData_                 Minimal root
├── TreeData.CObject_                        Persisted tree state
└── <IdmCatalogID>.IdmCatalog_               Package catalog
    └── <IdmCatalogID>/<CategoryID>.IdmCategory_   Package categories + folders
```

This is Designer-internal bookkeeping. Useful only if you're answering "what package catalog does this project use?"

## Designer/

```
Designer/
├── Toolbox/
│   ├── DocumentGenerator/                   Doc Gen templates + state
│   └── Simulator/                           Policy Simulator state
└── Documents/Generated/                     Outputs from the Document Generator
```

If the user asks "has someone generated docs?" — check `Documents/Generated/`. If it's empty, they haven't.

## File-type extensions you will see

Metadata files (all are `com.novell.designer.model:CObject` XML):

| Extension | Object |
| --- | --- |
| `.Domain_` | Modeler domain root |
| `.IdentityVault_` | An Identity Vault (eDirectory tree) |
| `.Server_` | An IDM engine/server |
| `.Application_` | A Modeler application icon |
| `.DriverSet_` | A DriverSet |
| `.Driver_` | A single driver |
| `.Subscriber_` / `.Publisher_` | Channels on a driver |
| `.Filter_` | The driver's sync filter |
| `.ScriptPolicy_` | A DirXML Script policy (most common policy type) |
| `.StylesheetPolicy_` | An XSLT stylesheet policy |
| `.MappingPolicy_` | A schema mapping policy |
| `.MappingTableResource_` | A mapping-table resource (used via `Map` tokens) |
| `.Entitlement_` | A driver entitlement definition |
| `.ECMAScriptResource_` | Shared JavaScript functions (Rhino) |
| `.GlobalConfig_` | A GCV definition bundle |
| `.Library_` | A policy library at DriverSet scope |
| `.Job_` | A scheduled job |
| `.NotfTemplate_` / `.NotfTemplateCollection_` | Email notification templates |
| `.SchemaDef_` | Identity Vault schema extension definition |
| `.IdmPackage_` / `.IdmPackageFolder_` | Package and package folder metadata |
| `.IDMResource_` | Miscellaneous driver resource |
| `.IdmCatalog_` / `.IdmCategory_` / `.IdmCategoryFolder_` | Package catalog |
| `.ModelerNodes_` | Modeler canvas layout |
| `.ProjectData_` / `.CObject_` | Project-level bookkeeping |

Content / non-metadata files:

| Pattern | Holds |
| --- | --- |
| `<ID>_contents.xml` | Payload for the paired `<ID>.<Type>_` (policy body, filter, mapping, entitlement, stylesheet, JS source) |
| `<SchemaDefID>_schema.xml` | eDirectory schema extensions |
| `<DriverID>_<ServerID>_DirXML-ConfigValues.xml` | Per-server GCV override bindings |
| `<GCVID>_initial_state.xml` | Initial GCV values for a GlobalConfig |
| `*.prd` | User App Provisioning Request Definition (workflow) |
| `*.entity`, `*.choice`, `*.query`, `*.relation` | DAL definitions |
| `*.formRequest`, `*.formApproval`, `*.formTemplate` | Workflow forms |
| `*.roleconfig`, `*.role20`, `*.attestation`, `*.rbpmTeam`, `*.resources`, `*.provisioning`, `*.appconfig`, `*.configuration` | UserApp / Provisioning section configs |
| `*.digest` | Change-detection hash — IGNORE for content, useful only to detect dirty objects |
| `*.gif`, `*.png` | Icons |

## Working rule of thumb

When the user points at an object and you need the content, remember the pairing:

- `<ID>.<Type>_` — metadata (small, relations-heavy)
- `<ID>_contents.xml` — real payload (if the object has one)
- `<ID>/` — child directory (if the object has children)

Read whichever of the three you actually need. For policies, it's almost always just the `_contents.xml`.
