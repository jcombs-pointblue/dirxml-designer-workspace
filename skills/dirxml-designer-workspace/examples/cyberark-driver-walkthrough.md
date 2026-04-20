# Worked example: the CyberArk driver in `ig-idm`

This walkthrough traces one real driver end-to-end so you can see how the metadata, relations, policies, filter, schema map, and entitlements connect. Generalize from this pattern to every other driver in a workspace.

The driver is **CyberArk** (ID `9HPNAS5Y`) in project `ig-idm`, DriverSet `8C8HWGZY`. It's a Point Blue Tech SCIM driver that provisions CyberArk Identity from eDirectory.

Paths below are relative to the project root `designer_workspace49/ig-idm/`.

## 1. Find the driver

```
Model/EdirOrphan/8C8HWGZY.DriverSet_          ← DriverSet metadata
Model/EdirOrphan/8C8HWGZY/                    ← DriverSet contents
  9HPNAS5Y.Driver_                            ← CyberArk driver metadata
  9HPNAS5Y/                                   ← CyberArk driver contents
```

`9HPNAS5Y.Driver_` has `name="CyberArk"` on its root CObject. The file is the best single starting point for any driver — it names the shim, carries auth, carries the driver-options / subscriber-options / publisher-options XML (`DirXML-ShimConfigInfo`), and lists every relation that owns or points at the driver's children.

## 2. Read the driver metadata

From `9HPNAS5Y.Driver_`:

- `name="CyberArk"`
- `DirXML-JavaModule=com.pointbluetech.idm.cyberark.scim.CyberarkShim` — Point Blue's SCIM shim, not a stock OpenText one.
- `DirXML-DriverStartOption=1` — manual start.
- `DirXML-TraceFile=/opt/novell/trace/ca/ca.trace`, `DirXML-TraceLevel=10`, `DirXML-TraceName=CyberArk`.
- `DirXML-ShimAuthServer=https://example-tenant.id.cyberark.cloud`
- `DirXML-ShimAuthID=service-account@cyberark.cloud.tenant-id` (per-server; under `associatedAttrSets objectURI="EdirOrphan/1DE5WUEO.Server_"`)
- `DirXML-ShimAuthPassword=^[]REDACTED…==` (obfuscated per-server value)

The per-server values live inside `<associatedAttrSets objectURI="EdirOrphan/1DE5WUEO.Server_">` — that's the pattern whenever a driver attribute varies by engine.

### Driver options (what the shim itself reads)

`DirXML-ShimConfigInfo` holds a nested `<driver-config>` XML blob (as CDATA). Key values:

- `authScope=oauth-scope` — OAuth scope.
- `directorySourcedUsers=true` — CyberArk users come from AD, not created fresh.
- `domainName=example.com` — the AD domain as CyberArk stores it.
- `dummySafeName=IGA-TEST1` — dummy safe used to activate new CyberArk users.
- `setGroupMembership=true` / `cyberArkGroupGuid=redacted-guid` / `cyberArkGroupEntDN=system\driverset1\Active Directory Driver\Group` — after a user is created, the driver auto-assigns this AD group entitlement to push activation.
- `provisionCyberArkADGroup=true`, `retryOnUserActivation=true`.
- Publisher: `pub-heartbeat-interval=1`, `polling-interval=1` (minutes).

These feed both the shim and — via the `$config.*` token namespace — any policies that want to behave differently per deployment.

### Engine Control Values

`DirXML-EngineControlValues` on the same per-server `associatedAttrSets` carries the stock ECV bundle. Default values, nothing surprising — worth noting only if a future investigation hinges on retry/timeout behavior.

## 3. Walk the relations on the Driver

