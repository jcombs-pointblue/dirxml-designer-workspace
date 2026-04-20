# Designer object types

Every Designer object on disk is an XML file with root `<com.novell.designer.model:CObject>`. The root carries `name=` (the display name — the *thing the user sees in Designer*) and `type=` (the Designer type class). Inside are two kinds of children:

- `<attributes xsi:type="com.novell.designer.model:C<Kind>" attrName="..." value="..."/>` — scalar attributes, where `<Kind>` is `String`, `Integer`, `Boolean`, `Structure`, `HeavyData`, etc. `CStructure` nests further attributes; `CHeavyData` means "the real content is in the paired `_contents.xml`".
- `<relations name="..." type="Child|Reference|BackReference" key="#<ID>.<Type>_"/>` — links to other CObjects. `Child` = owned; `Reference` = cited; `BackReference` = the inverse pointer maintained for the other side.
- `<associatedAttrSets objectURI="...">` — attributes whose scope is tied to a specific Server (e.g. per-server driver auth credentials). The `objectURI` points at a `.Server_` file.

The `name` attribute on the root is **the human name**. Don't make up a name from the file ID — always read `name=`.

## Structural types (hold relations; may have children on disk)

### `.Domain_`

The top of the Modeler tree for one deployment. Usually one per project.

### `.IdentityVault_`

An Identity Vault (eDirectory tree). Holds `SchemaDef_` children. Relates to the DriverSet (`Idm:DriverSets`).

### `.Server_`

A single IDM engine/server. Associated with a DriverSet. Per-server driver parameters (auth passwords, trace file paths) are carried on drivers as `associatedAttrSets objectURI="...<Server>"`.

### `.Application_`

A Modeler "app icon" — the little rectangle in the Modeler canvas that represents a connected system. Pairs with a Driver via `Idm:Drivers` / `Idm:Application` relations. **Content-wise it's thin metadata**; the real driver config lives on the `.Driver_` side. If the user asks about "the AD app icon" vs "the AD driver," they're usually pointing at the same logical thing and you want the Driver.

### `.DriverSet_`

A DriverSet. Holds `Idm:Drivers`, `Idm:Libraries`, `Idm:GlobalConfigs`, `Idm:Jobs`, `Idm:Servers`, `Idm:InstalledPackages` relations. Carries JVM config, named password metadata, driver-set-level log events.

### `.Driver_`

A single driver. The most information-dense metadata file in the workspace. Typical attributes:

- `name=` — driver name (e.g. `CyberArk`, `Active Directory Driver`)
- `DirXML-JavaModule` — the shim class (e.g. `com.novell.nds.dirxml.driver.ldap.LDAPDriverShim`, `com.novell.gw.driver.ADDriverShim`)
- `DirXML-ShimAuthServer` / `DirXML-ShimAuthID` / `DirXML-ShimAuthPassword` — auth to the connected system
- `DirXML-ConfigManifest` — the driver's config manifest XML
- `DirXML-ShimConfigInfo` — **large CDATA holding the driver-options / subscriber-options / publisher-options GCV definitions** used by the shim. This is the first place to look for "what can this driver be configured to do?"
- `DirXML-TraceFile` / `DirXML-TraceLevel` / `DirXML-TraceName` — tracing
- `DirXML-DriverStartOption` — 0 disabled, 1 manual, 2 auto
- `DirXML-DriverVersion` — package version

Key relations on a Driver:

- `Idm:Subscriber` (Child, singleton) — points at the driver's Subscriber channel
- `Idm:Publisher` (Child, singleton) — points at the Publisher
- `Idm:Filter` (Child, singleton) — points at the Filter
- `Idm:MappingPolicies` (Reference) — schema mapping policy for the driver
- `Idm:Policies` (Child) — every policy file belonging to the driver (referenced by the channels)
- `Idm:InputPolicies` / `Idm:OutputPolicies` — driver-level input/output transform policies
- `Idm:Entitlements` (Child) — every entitlement this driver can grant
- `Idm:ExtensionFunctions` (Reference) — ECMAScript resources available to policies
- `Idm:Application` (BackReference) — the Modeler app icon representing this driver
- `Idm:InstalledPackages` (Reference) — packages installed into the driver

### `.Subscriber_` / `.Publisher_`

Channel containers. Carry **policy-set relations in document order** — that order matters because it is the policy execution order:

