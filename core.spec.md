# Core Schemas

<h2>extensible GraphQL schemas for powering data cores</h2>

```raw html
<table class=spec-data>
  <tr><td>Status</td><td>Draft</td>
  <tr><td>Version</td><td>0.1</td>
</table>
<link rel=stylesheet href=/apollo-dark.css>
<link rel=stylesheet href=/tron.css>
<script type=module async defer src=/install-nav.js></script>
```

# Problem

[GraphQL](https://spec.graphql.org/) provides directives as a means of attaching user-defined metadata to a GraphQL document. Directives are highly flexible, and can be used to suggest behavior and define features of a graph which are not otherwise evident in the schema.

Alas, *GraphQL does not provide a mechanism to globally identify or version directives*. Given a particular directive—e.g. `@join`—processors are expected to know how to interpret the directive based only on its name, definition within the document, and additional configuration from outside the document. This means that programs interpreting these directives have two options:

  1. rely on a hardcoded interpretation for directives with certain signatures, or
  2. accept additional configuration about how to interpret directives in the schema.

The first solution is fragile, particularly as GraphQL has no built-in namespacing mechanisms, so the possibility of name collisions always looms.

The second is unfortunate: GraphQL schemas are generally intended to be self-describing, and requiring additional configuration subtly undermines this guarantee: given just a schema, programs do not necessarily know how to interpret it, and certainly not how to serve it. It also creates the possibility for the schema and configuration to fall out of sync, leading to issues which can manifest late in a deployment pipeline.

# Core Schemas

Introducing **core schemas**.

<div class=hbox>
  <a class=core>
    <div class=ring></div>
    <div class=nucleus>core schema</div>
  </a>
</div>

A basic core schema:

===[Basic core schema](basic.graphql) example

**Core schemas** provide a concise mechanism for schema documents to specify the metadata they provide. Metadata is grouped into **features**, which typically define directives and associated types (e.g. scalars and inputs which serve as directive inputs). Additionally, core schemas provide:
  - [**Flexible namespacing rules.**](#sec-Prefixing) It is always possible to represent any GraphQL schema within a core schema document. Additionally, documents can [choose the names](#@core/as-String) they use for the features they reference, guaranteeing that namespace collisions can always be resolved.
  - [**Versioning.**](#sec-Versioning) Feature specifications follow [semver-like semantic versioning principles](#sec-Versioning), which helps schema processors determine if they are able to correctly interpret a document's metadata.
  - [**Standard rules for extensibility**](#sec-Extensibility) of directives and their associated input types. These rules allow specs to annotate each others' metadata, effectively providing "directives on directives," which are otherwise impossible within GraphQL schemas

**Core schemas are not a new language.** All core schema documents are valid GraphQL schema documents. However, this specification introduces new requirements, so not all valid GraphQL schemas are valid core schemas.

The broad intention behind core schemas is to provide a *single document* which provides all the necessary configuration of a *data core*—some program which serves the schema to GraphQL clients, primarily by following directives in order to determine how to resolve queries made against that schema.

## Parts of a Core Schema

When talking about a core schema, we can broadly break it into two pieces:
- an **API** consisting of exported schema elements (objects, interfaces, enums, directives, etc.) which **should** be served to clients, and
- **machinery** containing document metadata. This typically consists of directives and associated input types (such as enums and input objects), but may include any schema element. Machinery **must not** be served to clients. Specifically, machinery **must not** be included in introspection responses or used to validate or execute queries.

This reflects how core schemas are used: a core schema contains a GraphQL interface (the *API*) along with metadata about how to implement that interface (the *machinery*). Exposing the machinery to clients is unnecessary, and may in some cases constitute a security issue (for example, the machinery for a public-facing graph router will likely reference internal services, possibly exposing network internals which should not be visible to the general public).

A key feature of core schemas is that it is always possible to derive a core schema's API without any knowledge of the features used by the document (with the exception of the `core` feature itself).

Approximately, the process is:
  - Any elements named `something__likeThis` are **not exported**, unless&#8230;
    - &#8230;the feature which provides them ("`something`") has `export: true` on its `@core(feature:)` declaration, *or*
    - &#8230;they are annotated with {@core__export} (or explicitly, {@core__export}`(isExport: true)`).
  - Any elements with `normalNames` are **exported**, unless&#8230;
    - &#8230;the element is annotated with {@core__export}`(isExport: false)`, *or*
    - &#8230;the element is a directive or directive definition whose name matches the name of a feature, in which case, it is exported only if the entire feature is exported (having been brought in with {@core}`(feature:, export: true)`).

A formal description is provided by the [is exported](#sec-Is-Exported-) algorithm.

# Roles

```mermaid diagram -- Actors who may be interested in the core schemas
graph TB
  author("👩🏽‍💻 🤖  &nbsp;Author")-->schema(["☉ Core Schema"])
  schema-->proc1("🤖 &nbsp;Processor")
  proc1-->output1(["☉ Core Schema[0]"])
  output1-->proc2("🤖 &nbsp;Processor")
  proc2-->output2(["☉ Core Schema[1]"])
  output2-->etc("...")
  etc-->final(["☉ Core Schema [final]"])
  final-->core("🤖 Data Core")
  schema-->reader("👩🏽‍💻  Reader")
  output1-->reader
  output2-->reader
  final-->reader
```

- **Authors (either human or machine)** write an initial core schema as specified in this document, including versioned {@core} requests for all directives they use
- **Machine processors** can process core schemas and output new core schemas. The versioning of directives and associated schema elements provided by the {@core} allows processors to operate on directives they understand and pass through directives they do not.
- **Human readers** can examine the core schema at various stages of processing. At any stage, they can examine the {@core} directives and follow URLs to the specification, receiving an explanation of the requirements of the specification and what new directives, types, and other schema objects are available within the document.
- **Data cores** can then pick up the processed core schema and provide some data-layer service with it. Typically this means serving the schema's API as a GraphQL endpoint, using metadata defined by machinery to inform how it processes operations it receives. However, data cores may perform other tasks described in the core schema, such as routing to backend services, caching commonly-accessed fields and queries, and so on. The term "data core" is intended to capture this multiplicity of possible activities.

# The Basics

Core schemas:
  1. **must** be valid GraphQL schema documents,
  2. **must** contain exactly one `SchemaDefinition`, and
  3. **must** use the {@core} directive on their schema definition to declare any features they reference by using {@core} to reference a [well-formed feature URL](#core__FeatureUrl).

The first {@core} directive on the schema **must** reference the core spec itself, i.e. this document.

===[Basic core schema using {@core} and `@example`](basic.graphql) example

## Unspecified directives are passed through

Existing schemas likely contain definitions for directives which are not versioned, have no specification document, and are intended mainly to be passed through. This is the default behavior for core schema processors:

```graphql example -- Unspecified directives are passed through
schema
  @core(feature: "https://lib.apollo.dev/core/v0.1")
{
  query: Query
}

type SomeType {
  field: Int @another
}

# `@another` is unspecified. Core processors will not extract metadata from
# it, but its definition and all usages within the schema will be exposed
# in the API.
directive @another on FIELD
```

## Renaming core itself

It is possible to rename the core feature itself with the same `as:` mechanism used for all features:

```graphql example -- Renaming {@core} to {@coreSchema}
schema
  @coreSchema(feature: "https://lib.apollo.dev/core/v0.1", as: "coreSchema")
  @coreSchema(feature: "https://example.com/example/v1.0")
{
  query: Query
}

type SomeType {
  field: Int @example
}

directive @example on FIELD
```

# Directives

##! @core

Declare a core feature present in this schema.

```graphql definition
directive @core(
  feature: core__FeatureUrl!,
  as: String,
  export: Boolean)
  repeatable on SCHEMA
```

Documents **must** include a definition for the {@core} directive. The provided definition must be *compatible* with the definition above, but may:
- **Omit optional arguments** if they are never used in the document,
- **Omit locations** where the directive never occurs,
- **Introduce new directive locations**, the behavior of which is unspecified here.
- **Introduce new arguments.** New arguments **must** be prefixed, e.g. `example__extensionArgument: Bool`. The prefix **must** be the name of a feature referenced with this or another {@core} directive within the document.
- **Use `String` in place of custom scalars.** This is an ease-of-authoring affordance which allows document authors to omit the definitions of custom scalar types provided by features. Processors **must** continue to interpret such arguments and fields according to the encoding and decoding rules of the custom scalar type. For example, {@core} may be defined as follows, with no change to the behavior:

```graphql example -- {@core} definition taking a String feature
directive @core(feature: String)
```

###! feature: core__FeatureUrl

A [feature URL](#core__FeatureUrl) specifying the directive and associated schema elements. When viewed, the URL **should** provide the content of the appropriate version of the specification in some human-readable form. In short, a human reader should be able to click the link and go to the docs for the version in use. There are [specific requirements](#core__FeatureUrl) on the format of the URL, but it is not required that the *content* be machine-readable in any particular way.

Feature URLs contain information about the spec's [prefix](#sec-Prefixing) and [version](#sec-Versioning), and should be generated and processed in accordance with the [requirements on the structure of feature URLs](#core__FeatureUrl).

###! as: String

Change the [names](#sec-Prefixing) of directives and schema elements from this specification. The specified string **must** be a valid GraphQL identifier and **must not** contain the namespace separator (two underscores, {"__"}).

When `as:` is provided, processors **must** replace the default name prefix on the names of all [prefixed schema elements](#sec-Elements-which-must-be-prefixed) with the specified name.

```graphql example -- Using {@core}`(feature:, as:)` to use a feature with a custom name
schema
  @core(feature: "https://spec.example.com/core/v1.0")
  @core(feature: "https://spec.example.com/example/v1.0", as: "eg")
{
  query: Query
}

type User {
  # Specifying `as: "eg"` transforms @example into @eg
  name: String @eg(data: ITEM)
}

# Additional specified schema elements must have their prefixes set
# to the new name.
#
# This data enum was specified as `example__Data`, but will be renamed
# as `eg__Data`:
enum eg__Data {
  ITEM
}

# Name transformation must also be applied to definitions pulled in from
# specifications.
directive @eg(data: eg__Data) on FIELD

# (...other definitions omitted...)
```

##! @core__export

Specify whether the annotated element is exported to the API.

```graphql definition
directive @core__export(isExport: Boolean! = true)
  repeatable on
  | SCALAR
  | OBJECT
  | FIELD_DEFINITION
  | ARGUMENT_DEFINITION
  | INTERFACE
  | UNION
  | ENUM
  | ENUM_VALUE
  | INPUT_OBJECT
  | INPUT_FIELD_DEFINITION
```

{@core__export} can occur at any type system location. Elements with {@core__export} will always be included in the API. Elements with {@core__export}`(isExport: false)` will always be excluded from the API.

###! isExport: Boolean

If true, the element is always exported, regardless of whether the feature which defines it is exported. If false, the element is never exported.

# Scalars

##! core__FeatureUrl

```graphql definition
scalar core__FeatureUrl
  @specifiedBy(url: "https://lib.apollo.dev/core/v0.1#core__FeatureUrl")
```

Feature URLs serve two main purposes:
  - Directing human readers to documentation about the feature
  - Providing tools with information about the specs in use, along with enough information to select and invoke an implementation
  
Feature URLs **should** be [RFC 3986 URLs](https://tools.ietf.org/html/rfc3986). When viewed, the URL **should** provide the specification of the selected version of the feature in some human-readable form; a human reader should be able to click the link and go to the correct version of the docs.

Although they are not prohibited from doing so, it's assumed that processors will not load the content of feature URLs. Published specifications are not required to be machine-readable, and [this spec](.) places no requirements on the structure or syntax of the content to be found there.

There are, however, requirements on the structure of the URL itself:

```html diagram -- Basic anatomy of a feature URL
<code class=anatomy>  
  <span class=pink style='--depth: 2'>https://spec.example.com/a/b/c/<span>exampleFeature<aside>name</aside></span><aside>identity</aside></span>/<span style='--depth: 2' class=green>v1.0<aside>version</aside></span>
</code>
```

The final two segments of the URL's [path](https://tools.ietf.org/html/rfc3986#section-3.3) **must** contain the feature's name and a [version tag](#sec-Versioning). The content of the URL up to and including the prefix—but excluding the version tag and trailing `/`—is the feature's *identity*. For the above example,
<dl>
  <dt>`identity: "https://spec.example.com/a/b/c/exampleFeature"`</dt>
  <dd>A global identifier for the feature. Processors can treat this as an opaque string identifying the feature (but not the version of the feature) for purposes of selecting an appropriate implementation.</dd>
  <dt>`name: "exampleFeature"`</dt>
  <dd>The feature's name, for purposes of [prefixing](#sec-Prefixing) schema elements it defines or extends.</dd>
  <dt>`version: "v1.0"`</dt>
  <dd>The tag for the [version](#sec-Versioning) of the feature used to author the document. Processors **must** select an implementation of the feature which can [satisfy](#sec-Satisfaction) the specified version.</dd>
</dl>

The version tag **must** be a valid {VersionTag}.

### Ignore meaningless URL components

When extracting the URL's `name` and `version`, processors **must** ignore any url components which are not assigned a meaning. This spec assigns meaning to the final two segments of the [path](https://tools.ietf.org/html/rfc3986#section-3.3). Other URL components—particularly query strings and fragments, if present—**must** be ignored for the purposes of extracting the `name` and `version`.

```html diagram -- Ignoring meaningless parts of a URL
<code class=anatomy>
  <span class=pink style='--depth: 2'>https://example.com/<span>exampleSpec<aside>name</aside></span><aside>identity</aside></span>/<span style='--depth: 2' class=green>v1.0<aside>version</aside></span><span class=grey>?key=val&k2=v2#frag<aside>ignored</aside></span>
</code>
```

### Why is versioning in the URL, not a directive argument?

The version is in the URL because when a human reader visits the URL, we would like them to be taken to the documentation for the *version of the feature used by this document*. Many text editors will turn URLs into hyperlinks, and it's highly desirable that clicking the link takes the user to the correct version of the docs. Putting the version information in a separate argument to the {@core} directive would prevent this.

# Prefixing

With the exception of a single root directive, core feature specifications **must** prefix all schema elements they introduce. The prefix:
1. **must** match the default name of the feature as derived from the feature's specification URL,
1. **must** be a string of characters valid within GraphQL names, and
2. **must not** contain the core namespace separator, which is two underscores ({"__"}).

Prefixed names consist of the name of the feature, followed by two underscores, followed by the name of the element (which can be any valid GraphQL identifier). For instance, the `core` specification (which you are currently reading) introduces elements named [{core__FeatureUrl}](#core__FeatureUrl) and [{@core__export}](#@core__export).

A feature's *root directive* is an exception to the prefixing requirements. Feature specifications **may** introduce a single directive which carries only the name of the feature, with no prefix required. For example, the `core` specification introduces a {@core} directive. This directive has the same name as the feature ("`core`"), and so requires no prefix.

```graphql example -- Using the @core directive with a spec's default name
schema @core(feature: "https://spec.example.com/example/v1.0") {
  query: Query
}

type User {
  name: String @example(data: ITEM)
}

# An enum used to provide structured data to the example spec.
# It is prefixed with the name of the spec.
enum example__Data {
  ITEM
}

directive @example(data: example__Data) on FIELD
```

The prefix **must not** be elided within documentation; definitions of schema elements provided within the spec **must** include the default prefix.

## Elements which must be prefixed

Feature specs **must** prefix the following schema elements:
- the names of any object types, interfaces, unions, enums, or input object types defined by the feature
- the names of any fields the spec introduces on *foreign types* defined in a different feature
- the names of any arguments the spec introduces on *foreign directives and fields* defined by a different feature
- the names of any directives introduced in the spec, with the exception of the *root directive*, which must have the same name as the feature

===[Prefixing examples](prefixing.graphql) example

# Versioning

VersionTag : "v" Version

Version : Major "." Minor

Major : NumericIdentifier

Minor : NumericIdentifier

NumericIdentifier : "0"
  | PositiveDigit Digit*

Digit : "0" | PositiveDigit

PositiveDigit : "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9"

Specs are versioned with a **subset** of a [Semantic Version Number](https://semver.org/spec/v2.0.0.html) containing only the major and minor parts. Thus, specifications **should** provide a version of the form {Major}.{Minor}, where both integers >= 0.

```text example -- Valid version tags
v2.2
v1.0
v1.1
v0.1
```

As specified by semver, spec authors **should** increment the:

{++

- MAJOR version when you make incompatible API changes,
- MINOR version when you add functionality in a backwards compatible manner

++}

Patch and pre-release qualifiers are judged to be not particularly meaningful in the context of core features, which are (by definition) interfaces rather than implementations. The patch component of a semver denotes a bug fix which is backwards compatible—that is, a change to the implementation which does not affect the interface. patch-level changes in the version of a spec denote wording clarifications which do not require implementation changes. As such, it is not important to track them for the purposes of version resolution.

As with [semver](https://semver.org/spec/v2.0.0.html), the `0.x` version series is special: there is no expectation of compatibility between versions `0.x` and `0.y`. For example, a processor must not activate implementation `0.4` to satisfy a requested version of `0.2`.

## Ordering

Given {Version}s {a} and {b}, return whether {a} is {LESS_THAN}, {GREATER_THAN}, or {EQUAL} to {b}.

CompareVersions(a, b) :
  1. If {a}.{Major} < {b}.{Major}, return {LESS_THAN}
  2. If {a}.{Major} > {b}.{Major}, return {GREATER_THAN}
  3. If {a}.{Major} == {b}.{Major}:
    1. If {a}.{Minor} == {b}.{Minor}, return {EQUAL}
    2. If {a}.{Minor} > {b}.{Minor}, return {GREATER_THAN}
    3. Otherwise, return {LESS_THAN}

Note: This ordering does *not* guarantee that if A > B then A can satisfy a version requirement of B. For example, `2.0 > 1.9`, but `2.0` cannot [satisfy](#sec-Satisfaction) the requirements of a document which references version `1.9`, since the major version update introduced possibly-breaking changes.

## Satisfaction

Given a version {requested} by a document and an {available} version of an implementation, the following algorithm will determine if the {available} version can satisfy the {requested} version:

Satisfies(requested, available) :
  1. If {requested}.{Major} ≠ {available}.{Major}, return {false}
  2. If {requested}.{Major} = 0, return {requested}.{Minor} = {available}.{Minor}
  3. Return {requested}.{Minor} <= {available}.{Minor}

## Referencing versions and activating implementations

Schema documents **must** reference a feature version which supports all the schema elements and behaviors required by the document. As a practical matter, authors should generally prefer to reference the [lowest](#sec-Ordering) versions which can [satisfy](#sec-Satisfaction) their requirements, as this imposes the fewest constraints on processors and is thus more likely to be supported. Authors **may** choose a higher version which they believe has sufficient support.

If a processor chooses to activate support for a feature, the processor **must** activate an implementation which can [satisfy](#sec-Satisfaction) the version required by the document.


# Extensibility

Any directives, scalars, enums, input objects, and other schema elements defined by specs **may** be **extensible**. This means that their definitions within a core schema document do not have to exactly match definitions provided by specs. Instead, definitions should match all usages within the document.

Standard extensibility rules:
- **Directives** may be defined as taking additional arguments, provided those arguments are [prefixed](#sec-Prefixing) with the name of a document feature followed by two underscores ({"__"}). These arguments are interpreted as metadata-on-metadata, and are routed to the declaring spec for interpretation
- **Enums** defined in features may include values of any name. For extensible enums, the space of **unprefixed** names belongs to the document. The space of **prefixed** names belongs to the spec with that prefix, and may be used to pass data between specs.
- **Input Objects** may be defined with additional fields, provided those fields are [prefixed](#sec-Prefixing) with the name of a feature followed by two underscores.

# Processing Schemas

```mermaid diagram
graph LR
  schema(["📄  Input Schema"]):::file-->proc("🤖 &nbsp;Processor")
  proc-->output(["📄  Output Schema"]):::file
  classDef file fill:none,color:#eee;
  style proc fill:none,stroke:fuchsia,color:fuchsia;
```

A common use case is that of a processor which consumes a valid input schema and generates an output schema.

The general guidance for processor behavior is: don't react to what you don't understand.

Specifically, processors:
  - **should** pass through {@core} directives which reference unknown feature URLs
  - **should** pass through prefixed directives, types, and other schema elements  

An exception to this is processors which prepare the schema for final public consumption. Such processors **may** choose to eliminate all unknown directives and prefixed types in order to hide schema implementation details within the published schema. This will impair the operation of tooling which relies on these directives—such tools will not be able to run on the output schema, so the benefits and costs of this kind of information hiding should be weighed carefully on a case-by-case basis.

# Validations &amp; Algorithms

## Bootstrapping (validates Has Core Feature)

Determine the name of the core specification within the document *or* fail the *Has Core Feature* validation.

It is possible to [rename the core feature](#sec-Renaming-core-itself) within a document. This process determines the actual name for the core feature if one is present.

Bootstrap(document) :
1. For each directive {d} on the SchemaDefinition within {document},
  1. If {d} has a `feature:` argument whose value is "https://lib.apollo.dev/core/v0.1", *and either*:
    - &#8230;{d} has an `as:` argument whose value is equal to {d}'s name
    - &#8230;*or* {d} does not have an `as:` argument and {d}'s name is `core`
    - *then* **Return** `coreName` = {d}'s name
- If no matching directive was found, the ***Has Core Feature* validation fails**.

## Feature Collection (validates Name Uniqueness)

Collect a map of ({featureName}: `String`) -> `Directive`, where `Directive` is a {@core} Directive which introduces the feature named {featureName} into the document.

CollectFeatures(document) :
  - Let {coreName} be the name of the core feature found via {Bootstrap(document)}
  - Let {features} be a map of {featureName}: `String` -> `Directive`, initially empty.
  - For each directive {d} named `coreName` on the SchemaDefinition within {document},
    - Let {name} be the spec's [name](#sec-Prefixing) as specified by the directive's `as:` argument or, if the argument is not present, the default name from the [feature url](#core__FeatureUrl).
    - If {name} exists within {features}, the ***Name Uniqueness* validation fails**.
    - Insert {name} => {d} into {features}
  - **Return** {features}


Prefixes, whether implicit or explicit, must be unique within a document. Valid:

===[Unique prefixes](prefixing.graphql#schema[0]) example

It is also valid to reference multiple versions of the same spec under different prefixes:

===[Explicit prefixes allow multiple versions of the same spec to coexist within a Document](prefix-uniqueness.graphql#schema[0]) example

Without the explicit `as:`, the above would be invalid:

===[Non-unique prefixes with multiple versions of the same spec](prefix-uniqueness.graphql#schema[1]) counter-example

Different specs with the same prefix are also invalid:

===[Different specs with non-unique prefixes](prefix-uniqueness.graphql#schema[2]) counter-example

## Assign Features

Create a map of {element}: *Any Named Element* -> {feature}: `Directive` | {null}, associating every named schema element within the document with a feature directive, or {null} if it is not associated with a feature.

AssignFeatures(document) :
  - Let {features} be the result of collecting features via {CollectFeatures(document)}
  - Let {assignments} be a map of ({element}: *Any Named Element*) -> {feature}: `Directive` | {null}, initally empty
  - For each named schema element {e} within the {document}
    - Let {name} be the name of the {e}
    - If {name} begins with {"__"},
      - Insert {e} => {null} into {assignments}
      - **Continue** to next {e}
    - If {name} contains the substring {"__"},
      - Partition {name} into `[`{prefix}, {base}`]` at the first {"__"} (that is, find the shortest {prefix} and longest {base} such that {name} = {prefix} + {"__"} + {base})
      - If {prefix} exists within {features}, insert {e} => {features}`[`{prefix}`]` into {assignments}
        - Else, insert {e} => {null} into {assignments}
      - **Continue** to next {e}
    - Insert `e => null` into {assignments}
  - **Return** {assignments}

## Is Exported?

Determine if any schema element is [exported](#sec-Parts-of-a-Core-Schema) to the API. A core schema's [API](#sec-Parts-of-a-Core-Schema) is always the subset of the entire Document containing only exported elements.

IsExported(element) :
  - Let {coreName} be the name of the core feature found via {Bootstrap(document)}
  - Let {assignments} be the result of assigning features to elements via {AssignFeatures(document)}
  - For each Directive {d} on {element},
    - If {d}'s name is {coreName}`__export`,
      - If {d} does not have an `isExport:` argument *or* `isExport:` is {true}, **Return** {true}
      - If {d} has an `isExport:` argument whose value is {false}, **Return** {false}
  - If {assignments}`[`{element}`]` is {null}, **Return** {true}
  - Let {feature} be the directive referenced from {assignments}`[`{element}`]`
  - If {feature} has a `export:` argument whose value is {true}, **Return** {true}
  - Else, **Return** {false}