```xml
<relations name="Idm:Application"         type="BackReference" key="#L02ISABZ.Application_"/>
<relations name="Idm:Entitlements"        type="Child"         key="#VDMLE55M.Entitlement_"/>  <!-- SafePermission -->
<relations name="Idm:Entitlements"        type="Child"         key="#DVFFQAEV.Entitlement_"/>  <!-- Entitlement (legacy)  -->
<relations name="Idm:Entitlements"        type="Child"         key="#0HE17EY9.Entitlement_"/>  <!-- Group -->
<relations name="Idm:Entitlements"        type="Child"         key="#284U9YGD.Entitlement_"/>  <!-- UserAccount -->
<relations name="Idm:ExtensionFunctions"  type="Reference"     key="#YTH4IZEU.ECMAScriptResource_"/>
<relations name="Idm:ExtensionFunctions"  type="Reference"     key="#JY67WNIY.ECMAScriptResource_"/>
<relations name="Idm:Filter"              type="Child"         key="#CCDOVU0L.Filter_"/>
<relations name="Idm:InputPolicies"       type="Reference"     key="#95RKY8MV.ScriptPolicy_"/>
<relations name="Idm:MappingPolicies"     type="Reference"     key="#BQWYE4N1.MappingPolicy_"/>
<relations name="Idm:Policies"            type="Child"         key="#95RKY8MV.ScriptPolicy_"/>
<relations name="Idm:Policies"            type="Child"         key="#BQWYE4N1.MappingPolicy_"/>
<relations name="Idm:Publisher"           type="Child"         key="#OT93JX5M.Publisher_"/>
<relations name="Idm:Resources"           type="Child"         key="#JY67WNIY.ECMAScriptResource_"/>
<relations name="Idm:Subscriber"          type="Child"         key="#68UIVV7H.Subscriber_"/>
```

Reading this graph cold, you already know:

- Four entitlements: `SafePermission`, `Entitlement`, `Group`, `UserAccount`.
- One filter (`CCDOVU0L.Filter_`), one schema mapping (`BQWYE4N1.MappingPolicy_`), one driver-level input transform (`95RKY8MV.ScriptPolicy_`).
- One Subscriber channel (`68UIVV7H.Subscriber_`) with its own policies; Publisher (`OT93JX5M.Publisher_`) is present but empty — this driver is effectively subscriber-only.
- Two ECMAScript resources: one is driver-scoped (`JY67WNIY`, a Child under `Idm:Resources`), the other (`YTH4IZEU`) is a Library-scoped helper reachable through `Idm:ExtensionFunctions`.
- The Modeler app icon is `L02ISABZ.Application_` — BackReference, so the Application file points back at this driver.

## 4. The Subscriber channel and its policy sets

`9HPNAS5Y/68UIVV7H.Subscriber_` has only `<relations>` — no attributes. The **order of the `<relations>` elements is the execution order** inside each policy set.

```xml
<!-- Event Transformation (runs first on Subscriber) -->
<relations name="Idm:EventPolicies"  type="Reference" key="#HHPESU9W.StylesheetPolicy_"/>
<relations name="Idm:EventPolicies"  type="Reference" key="#P14GZC5T.ScriptPolicy_"/>
<relations name="Idm:EventPolicies"  type="Reference" key="#78WCMJ86.ScriptPolicy_"/>
<relations name="Idm:EventPolicies"  type="Reference" key="#PXEKDD2K.ScriptPolicy_"/>
<relations name="Idm:EventPolicies"  type="Reference" key="#QL6IEZFP.ScriptPolicy_"/>

<!-- Matching -->
<relations name="Idm:MatchingPolicies" type="Reference" key="#E9IMBZYH.ScriptPolicy_"/>

<!-- Create (when no match found) -->
<relations name="Idm:CreatePolicies"   type="Reference" key="#OCZYVM8C.ScriptPolicy_"/>

<!-- Command Transformation (runs last before dispatch to shim) -->
<relations name="Idm:CommandPolicies"  type="Reference" key="#HQNG5INE.ScriptPolicy_"/>
<relations name="Idm:CommandPolicies"  type="Reference" key="#A7ATIROM.ScriptPolicy_"/>
<relations name="Idm:CommandPolicies"  type="Reference" key="#CSM00TCG.ScriptPolicy_"/>

<!-- Every policy the Subscriber owns (separate Child list) -->
<relations name="Idm:Policies" type="Child" key="..."/>  <!-- 11 entries including two not bound above -->
```

Resolving each key to `name=` from its `.ScriptPolicy_` or `.StylesheetPolicy_` gives the readable picture:

