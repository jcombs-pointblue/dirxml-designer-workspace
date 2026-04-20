# User Application / RBPM layout

If the project's `Model/` tree contains a `Provisioning/` subdirectory, the project includes a User Application (also called RBPM — Role-Based Provisioning Module, or more recently Identity Applications / Identity Governance). This is the identity portal: self-service requests, approval workflows, roles, resources, SoD, attestations.

The top-level container is `Model/Provisioning/AppConfig/`. Every object here is stored as a content file (`.entity`, `.formRequest`, `.prd`, etc.) with a paired `.digest` for change detection — ignore the digests when reading.

## High-level tree

```
Model/Provisioning/AppConfig/
├── AppConfig.{appconfig,digest}          Application-wide config root
├── AppDefs/
│   ├── AppDefs.digest
│   └── locale-configuration.{locale,digest}
├── AuthTypes/
│   └── AuthTypes.digest                  Authentication types (name-password, SAML, etc.)
├── UIConfig/
│   ├── UIConfig.digest                   Portal UI config
│   └── NavItems/NavItems.digest          Navigation items shown in the portal
├── DirectoryModel/                       Directory Abstraction Layer (DAL)
│   ├── configuration.{configuration,digest}
│   ├── EntityDefs/                       Entity definitions — user, group, container, lookup entities
│   ├── RelationshipDefs/                 Entity relationships (e.g. user2groups)
│   ├── QueryDefs/                        Named LDAP queries
│   └── ChoiceDefs/                       Choice/lookup lists
├── RequestDefs/*.prd                     Provisioning Request Definitions (workflows)
├── RoleConfig/
│   ├── RoleConfig.{roleconfig,digest}
│   ├── RoleDefs/                         Role definitions
│   ├── ResourceDefs/                     Resource definitions (map to entitlements)
│   ├── ResourceAssociations/             Role ↔ Resource bindings
│   ├── ResourceRequests/                 PRD wrappers for resource requests
│   ├── CprsRequests/                     Custom provisioning request wrappers
│   ├── SoDDefs/                          Separation-of-Duties constraints
│   ├── Attestations/                     Attestation campaign definitions
│   └── ReportDefs/                       Built-in report definitions
├── TeamDefs/                             Team and delegation definitions
└── WorkflowForms/
    ├── WorkflowRequestForms/*.formRequest       Forms shown to the requester
    ├── WorkflowApprovalForms/*.formApproval     Forms shown to approvers
    └── WorkflowTemplateForms/*.formTemplate     Templates reused across workflows
```

## Key object types

### Provisioning Request Definition (`*.prd`)

The core workflow object. A PRD defines a request with:

- **Initiator / trigger** — typically a user requesting something
- **Form** — the request form (one of the `*.formRequest` files)
- **Pre-activities** — token mappings, external calls
- **Activity flow** — a graph of Start → Approval(s) → Provisioning steps → End
- **Approvers** — users, managers, roles, groups, or JavaScript-computed expressions
- **Timeouts / escalations**
- **Post-activities** — things to do after approval (grant role, provision resource, notify)

PRDs reference approval forms (`*.formApproval`) and frequently call into ECMAScript functions. When describing a workflow, a quick summary is: read the PRD's activity list, identify the approval gates and their approver expressions, note whether it terminates by granting a role, assigning a resource, or running a script activity.

Templates in `AppConfig/RequestDefs/` prefixed `Template*ParallelApproval*` / `TemplateSingleApproval*` / `TemplateNoApproval*` are the out-of-the-box starting points Designer ships; customer-specific PRDs are usually named for their business purpose.

### Forms (`*.formRequest`, `*.formApproval`, `*.formTemplate`)

XML definitions of request/approval forms — fields, data bindings, display labels (often locale-keyed), validation scripts. When describing a workflow, the request form tells you what data the user must supply.

### Role definition (RoleConfig/RoleDefs)

Roles have a level (Business / IT / Permission), expiration policy, category membership (from `ChoiceDefs/roles-category.choice`), SoD constraints, and a set of associated resources (in `ResourceAssociations/`).

### Resource definition (RoleConfig/ResourceDefs)

A Resource maps a user-facing grant onto one or more **driver entitlements**. The ResourceDef carries the entitlement DN (pointing at an `.Entitlement_` in a driver), any required parameters, approval workflow override, revocation policy.

When the user asks "how does the portal grant X?" the trace is usually:

```
User request → PRD → Approvals → Grant Resource → Resource → Entitlement(s) on driver(s) → Driver policies honor entitlement → eDir attribute change → Driver subscriber channel → Connected system change
```

### SoD (RoleConfig/SoDDefs)

Separation-of-duties constraints. Two-role conflict rules with optional approval-override workflow.

### Attestations (RoleConfig/Attestations)

Attestation campaigns — periodic reviews: user profile, role assignment, SoD violations, resource assignment. Each defines scope, reviewers, frequency, escalation.

### DirectoryModel (DAL)

The **Directory Abstraction Layer** — the portal's object model:

- `EntityDefs/*.entity` — e.g. `user.entity`, `group.entity`, `container.entity`, `FullName.entity` (a compound type), `user-lookup.entity`, `quick-user-info.entity`
- `RelationshipDefs/*.relation` — e.g. `user2groups`, `user2users`, `group2users`
- `QueryDefs/*.query` — named LDAP queries the portal can run
- `ChoiceDefs/*.choice` — dropdown lists (role categories, resource categories, application notifications, delegatee relationships)

Entity definitions enumerate attributes, their display labels (often via locale refs), read/write/search/required flags, and validation. Adding a new field to a user form usually means editing an `.entity` plus the request form.

### TeamDefs

Team definitions for the delegation and team-manager features — who manages whom, for which PRDs.

## How Provisioning connects to the driver side

Two explicit link points:

1. **User Application Driver** — there's almost always a driver named `User Application Driver` in the DriverSet (`EQJ4FM7U.Driver_` in the `ig-idm` example). That driver is what the portal uses to talk to the Identity Vault. Its policies are almost entirely package-sourced; customizations here are rare.
2. **Resources map to Entitlements** — each `.ResourceDef_` in `ResourceDefs/` names one or more Entitlement DNs. Those DNs dereference to `.Entitlement_` files under specific drivers. To trace a portal-facing "grant X" end-to-end, follow the resource's entitlement DN into the appropriate driver.

## Describing UserApp content quickly

When the user asks about the portal:

- "What workflows exist?" — list `RequestDefs/*.prd` names; group the `Template*` ones separately from custom PRDs.
- "How is X approved?" — open the PRD, walk the activity flow, note approver expressions.
- "What roles/resources exist?" — list `RoleConfig/RoleDefs/` and `RoleConfig/ResourceDefs/`.
- "What SoD constraints?" — list `RoleConfig/SoDDefs/` and summarize their conflict pairs.
- "What does the user profile page show?" — read `DirectoryModel/EntityDefs/user.entity` for the attributes and `UIConfig/` for the page layout.

## A note on the Provisioning side being Designer-friendly but XML-gnarly

The `.prd` and `.form*` files are verbose, deeply nested, and heavy on locale keys. Expect labels like `$NAMEDTEXT$some.key` that Designer resolves from a locale bundle. When quoting, substitute the resolved label from the locale file or just call out the key — the customer will recognize their own terminology.
