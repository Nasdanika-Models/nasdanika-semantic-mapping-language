# NSML - Nasdanika Semantic Mapping Language

## Overview

NSML is a declarative, model-driven language for transforming and mapping EMF Ecore
models. It applies the XSLT-style match-and-transform pattern to Ecore models, with
a pluggable expression language layer (SpEL, Eclipse Model XPath, JXPath, Groovy,
and others) selected per expression.

NSML is designed to address three usage patterns:

1. **Standalone model transformation** - Ecore-to-Ecore, Ecore-to-application-model,
   model-to-site generation, executed from CLI or embedded in build pipelines.
2. **Workflow context mapping** - [OpGraph](https://op-graph.models.nasdanika.org/) [operators](https://op-graph.models.nasdanika.org/references/eClassifiers/Operator/index.html) use NSML transformations as their
   `map` / `flatMap` declarations to project execution context for reuse, focus, and
   agent semantic boundaries.
3. **Documentation and explainability** - transformation models are diagrammable,
   inspectable, and amenable to AI generation and review, making them suitable as
   communication artifacts between SMEs, developers, and AI agents.

Mapping definitions can be compiled to Java as targets for the [`org.nasdanika.common.Transformer`](https://github.com/Nasdanika/core/blob/master/common/src/main/java/org/nasdanika/common/Transformer.java) - providing composability with hand-written Java transformations.
Mapping definitions can map to an Ecore model defined elsewhere or be a mapping + Ecore model definition in one.

## Design Principles

### 1. Declarative Match-and-Transform

A transformation is a set of **rules**. Each rule has:

- A **match** - an EClassifier identified by namespace URI and classifier name.
- An optional **condition** - a boolean expression that the matched object must
  satisfy.
- A **produce** clause - an EClassifier identified by namespace URI and classifier
  name, with feature values computed from expressions.

The engine evaluates rules against input objects, applying matching rules to produce
output objects. This is the XSLT model applied to Ecore: matching by type and
condition, producing by template.

### 2. Late-Bound Type References

Rule matches and produces reference EClassifiers by **namespace URI and classifier
name as strings**, not by compile-time EClass references. This matters because:

- AI agents produce model fragments without compile-time access to metamodels.
- Cross-version transformations may target metamodels not present at the
  transformation's authoring time.
- Transformations can be authored against a metamodel by reference, without that
  metamodel being on the classpath during authoring.

Late binding does not preclude validation. When a metamodel is available, NSML
validates references against it. When it is not, NSML defers validation to runtime.

### 3. Pluggable Expression Languages

NSML does not have a built-in expression language. Every expression - conditions,
feature value computations, computed URIs and classifier names - is a string in the
form `<language>:<expression>` or `<language>@<relative expression resource uri>`. 
The language identifier resolves to a registered
**expression evaluator** that parses and evaluates the expression.

Default registered languages:

- **`spel`** - Spring Expression Language. Familiar to most Java developers. Strong
  for property access, method invocation, and conditional logic.
- **`expath`** - Eclipse Model XPath (`org.eclipse.e4.emf.xpath`). XPath-style
  navigation over EMF models. Familiar to anyone who has worked with XML.
- **`jxpath`** - Apache Commons JXPath. Generic XPath-like queries over Java object
  graphs.
- **`groovy`** - full Groovy expressions. Suitable for complex logic.

Additional evaluators can be registered via the capability model and as JSR-223 script
engines.

### 4. Pluggable Discovery via JSR-223 (Recommended)

Expression evaluators are discovered through Java's standard
`javax.script.ScriptEngineFactory` mechanism (JSR-223) plus a thin NSML-specific
interface for model-aware evaluation. This means:

- Evaluators register themselves via `ServiceLoader` - no central registry.
- The standard `Compilable` interface enables pre-compilation of expressions.
- The standard `Bindings` interface handles context propagation.
- Third parties can add expression languages without modifying NSML.

SpEL, JXPath, and Eclipse Model XPath are wrapped as JSR-223 engines by NSML.

### 5. Expression Externalization

An expression may be inline or external. The form is:

```
<language>: <inline-expression>
```

When the content after `<language>:` is a URI with a scheme the language handler
recognizes, it is loaded from that URI. Otherwise, it is parsed as an inline
expression. Examples:

```
spel:state.tickets.?[priority > 5].size()
expath:/tickets[priority > 5]
groovy@classpath:/scripts/categorize.groovy
groovy@scripts/categorize.groovy
spel@classpath:/expressions/priority-filter.spel
```

Externalization is the same mechanism for every language - the language handler
fetches the URI and parses its content. Inline expressions remain short and readable
in YAML; complex expressions move to separate files with proper syntax highlighting,
testing, and version control.

### 6. Compilation to Transformer

An NSML transformation compiles to a target of `org.nasdanika.common.Transformer`. 
This means:

- Compiled transformations are ordinary Java code, fast and statically integrable.
- Transformations compose with hand-written Transformer implementations.
- Transformations can be packaged as Maven artifacts and consumed like any other
  Java library.
- The fallback path - interpretation when full compilation is not possible.

The compilation target is described at:
https://github.com/Nasdanika/core/blob/master/common/src/main/java/org/nasdanika/common/Transformer.java
and https://medium.com/nasdanika/documenting-json-schemas-15e3bd690c33.

### 7. Operations as first-class transformation targets

Ecore is a metamodel rather than a pure data format. Classifiers carry **structural features** (attributes and references) **and operations** - typed methods with parameters and return types. XSLT-style transformation languages historically address only structure because XML has only structure; NSML targets Ecore and therefore treats operations as first-class transformation targets alongside features.

A rule may:

- **Hide an operation** that exists on the source classifier - the produced classifier exposes a strict subset of the source's interface.
- **Modify the behavior of an operation** - the produced classifier's operation delegates to a different implementation than the source's, declared through the same expression mechanism used for feature values.
- **Introduce an operation** that does not exist on the source - the produced classifier exposes behavior the source does not, computed from the source's state.
- **Bind parameters** - the produced operation pre-supplies values for one or more of the source operation's parameters, narrowing its signature. The analogy is JavaScript's `Function.prototype.bind()`, with two differences. First, JavaScript `bind` captures the value passed at bind time and freezes it for every subsequent call; NSML binds a parameter to an **expression** that is re-evaluated at invocation time **in the context of the source object**, so the bound value can depend on the state of the source when the operation is called rather than being fixed when the transformation was authored. Second, JavaScript `bind` only pre-supplies a consecutive run of leading parameters; NSML can bind **any parameter, by name or position**, leaving the remaining parameters exposed.
The implementation behind a transformed or introduced operation is declared the same way feature values are: as a typed expression in one of the registered languages, with the same externalization options. The expression may invoke a hand-written Java method, a service, another transformation, or - the case that distinguishes NSML's operation support most clearly from XSLT-era languages - an **agent**. An operation introduced by a transformation and backed by an LLM agent operating over the produced view turns the view into a complete agent contract: what the agent sees (the produced data) and what the agent does (the produced operations) are both declared in the same artifact.

### 8. Privacy and access control through transformation

The federated model assumed by NSML's other use cases is open by default. A complementary modality is the *protected* federation: a model assembled in a controlled environment - private repositories, on-prem stores, or a local model that references external public artifacts - over which an access policy is applied before the model is shared with any consumer.

NSML's slicing mechanism is the natural vehicle for that policy. An access-control transformation expressed in NSML:

- Suppresses elements the principal is not permitted to read.
- Suppresses features and operations the principal is not permitted to access.
- Optionally renames or generalizes elements where the existence of the element is visible to the principal but its identity is not.

Policies compose with [Apache Shiro](https://shiro.models.nasdanika.org/) primitives - subjects, roles, groups, permissions - where element and feature URIs serve as the permission strings. A subject's allowed view is the transformation of the full model that retains exactly the elements and features the subject's permissions admit.

The transformation produces a self-contained model - same metamodel conformance, same NSML and OpGraph tooling - that can be served from a Web UI session, published to a repository the principal can read, or rendered as a static site for offline distribution. **Agent semantic contexts derived from a transformed view inherit the same access bounds by construction:** an agent cannot reason over elements the transformation removed. Confidentiality is a property of the view, not a property the agent has to be trusted to respect.

## The Transformation Model

NSML transformations are themselves Ecore models. 

### Core Concepts

| Concept | Description |
|---|---|
| `Transformation` | Top-level container. Holds rules, imports, and metadata. |
| `Rule` | Match-and-produce unit. Has a match, optional condition, optional priority, and a produce clause. |
| `Match` | Specifies the input element type by namespace URI and classifier name, plus an optional condition expression. |
| `Produce` | Specifies the output element type by namespace URI and classifier name (each may itself be a computed expression), plus feature value mappings. |
| `FeatureMapping` | Pairs an output feature name with an expression that computes its value. Output feature names may also be computed expressions. |
| `Import` | References another NSML transformation for composition and reuse. |

### Rule Resolution

When the engine applies a transformation to an input element, it:

1. Evaluates each rule's `match` against the input element's type and properties.
2. Filters by the rule's `condition` expression if present.
3. Resolves conflicts among matching rules by explicit priority, then by specificity
   (more specific matches win), then by declaration order as a final tiebreaker.
4. Applies the winning rule's `produce` clause, evaluating each feature mapping
   expression in the context of the input element.
5. Recursively applies the transformation to feature values that are themselves
   EObjects, allowing transformation of object graphs.

### Composition

A transformation may import others. Imported transformations contribute their rules
to the importing transformation's rule set. Rule resolution then operates over the
combined set, with the importing transformation's rules taking precedence over
imported rules at the same specificity.

This enables building a library of reusable mapping fragments - e.g., a "common Jira
field mappings" transformation imported by multiple workflow-specific transformations.

## Surfaces

NSML exposes several surfaces over the same underlying model:

### 1. YAML / JSON (via EMF JSON)

The default authoring format. Suitable for hand-editing, version control, and AI
generation. Example:

```yaml
transformation:
  name: jira-to-task
  rules:
    - match:
        uri: "http://jira.example.com/issues"
        classifier: "Issue"
      condition: "spel: type == 'Task' and status != 'Closed'"
      produce:
        uri: "http://tasks.example.com"
        classifier: "Task"
        features:
          - name: id
            value: "spel: key"
          - name: summary
            value: "spel: fields.summary"
          - name: priority
            value: "spel: fields.priority?.name ?: 'Normal'"
```

### 2. Draw.io Diagrams

Transformations can be authored as Draw.io diagrams using conventions. 
Rules become diagram elements; matches and produces
are visualized as labeled boxes with feature mappings as edges; nested transformations
appear as nested pages.

No proprietary editor is required. Draw.io is the tool; NSML provides the conventions
and a small user library of stencils.

Bidirectionally, an NSML transformation can be **rendered** to a Draw.io diagram with
auto-layout via [ELK](https://elk.models.nasdanika.org/). 
This is the documentation path: text-authored transformations
become inspectable diagrams without manual diagramming work.

### 3. XText DSL (planned)

A textual DSL with IDE support - syntax highlighting, content assist, rule reference
resolution, expression-language-aware completion within `spel:`, `expath:`, etc.
prefixes. Built when the YAML form's limits are felt.

### 4. AI Elicitor

A reusable elicitor - chat or prompt-driven - that produces valid NSML models from
natural-language descriptions of mappings. The elicitor is not NSML-specific; it is a
reusable Nasdanika capability that takes any Ecore metamodel and produces conforming
instances. NSML is one consumer; other Nasdanika models are others.

The elicitor produces YAML / JSON form by default, optionally diagram form, optionally
both. The output is always a valid model - the elicitor validates against the
metamodel before returning.

### 5. CLI

A command-line tool to:

- Execute a transformation against an input model and produce an output model.
- Compile a transformation to a Java Transformer source file.
- Validate a transformation against its declared input and output metamodels.
- Render a transformation as a Draw.io diagram.
- Generate documentation from a transformation.
- Chain transformations in pipelines.

Example:

```
nsml transform --rules jira-to-task.nsml --input jira-export.xmi --output tasks.xmi
nsml compile --rules jira-to-task.nsml --output JiraToTaskTransformer.java
nsml render --rules jira-to-task.nsml --output jira-to-task.drawio
```

## Use Cases

### UC-1: Model-to-Application-Model-to-Site

A common Nasdanika pattern is generating web sites from domain models. NSML supports
the staged pipeline:

```
domain.xmi  ->  [NSML transformation] ->  app-model.xmi  ->  [HTML generator]  ->  site
```

The intermediate
application model (e.g., from `html-app.models.nasdanika.org/`) is fully inspectable
and itself a model artifact.

### UC-2: OpGraph Operator Context Mapping

An OpGraph operator declares its `map` or `flatMap` as an NSML transformation. The
operator's downstream execution context is computed by the transformation, with
compile-time type generation and runtime adapter delegation. See the OpGraph NSML
integration addendum for detail.

### UC-3: Agent Semantic Context Definition

An AI agent operating on a complex domain model is given a narrowed semantic context
defined by an NSML transformation. The transformation:

- Projects only the entities the agent should see.
- Renames fields to terms familiar to the agent's persona (e.g., engineer vs.
  product-manager vocabulary).
- Computes derived fields that the agent needs but that don't exist on the source.
- Hides fields the agent must not see.

The transformation is reviewable, testable, and version-controlled - far more
auditable than prompt-level context construction. When the transformation is also
the subject's access policy (see UC-6), the agent's context is bounded by the same
policy as the human consumer's view, and confidentiality is inherited by construction
rather than enforced by trust.

### UC-4: Cross-Version Model Migration

A metamodel evolves; existing model instances must be migrated. An NSML transformation
declares the migration rules. Because matches are by namespace URI and classifier
name, the transformation can reference both the old and new metamodels without
either being on the compile classpath of the other.

### UC-5: Ad-Hoc Data Shaping

A developer or analyst has data in one Ecore-modeled form and needs it in another for
a one-off purpose. NSML provides an interactive workflow: open the input in a tool,
sketch a transformation in YAML or via the elicitor, run it through the CLI, inspect
the output. No code, no compilation, no project setup.

With the [SQL metadata model](https://sql.models.nasdanika.org/) it can be done for databases,
including data migration or defining a data access layer.

### UC-6: Access-Controlled Federated Views

A federated model assembled in a protected environment combines elements from many
sources - some public, some restricted to specific teams, some sensitive to particular
stakeholders. An NSML transformation expressing the access policy of a given subject
produces the view that subject is permitted to see, drawing from the full federation.

The transformation is itself an artifact - version-controlled, reviewable, testable.
Compliance teams audit the policy by reading the transformation; security teams run
conformance tests against it. The produced view is a self-contained model that ships
through the standard Nasdanika tooling: served interactively from a Web UI session,
published to a repository the principal can read, or rendered as a static site for
offline distribution. Agent semantic contexts derived from the produced view operate
under the same access bounds by construction.

This is the foundation that makes NSML usable in environments where total openness
is not an option: organizations with strategic confidentiality requirements, vendors
maintaining customer-specific federations, and portals where each consumer authors
their own personas privately while consuming shared capability declarations.

### UC-7: Agent-Backed Operations

A view produced by an NSML transformation exposes data the consumer is allowed to
see and operations the consumer is allowed to invoke. Some of those operations are
backed by agents: "summarize this for my role," "find elements like this for a
different persona," "compare these two proposals on the dimensions that matter to me."

The transformation declares each operation, its typed signature, its access
constraints, and the agent (or pipeline) that implements it. The agent operates over
the produced view - that is, under the same data access bounds as the human consumer.
The view becomes a complete contract for both human and machine interaction: data
plus behavior, both authored as part of the same NSML transformation, both
inspectable and citable from the model.

### UC-8: Resource Content Filters - Filename-Driven Loading Pipelines

EMF's `Resource.Factory` mechanism dispatches by file extension to produce model
content.
The Nasdanika extension generalizes this: a filename can declare a *chain* of extensions, each registered as a `ResourceContentFilter` capability that
transforms the previous stage's model into the next.
A file named `my-product.pm.md` loads as Markdown (the `.md` factory), then is mapped to the
Product Management model (the `.pm` filter); a file named
`internet-banking-system.c4.drawio` loads as a draw.io diagram, then is mapped to
a C4 model; a file named `adams.family.xlsx` loads as Excel, then is mapped to
a family model.
The client code calls `ResourceSet.getResource(uri)` and receives
the target model directly, with the conversion machinery hidden behind the
standard EMF abstraction.

NSML is the natural implementation substrate for these filters. Each filter is
declared as an NSML transformation - inputs are the previous stage's model,
outputs are the target model, mapping rules express the conversion.
Writing a new filter becomes writing a new transformation rather than writing custom Java
loader code. Cross-format filters compose freely: a draw.io file can reference
Markdown files (via prototype references), and the Markdown files are loaded
through their own filter chain before being merged into the diagram's mapped
output.
Enrichment from external systems - resolving references against a live
service, fetching authoritative data - is handled by NSML rules invoking external
expression evaluators during the transformation; the source filename does not
change, but the resolved model carries the enriched content.

Bi-directional filters are possible where the target model can be projected back
to the source format.
For diagrams, geometry-preserving back-projection is the
canonical hard case: changes to the C4 model that correspond to existing diagram
elements preserve those elements' geometry, while changes that introduce new
elements get geometry from ELK auto-layout.
The bi-directionality is hidden behind the same filter contract - `Resource.save()` writes back through the chain
when possible, with explicit diagnostics when it is not.

The pattern composes with CLI command pipelines. A CLI invocation like `nsd model internet-banking-system.c4.md html-app site` runs the filename pipeline
(`.md` to Markdown to C4) to produce the input for the command pipeline (`html-app` to `site`).
Two grammars meet at the model boundary: filename grammar produces the typed model, command grammar consumes and transforms it.
The combined sentence is a single CLI invocation that hides arbitrary depth of loading and processing behind a uniform interface.

A `ResourceContentFilter` is also the natural place to apply *access control and
projection*.
The filter receives a security principal through the capability
framework as a requirement and produces a target model that contains only the elements the
principal is entitled to see.
The principal can reach the framework two ways, and well-designed filters support both transparently:

- **Ambient discovery** for single-principal contexts (CLI tools, build pipelines,
  single-user services). The capability resolver discovers the principal from
  the OS user name, an environment variable, a JAAS subject, an OAuth token in
  the process context, or any other provider registered against the
  principal-source capability. The consumer code calls
  `ResourceSet.getResource(uri)` without supplying anything; the principal is
  acquired implicitly.
- **Explicit binding** for multi-principal contexts (HTTP servers, gRPC services,
  MCP server endpoints, anything serving concurrent per-request authentication).
  The consumer supplies the principal as a *requirement* when constructing the
  `ResourceSet` - the resource set is built *for* that principal, and filters
  loading content within it resolve the principal capability against the bound
  requirement rather than against ambient state. A request handler resolves the
  principal from the request (OAuth token, mTLS certificate, session token,
  whatever the protocol carries), constructs a per-request resource set for that
  principal, loads resources within it, returns the projected content, and
  discards the resource set when the request completes. Concurrent requests with
  different principals do not interfere because each operates in its own
  resource-set scope; the filter implementation does not change between
  single-principal and multi-principal deployments.

Different consumers loading the same source file receive different target
models; the consumer code in both cases calls `ResourceSet.getResource(uri)`
without knowledge of the projection. This is the deployment shape of
[UC-6: Access-Controlled Federated Views](#uc-6-access-controlled-federated-views)
and [UC-7: Agent-Backed Operations](#uc-7-agent-backed-operations) at the
resource-loading boundary: the same content can serve a senior leader's view, a
delivery team's view, a partner's view, or an AI agent's view, with the access
policy expressed as an NSML transformation rather than enforced through ad-hoc
checks scattered through application code. An agent reasoning over a
filter-projected resource sees only what the resource exposes, so the agent's
data-access boundary is identical to the user's by construction rather than by
trust.

## Compilation Pipeline

```
NSML transformation (YAML / JSON / DSL / diagram)
    ↓
Ecore model (canonical form)
    ↓
Static analysis (input/output metamodel subsets, expression validation)
    ↓
Compile to Transformer (Java)        
    ↓
Package as Maven artifact            ← discoverable via ServiceLoader
```

Compilation is optional. An NSML transformation can be executed by the interpreter
directly without any compilation step - useful for development, ad-hoc use, and
contexts where the transformation evolves frequently. Compilation is for production
performance and for integration with other compiled code.

## Roadmap

### Phase 1 - Engine

- Core metamodel
- Interpreter with SpEL, JxPath, Groovy and Eclipse Model XPath evaluators
- JSON / YAML loading via EMF JSON
- Basic CLI (transform, validate)

### Phase 2 - Compilation

- Compilation to Transformer target

### Phase 3 - Tooling

- Draw.io diagram generation (render)
- Documentation generation
- AI elicitor (chat → NSML)

### Phase 4 - Authoring

- Draw.io diagram-to-NSML conventions
- XText DSL with IDE support

### Phase 5 - Operations and Access Control

- Operation transformation: hide, redirect, introduce
- Agent-backed operation bindings
- Apache Shiro integration: subjects, roles, groups, URI-based permissions
- Access policy authoring as NSML transformations
- Conformance tests for access policies

## Relationship to Existing Tools

| Tool | Relationship |
|---|---|
| **ATL** | Closest academic peer. ATL is mature, OCL-based, Eclipse-centric. NSML differs in pluggable expression languages, late-binding, AI-friendly tooling, and workflow integration. |
| **QVT** | OMG standard. Both Operational and Relations exist; NSML is closer to Operational in pragmatics, closer to declarative ATL in style. |
| **Henshin** | Graph-transformation-based, visual rules, in-place. NSML is rule-based but not graph-transformation; rules are templates, not graph-rewriting productions. |
| **VIATRA** | Reactive incremental transformation with sophisticated pattern matching. NSML does not currently target incremental evaluation; it is a possible future direction. |
| **Epsilon (ETL)** | Closest in spirit - declarative, pragmatic, Eclipse-hosted. NSML's pluggable expression languages and AI-era tooling are the main differentiation. |
| **DataWeave** | Closest commercial peer (MuleSoft). DataWeave is JSON/XML-focused, not Ecore-focused. NSML targets the model-driven engineering audience DataWeave does not. |
| **XSLT** | The conceptual ancestor. NSML applies the match-and-template idea to Ecore instead of XML, with a pluggable expression language layer in place of XPath alone. Because Ecore - unlike XML - has operations, NSML extends the XSLT model to transform operations (hide, redirect, introduce) as well as features, with operation backings that may include agents. |

## Open Design Questions

1. **Externalization syntax:** `<language>:<URI>` parsed by the language handler, vs.
   `<language>@<URI>` as a distinct syntactic form. The former is simpler; the latter
   is more explicit.
2. **Rule priority and specificity:** Is priority always explicit, or is specificity
   computed automatically (more specific matches win)? XSLT does the latter; the
   trade-off is predictability vs. concision.
3. **Recursive transformation control:** When a feature mapping produces an EObject
   that itself matches rules, should the transformation recurse automatically (XSLT
   apply-templates), only on explicit declaration, or both modes with a per-rule flag?
4. **Expression evaluator capability negotiation:** Different evaluators have
   different capabilities (compilable, sandboxed, reactive). How does NSML query
   and adapt to these?
5. **Conflict resolution for imports:** When two imported transformations have rules
   that match the same input, which wins? Explicit precedence, import order,
   specificity, or an error?
6. **Operation backing implementations:** A rule that introduces or modifies an
   operation declares the backing as a typed expression. Should the same set of
   expression languages be admissible for operation backings as for feature values,
   or should backings be restricted (e.g., to registered services, named agents, and
   Java methods, excluding ad-hoc Groovy)? The trade-off is authoring flexibility
   versus operational safety.
7. **Access policy composition:** When a subject has multiple roles and groups, each
   contributing an access policy expressed as an NSML transformation, how are the
   policies composed? Intersection (most restrictive) by default with explicit
   override, union (most permissive), or a higher-order transformation that combines
   them? How is the composed policy itself audited?
8. **Operation visibility under access control:** An operation may be visible to the
   consumer but invocable only by a subset of subjects. Is operation invocability a
   separate permission from operation visibility, or is the visible-but-not-invocable
   case expressed by declaring two operations - one read-only, one privileged?