| Stage | Order | ID | Name |
| --- | --- | --- | --- |
| Event Transform | 1 | `HHPESU9W` | *(XSLT — presents shim output conversions)* |
| Event Transform | 2 | `P14GZC5T` | `sub-etp-Trigger-Test-Events` |
| Event Transform | 3 | `78WCMJ86` | `sub-etp-IG-Only-Sync-Mode` |
| Event Transform | 4 | `PXEKDD2K` | `sub-etp-Veto-Unsupported-Event` |
| Event Transform | 5 | `QL6IEZFP` | `sub-etp-Scoping` |
| Matching | 1 | `E9IMBZYH` | `sub-mp-Matching Rule` |
| Create | 1 | `OCZYVM8C` | `CreatePolicy` |
| Command Transform | 1 | `HQNG5INE` | `sub-ctp-ConvertAddToGroupAssignment` |
| Command Transform | 2 | `A7ATIROM` | `sub-ctp-Set Entitlements` |
| Command Transform | 3 | `CSM00TCG` | `sub-ctp-Add Attributes` |

The two extra policies under `Idm:Policies` that aren't bound to a stage (`P71PZR2B` "FIndUser", etc.) are owned by the Subscriber for organization but currently inactive — typical workbench leftovers.

### A representative rule: `sub-etp-IG-Only-Sync-Mode`

The content is at `9HPNAS5Y/68UIVV7H/78WCMJ86_contents.xml`. Relevant excerpt:

```xml
<policy>
  <rule>
    <description>Determine if IG Sync Mode</description>
    <comment xml:space="preserve">Disable all User transactions, allow IG collection only</comment>
    <conditions>
      <and>
        <if-global-variable mode="nocase" name="drv.ig.enable.sync.only.mode" op="equal">true</if-global-variable>
        <if-class-name mode="nocase" op="equal">User</if-class-name>
        <if-operation mode="regex" op="equal">add|modify|delete|sync|rename|move</if-operation>
      </and>
    </conditions>
    <actions>
      <do-veto/>
    </actions>
  </rule>
</policy>
```

Read that as: *"if the deployment has been flipped into IG-only mode, drop every User mutation on the Subscriber side."* The GCV `drv.ig.enable.sync.only.mode` is defined at DriverSet scope (it's the feature flag Point Blue uses to pause provisioning while Identity Governance is performing a cold-load collection).

The pattern — `<description>` is the human intent, `<conditions>/<and>/<if-*>` is the guard, `<actions>/<do-*>` is the effect — is universal across DirXML Script policies.

## 5. The Filter

`9HPNAS5Y/CCDOVU0L_contents.xml` (root `<filter>`) tells you what classes and attributes actually flow:

- `<filter-class class-name="User" publisher="ignore" subscriber="sync">` with `<filter-attr>` entries for `Given Name`, `Surname`, `CN`, `Internet EMail Address`, `workforceID`, `DirXML-ADContext`, `DirXML-EntitlementRef` (subscriber="notify" publisher="sync" — entitlement events are read-only to the subscriber), etc.
- `<filter-class class-name="Group" publisher="ignore" subscriber="sync">` for the AD group passthrough.

Because every `filter-class` has `publisher="ignore"`, this is a one-way driver: eDir → CyberArk.

## 6. The Schema Mapping

`9HPNAS5Y/BQWYE4N1_contents.xml` (root `<attr-name-map>`) is the translation table. Example entries:

```xml
<class-name>
  <nds-name>User</nds-name>
  <app-name>User</app-name>
</class-name>
<attr-name class-name="User">
  <nds-name>Given Name</nds-name>
  <app-name>name.givenName</app-name>
</attr-name>
<attr-name class-name="User">
  <nds-name>Internet EMail Address</nds-name>
  <app-name>emails[type eq "work"].value</app-name>
</attr-name>
```

The `app-name` values are SCIM attribute paths — this is how the same eDir attribute ends up on the right SCIM leaf when the shim builds the request.

## 7. The Entitlements

Each `.Entitlement_` in `9HPNAS5Y/` has a paired `_contents.xml` whose root is `<entitlement>`. For example, `0HE17EY9_contents.xml` (the `Group` entitlement) looks like:

```xml
<entitlement conflict-resolution="union" display-name="Group" description="Group Membership">
  <multi-value/>
  <query>
    <query-app object-class="Group"/>
    <display-name-ref>
      <token-attr name="CN"/>
    </display-name-ref>
    <description-ref>
      <token-attr name="description"/>
    </description-ref>
    <ent-value-ref>
      <token-association/>
    </ent-value-ref>
  </query>
</entitlement>
```

That's a **dynamic** entitlement: at runtime the driver queries CyberArk for all Group objects and exposes each as an assignable value. `SafePermission` and `UserAccount` follow the same pattern but for their respective SCIM resource types.

When the User App (`Model/Provisioning/`) grants a Resource, the Resource's `ResourceDef_` names one of these entitlements by DN. That entitlement's values are then written to `DirXML-EntitlementRef` on the User in eDir, the sub-channel command policy `sub-ctp-Set Entitlements` picks it up, and the shim dispatches the SCIM request.

## 8. Driver-level Input Transform

`9HPNAS5Y/95RKY8MV.ScriptPolicy_` is the lone driver-wide input transform — it runs on every event coming in from the shim before the Publisher channel sees it. In this driver it's thin (a pass-through with a couple of status-mapping rules), but it's the right place to normalize inbound XDS before the publisher stages.

## 9. GCVs and Library helpers

The DriverSet (`8C8HWGZY.DriverSet_`) owns several `.GlobalConfig_` bundles and a `.Library_`. The library's `.Library_` file has `Idm:GlobalConfigs` and `Idm:Resources` relations — inside that library is the ECMAScript resource `YTH4IZEU` that the driver imports via `Idm:ExtensionFunctions`. Policies call those functions with `<token-xpath>es:helperName(...)</token-xpath>`.

The driver-scoped ECMAScript (`JY67WNIY`) lives directly under the driver's children — same JS dialect, narrower scope.

## 10. Packages

The DriverSet's `Idm:InstalledPackages` relations enumerate every vendor/partner package installed. Almost every policy prefixed `sub-etp-`, `sub-ctp-`, `pub-ctp-`, etc. is package-shipped and should be treated as "vendor code — change only with care." Custom policies will usually have a distinct naming convention (here: the `CreatePolicy`, `FIndUser`, plain `Entitlement` names that don't follow the `{sub|pub}-{stage}-...` pattern).

