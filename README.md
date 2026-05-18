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
auditable than prompt-level context construction.

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

## Relationship to Existing Tools

| Tool | Relationship |
|---|---|
| **ATL** | Closest academic peer. ATL is mature, OCL-based, Eclipse-centric. NSML differs in pluggable expression languages, late-binding, AI-friendly tooling, and workflow integration. |
| **QVT** | OMG standard. Both Operational and Relations exist; NSML is closer to Operational in pragmatics, closer to declarative ATL in style. |
| **Henshin** | Graph-transformation-based, visual rules, in-place. NSML is rule-based but not graph-transformation; rules are templates, not graph-rewriting productions. |
| **VIATRA** | Reactive incremental transformation with sophisticated pattern matching. NSML does not currently target incremental evaluation; it is a possible future direction. |
| **Epsilon (ETL)** | Closest in spirit - declarative, pragmatic, Eclipse-hosted. NSML's pluggable expression languages and AI-era tooling are the main differentiation. |
| **DataWeave** | Closest commercial peer (MuleSoft). DataWeave is JSON/XML-focused, not Ecore-focused. NSML targets the model-driven engineering audience DataWeave does not. |
| **XSLT** | The conceptual ancestor. NSML applies the match-and-template idea to Ecore instead of XML, with a different and pluggable expression language layer instead of XPath alone. |

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