- `Idm:EventPolicies` — Event Transformation (runs first on the channel)
- `Idm:MatchingPolicies` — Matching
- `Idm:CreatePolicies` — Create (the object-doesn't-exist branch)
- `Idm:PlacementPolicies` — Placement
- `Idm:CommandPolicies` — Command Transformation (runs last before dispatch)
- `Idm:InputPolicies` / `Idm:OutputPolicies` — Input/Output Transformation on the channel
- `Idm:SchemaMappingPolicies` — schema mapping on this channel
- `Idm:TransformPolicies` — generic transform stage

Also `Idm:Policies` (Child) listing every policy file owned by the channel (whether or not currently bound to a policy set).

### `.Filter_`

The driver's sync filter. The content is in the paired `_contents.xml`.

### `.Library_`

A shared policy library at DriverSet scope. Holds `Idm:Resources` (ECMAScript resources) and `Idm:GlobalConfigs`. Children live in `<LibraryID>/`.

### `.GlobalConfig_`

A GCV bundle. The `.GlobalConfig_` metadata file itself is thin; the GCV definitions are in `DirXML-ConfigValues` (as a `CHeavyData` attribute referencing content) or in the separate `<ID>_initial_state.xml`. Per-server binding of GCV values is in `<GCVID>_<ServerID>_DirXML-ConfigValues.xml`.

### `.Job_`

A scheduled job (e.g. nightly resync). `_contents.xml` holds the job's XDS config.

### `.IdmPackage_` / `.IdmPackageFolder_`

Package metadata — packages installed into the project or DriverSet. The actual package catalog is in `Model/Project/<ProjectID>/<IdmCatalogID>.IdmCatalog_`.

## Content types (have a `_contents.xml` payload)

### `.ScriptPolicy_`

DirXML Script policy. Content: `<policy>` with one or more `<rule>` elements. See `policies-and-rules.md` for the grammar.

The metadata file tells you *what channel stage* it's bound to via BackReference relations (e.g. `Idm:CommandPolicyRefs` pointing at the Subscriber).

### `.StylesheetPolicy_`

XSLT stylesheet. Content: `<xsl:stylesheet>`. Common in legacy drivers and for complex transforms where DirXML Script is awkward. Watch for the IDM-injected params (`srcQueryProcessor`, `destQueryProcessor`, `srcCommandProcessor`, `destCommandProcessor`, `dnConverter`, `fromNds`) — policies often call them via `<xsl:variable>` and extension namespaces like `cmd:`, `query:`, `dncv:`.

### `.MappingPolicy_`

Schema mapping between eDirectory (`<nds-name>`) and the connected application (`<app-name>`). Content is `<attr-name-map>` with `<class-name>` and `<attr-name>` children.

### `.MappingTableResource_`

A mapping-table resource — used by the `Map` token in DirXML Script to do table-driven lookups.

### `.Entitlement_`

Content: `<entitlement>` with `<values>` (and possibly `<query>` for dynamic entitlements). Describes what the driver can grant (a license, a group membership, an AD account, etc.) and whether the values are multi-valued and how conflicts resolve.

### `.ECMAScriptResource_`

Content: JavaScript (Rhino). These are libraries of helper functions that DirXML Script policies call via `<token-xpath>es:myFunction(...)` using the `es:` namespace extension.

### `.NotfTemplate_`

Email notification template — used by the UserApp and some drivers to send email. Content is the MIME-ish template XML.

### `.SchemaDef_`

Identity Vault schema extensions. `_schema.xml` is the NDS-format schema definition.

## Attribute kinds you'll see

| `xsi:type` | Meaning |
| --- | --- |
| `CString` | plain string value |
| `CInteger` | integer |
| `CBoolean` | true/false |
| `CStructure` | holds nested `attributes` — think sub-object |
| `CHeavyData` | the real content is in the paired `_contents.xml` (or the `extension=` attribute names an external file like `gif`) |

## Relation kinds

| `type=` | Meaning |
| --- | --- |
| `Child` | Ownership — this object contains the target |
| `Reference` | Citation — this object points at the target but doesn't own it |
| `BackReference` | The inverse pointer maintained for a Reference/Child on the other side — read-only, don't try to derive ownership from it |

## The `_contents.xml` sibling, explained

Any attribute of kind `CHeavyData` on the metadata object is a signal that the *real* content lives in the adjacent `<ID>_contents.xml`. Designer does this to keep the CObject model compact and the big XML blobs diffable. You don't need to decode the metadata's `CHeavyData` declaration — just open the `_contents.xml` directly.
