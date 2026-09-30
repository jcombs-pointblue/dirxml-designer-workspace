# Policies, rules, filters, mappings

The content of a `*.ScriptPolicy_` is in its paired `<ID>_contents.xml`. Three dialects dominate — DirXML Script, XSLT, and attribute mapping — with filter XML as a fourth related dialect.

- A policy operates on an XDS document. Its primary purpose is to examine and modify that document.
- XDS is an XML document that follows the [nds DTD](https://www.netiq.com/documentation/identity-manager-developer/dtd-documentation/nds.dtd).
- An operation is any element in the XDS document that is a child of the `input` element or the `output` element.
- An operation usually represents an event, a command, or a status.
- A policy can also get additional context from outside the document and cause side effects not reflected in the result document.
