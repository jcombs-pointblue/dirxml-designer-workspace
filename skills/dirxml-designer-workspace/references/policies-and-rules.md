# Policies, rules, filters, mappings

The content of a `*.ScriptPolicy_` is in its paired `<ID>_contents.xml`. Three dialects dominate — DirXML Script, XSLT, and attribute mapping — with filter XML as a fourth related dialect.

- A policy operates on an XDS document. Its primary purpose is to examine and modify that document.
- XDS is an XML document that follows the [nds DTD](https://www.netiq.com/documentation/identity-manager-developer/dtd-documentation/nds.dtd)`.
- An operation is any element in the XDS document that is a child of the `input` element or the `output` element.
- An operation usually represents an event, a command, or a status.
- A policy can also get additional context from outside the document and cause side effects not reflected in the result document.

## DirXML Script (the most common dialect)

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

Key grammar:

- `<policy>` — root
- `<rule>` — one rule; optionally `disabled="true"`. `<description>` is the short label developers use as the rule's intent. `<comment>` often carries a longer explanation.
- `<conditions>` — either `<and>` or `<or>` (or nested) wrapping `<if-*>` tests
- `<actions>` — a sequence of `<do-*>` actions

A policy is applied separately to each operation. As the policy is applied to each operation in turn, that operation becomes the current operation. Each rule is applied sequentially to the current operation. All of the rules are applied to the current operation unless an action is executed by a prior rule that causes subsequent rules to no longer be applied.

### Common conditions (`if-*`)

| Condition | Tests |
| --- | --- |
| `if-class-name` | The NDS class of the current object |
| `if-operation` | `add`, `modify`, `delete`, `query`, `rename`, `move`, `sync`, `status` |
| `if-association` | Association state: `associated`, `disabled`, `manual`, `migrate`, `pending`, `any` |
| `if-attr` | An attribute on the current object (optionally with a value test) |
| `if-src-attr` / `if-dest-attr` | Attribute on the *source* or *destination* object — triggers a query |
| `if-global-variable` / `if-local-variable` | GCV / local variable value test |
| `if-xpath` | Arbitrary XPath against the current event document — escape hatch |
| `if-xml-attr` | Test on an XML attribute of the event |
| `if-src-dn` / `if-dest-dn` / `if-entitlement` | Self-explanatory |
| `if-op-attr` / `if-op-property` | Operational attribute / operation property |

`mode=` on a condition governs comparison: `case`, `nocase`, `regex`, `src-dn`, `nds`, `numeric`, etc.

### Common actions (`do-*`)

| Action | Effect |
| --- | --- |
| `do-veto` / `do-veto-if-op-attr-not-available` | Drop the event |
| `do-break` | Stop this policy (next policy in the set runs) |
| `do-return` | Stop the whole channel's policy chain for this event |
| `do-set-dest-attr-value` | Set an attribute on the outgoing event |
| `do-add-dest-attr-value` / `do-remove-dest-attr-value` | Add/remove a value on the outgoing event |
| `do-set-src-attr-value` | Set attribute on the *source* (issues a write-back) |
| `do-set-dest-password` | Set password on destination |
| `do-set-op-dest-dn` | Change the destination DN of the operation |
| `do-add-association` / `do-remove-association` / `do-find-matching-object` | Association management |
| `do-set-local-variable` / `do-set-op-property` | Local state |
| `do-set-xml-attr` / `do-append-xml-element` | Direct event-document manipulation |
| `do-status` | Emit a status document (`success`, `warning`, `error`, `retry`, `fatal`) |
| `do-trace-message` | Write to driver trace |
| `do-send-email` | Dispatch via notification template |
| `do-reformat-op-attr` | Replace each value using an embedded token expression |
| `do-for-each` | Iterate; body usually uses `$current-node` |
| `do-implement-entitlement` | Standard boilerplate for entitlement handlers |

### Tokens (nested inside `arg-*`)

Actions take typed arguments: `<arg-string>`, `<arg-dn>`, `<arg-value>`, `<arg-association>`, `<arg-node-set>`, `<arg-match-attr>`, `<arg-component>`. Inside those go **tokens**, which are expressions:

| Token | Reads |
| --- | --- |
| `token-text` | Literal text |
| `token-src-attr` / `token-dest-attr` | Attribute value from source or dest object (triggers query on dest) |
| `token-attr` | Attribute value from the current operation document |
| `token-xml-attr` | XML attribute of the current node |
| `token-association` | The object's association for this driver |
| `token-src-dn` / `token-dest-dn` | DN of source / dest |
| `token-local-variable` / `token-global-variable` | Local var / GCV |
| `token-op-attr` / `token-added-attr` / `token-removed-attr` | Operation attribute values |
| `token-xpath` | Arbitrary XPath; the escape hatch |
| `token-query` | Embedded query to source or destination |
| `token-map` | Mapping-table lookup |
| `token-parse-dn` / `token-unmatched-src-dn` / `token-replace-all` | String manipulation |
| `token-base64-encode` / `token-base64-decode` / `token-upper-case` / `token-lower-case` / `token-substring` / `token-split` / `token-join` | Scalar transforms |
| `token-if-defined` | Conditional evaluation |

Namespaces you'll see on `<token-xpath>` / `<xsl>` expressions: `es:` (ECMAScript resource function — looks up a function defined in an `.ECMAScriptResource_`), `Map:` (mapping tables), and the runtime-injected ones below.

### How a policy attaches to a channel

A ScriptPolicy by itself is inert. It runs because a Subscriber or Publisher's metadata has a relation like:

```xml
<relations name="Idm:CommandPolicies" type="Reference" key="#HQNG5INE.ScriptPolicy_"/>
<relations name="Idm:CommandPolicies" type="Reference" key="#A7ATIROM.ScriptPolicy_"/>
<relations name="Idm:CommandPolicies" type="Reference" key="#CSM00TCG.ScriptPolicy_"/>
```

The ordering in the XML **is** the execution order within that policy set. Policy sets on a channel run in this order:

1. Event Transformation (`Idm:EventPolicies`)
2. Matching (`Idm:MatchingPolicies`) — only on add/modify where there's no association yet
3. Create (`Idm:CreatePolicies`) — only when no match found
4. Placement (`Idm:PlacementPolicies`)
5. Command Transformation (`Idm:CommandPolicies`)
6. Schema Mapping (`Idm:SchemaMappingPolicies`) — usually one shared `.MappingPolicy_`
7. Output Transformation (`Idm:OutputPolicies`) — driver-level, both channels share it
8. Input Transformation (`Idm:InputPolicies`) — driver-level, runs on events coming in

Policy sets on the *driver* (not a channel) are Input/Output Transformation, running just inside / just outside the shim.

## XSLT stylesheets

Root `<xsl:stylesheet>`. IDM injects these params — expect to see them declared at the top:

- `srcQueryProcessor`, `destQueryProcessor` — Java objects for issuing queries
- `srcCommandProcessor`, `destCommandProcessor` — Java objects for issuing commands
- `dnConverter` — DN conversion helper
- `fromNds` — boolean, true if the event is coming from the Identity Vault

Common extension namespaces:

```
xmlns:cmd="http://www.novell.com/nxsl/java/com.novell.nds.dirxml.driver.XdsCommandProcessor"
xmlns:query="http://www.novell.com/nxsl/java/com.novell.nds.dirxml.driver.XdsQueryProcessor"
xmlns:dncv="http://www.novell.com/nxsl/java/com.novell.nds.dirxml.driver.DNConverter"
```

Stylesheets typically include an identity transform (`xsl:template match="node()|@*"` that recursively copies) plus specific `<xsl:template match="...">` rules for the events they transform.

## Mapping policies

Root `<attr-name-map>`. Each `<class-name>` maps `<nds-name>` ↔ `<app-name>` for an NDS class. `<attr-name class-name="...">` does the same for attributes of a class.

```xml
<attr-name-map>
  <class-name>
    <nds-name>User</nds-name>
    <app-name>User</app-name>
  </class-name>
  <attr-name class-name="User">
    <nds-name>Given Name</nds-name>
    <app-name>givenName</app-name>
  </attr-name>
</attr-name-map>
```

When describing a driver, the mapping policy is the fastest way to answer "what do NDS attributes get called on the other end?"

## Filter

Root `<filter>` with `<filter-class>` per NDS class, `<filter-attr>` per attribute. Attributes:

- `publisher=` — `sync`, `notify`, `ignore`, `reset` (what the Publisher channel does with this attr)
- `subscriber=` — same choices for the Subscriber
- `merge-authority=` — `edir`, `app`, `default`, `none` — who wins conflicts
- `publisher-optimize-modify=` / `subscriber-optimize-modify=` — whether to filter out no-op writes

```xml
<filter>
  <filter-class class-name="User" publisher="sync" subscriber="sync" publisher-create-homedir="true" ...>
    <filter-attr attr-name="Given Name" subscriber="sync" publisher="ignore" merge-authority="default" .../>
    <filter-attr attr-name="DirXML-EntitlementRef" subscriber="notify" publisher="sync" merge-authority="edir" .../>
  </filter-class>
</filter>
```

"What classes and attributes flow, and which direction?" is answered entirely from this file.

## GCVs / ECVs

**Global Configuration Values** (GCVs) are typed, named parameters. They live at three scopes:

- DriverSet-level: `.GlobalConfig_` objects under the DriverSet
- Driver-level: embedded `DirXML-ConfigValues` CDATA inside the `.Driver_` metadata file
- Server-specific overrides: `<DriverID>_<ServerID>_DirXML-ConfigValues.xml`

**Engine Control Values** (ECVs) are a separate, IDM-engine-level set carried on the `.Driver_` metadata as `DirXML-EngineControlValues`. These tune engine behavior (retry interval, qualified DN form, XSLT backward-compat mode, etc.).

Policies access both via `<if-global-variable>` / `<token-global-variable>` — the GCV namespace is flat.

## Recognizing package-sourced policies

Many policies are unchanged from vendor IDM packages. Tell-tales:

- `DirXML-ConfigManifest` on the Driver pulling them in
- `Idm:InstalledPackages` relations on the Driver or DriverSet
- Policy names prefixed with `NOV-`, `NETQ-`, `MICF-`, or following the `{sub|pub}-{stage}-{description}` convention (e.g. `sub-etp-IG-Only-Sync-Mode`, `pub-ctp-Detect-Attribute-Changes`)
- `com.novell.idm:ReadOnly` or similar attribute flags in the metadata

When describing a driver, flag which policies are custom vs. package-stock — the customer's intent is in the custom ones.
