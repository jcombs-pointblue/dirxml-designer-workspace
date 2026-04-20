# Identity Vault, packages, servers, GCVs

## Identity Vault and schema

The Identity Vault is one eDirectory tree that IDM treats as the authoritative identity source. In Designer it lives under:

```
Model/IdentityManager/<DomainID>/<IdentityVaultID>.IdentityVault_
Model/IdentityManager/<DomainID>/<IdentityVaultID>/
├── <SchemaDefID>.SchemaDef_
└── <SchemaDefID>_schema.xml
```

The `_schema.xml` is a standard NDS schema document listing:

- **Attribute definitions** — name, syntax (e.g. `SYN_CI_STRING`, `SYN_INTEGER`, `SYN_DN`, `SYN_TYPED_NAME`), flags (multi-valued, read-only, hidden), ASN.1 OID
- **Class definitions** — name, containment rules (which classes it can live under), `MANDATORY`/`OPTIONAL` attribute sets, auxiliary class flags

Customers extend the schema for things the stock NDS schema can't represent: `DirXML-ADContext`, app-specific external-key attributes, custom multi-valued membership attrs, etc. When you see an attribute referenced in a filter or policy that isn't stock NDS, check this file first.

## Servers

```
Model/EdirOrphan/<ServerID>.Server_
```

Represents one IDM engine instance. The `.Driver_` files carry `associatedAttrSets objectURI="EdirOrphan/<ServerID>.Server_"` for per-server values — typically auth credentials, trace file paths, and initial driver state.

The `.DriverSet_` carries `Idm:Servers` Reference relations pointing at every Server the DriverSet is deployed to.

## Packages

Packages are the modern delivery mechanism for IDM — both vendor-shipped (from Micro Focus / OpenText, partner packages like Point Blue Tech's, etc.) and customer-built. Every non-trivial driver in the workspace was almost certainly built from packages.

**Package metadata** lives in two places:

1. The **catalog** — under `Model/Project/<ProjectID>/<IdmCatalogID>.IdmCatalog_` and the sibling `<IdmCatalogID>/` directory, which contains `IdmCategory_` and `IdmCategoryFolder_` nodes that group packages (e.g., "Active Directory Default Configuration Package", "Password Synchronization Common"). This is Designer's view of *available* packages.
2. **Installed packages** — referenced from the DriverSet and Drivers:

```xml
<relations name="Idm:InstalledPackages" type="Reference" key="#YOX3C2OQ.IdmPackage_"/>
```

Each `.IdmPackage_` object records the package id, version, vendor, and the set of objects (policies, entitlements, GCVs, schema) it installed. Tracing `Idm:InstalledPackages` tells you the provenance of every stock policy in the driver.

## Global Configuration Values (GCVs)

GCVs are typed, named driver/DriverSet parameters that policies read at runtime. There are three storage locations:

### 1. DriverSet-scoped GCVs

```
Model/EdirOrphan/<DriverSetID>/<GCVID>.GlobalConfig_
Model/EdirOrphan/<DriverSetID>/<GCVID>_initial_state.xml
Model/EdirOrphan/<DriverSetID>/<GCVID>_<ServerID>_DirXML-ConfigValues.xml
```

The `.GlobalConfig_` metadata file declares the GCV bundle. Initial values are in `_initial_state.xml`. Per-server overrides are in `_<ServerID>_DirXML-ConfigValues.xml`.

The DriverSet's `.DriverSet_` file relates to each GlobalConfig via `Idm:GlobalConfigs` (Child for ones it owns, Reference for `Idm:ConfigExtensions` that it imports).

### 2. Driver-scoped GCVs

Embedded in the `.Driver_` metadata as the `DirXML-ConfigValues` attribute (a `CHeavyData`, meaning its real content is in `<DriverID>_contents.xml` if the driver has one, or inline in the metadata file). The `DirXML-ShimConfigInfo` attribute holds a closely related but distinct blob — the driver-options / subscriber-options / publisher-options XML that the shim itself reads.

Per-server driver GCV bindings: `<DriverID>_<ServerID>_DirXML-ConfigValues.xml`.

### 3. Library-scoped

Libraries (`Library_`) can also hold `Idm:GlobalConfigs` children — shared GCVs available to any driver in the DriverSet.

## Engine Control Values (ECVs)

Separate from GCVs, ECVs tune the DirXML engine itself (not driver logic). They're carried on each `.Driver_` as `DirXML-EngineControlValues`, a CDATA XML blob of `<configuration-values>` definitions. Typical ECVs include `dirxml.engine.retry-interval`, `dirxml.engine.qualified-dn-values`, `dirxml.engine.max-replication-wait`, `dirxml.engine.rhino-ecma-engine`. Customers rarely change these; if they do, flag it.

## Named passwords

The DriverSet metadata can declare named passwords under `DirXML-NamedPasswords`:

```xml
<attributes xsi:type="com.novell.designer.model:CStructure" attrName="NamedPasswords">
  <attributes xsi:type="com.novell.designer.model:CStructure" attrName="NamedPassword">
    <attributes xsi:type="com.novell.designer.model:CString" attrName="NamedPassword:Name" value="NOVLLIBLDAP.password"/>
    <attributes xsi:type="com.novell.designer.model:CString" attrName="NamedPassword:DisplayName" value="LDAP Search Password"/>
  </attributes>
</attributes>
```

The password *values* are stored encrypted and not committed to the project — only the declarations exist on disk.

## Jobs

Scheduled jobs (`.Job_`) sit alongside drivers in the DriverSet. Each Job's `_contents.xml` declares the schedule (cron-like), the driver it runs against, and its XDS command document. Typical examples: nightly Data Collection Service resync, password-expiration notification.

## Notification templates

```
Model/EdirOrphan/<NotfCollectionID>.NotfTemplateCollection_
Model/EdirOrphan/<NotfCollectionID>/<TemplateID>.NotfTemplate_
```

The templates are reusable email bodies — policies and workflows reference them by name. The collection's `.digest` tracks change state for deployment.