## 11. How to describe this driver in one paragraph

> The **CyberArk** driver (`9HPNAS5Y`, shim `com.pointbluetech.idm.cyberark.scim.CyberarkShim`) is a subscriber-only SCIM provisioner targeting `https://example-tenant.id.cyberark.cloud` under OAuth scope `oauth-scope`. It expects CyberArk users to be created from AD (`directorySourcedUsers=true`) and completes activation by auto-assigning the AD Group entitlement `system\driverset1\Active Directory Driver\Group` (GUID `redacted…`). The Subscriber filter passes `User` and `Group` events only; the schema map translates NDS names to SCIM attribute paths. Four dynamic entitlements are exposed: `SafePermission`, `Entitlement` (legacy), `Group`, and `UserAccount`. Event-transform policies include an IG-only kill switch (`drv.ig.enable.sync.only.mode` GCV) that vetoes User mutations while Identity Governance is cold-loading. Publisher is present but empty — no inbound sync from CyberArk.

That paragraph is what a "document this driver" request should produce. It pulls from: Driver metadata, relations graph, filter, schema map, entitlements, a single telltale policy rule, and the DriverSet-level feature flag.

## 12. Generalizing

For any driver in any workspace, the recipe is:

1. Grep `*.Driver_` for `name="..."` to find the file and ID.
2. Read the `.Driver_` metadata: shim module, auth, config-info CDATA, trace, start option.
3. List the relations — Subscriber, Publisher, Filter, MappingPolicies, Entitlements, Policies, InputPolicies.
4. Open `.Subscriber_` / `.Publisher_` and record the ordered policy-set relations. Resolve each key to a readable name.
5. For each interesting policy, read its `_contents.xml` and quote the `<description>` of rules that look custom (i.e., non-`sub-etp-*` / `pub-ctp-*` naming, or with non-obvious conditions).
6. Summarize the Filter: what classes flow, which direction.
7. Summarize the MappingPolicy: notable NDS↔app renames.
8. Enumerate Entitlements: static values vs dynamic queries, multi-valued or not.
9. Note GCVs that policies key on (anything the user would need to flip in production).
10. Mention package provenance via `Idm:InstalledPackages`.

Everything else is elaboration.
