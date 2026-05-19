# NSML Execution Semantics - Tokens as the Runtime Substrate

## 1. Overview

NSML is declarative - it describes *what* maps to *what* via rules. 
Execution is a separate concern. 
This document names **`org.nasdanika.token`** as the runtime substrate for executing NSML mappings, and lays out the semantics that follow from that choice.

The [token module](https://github.com/Nasdanika/token) provides generic primitives for tracking execution: an immutable position-and-parents record, 
with composable facets for state, editing-domain command execution, and OpenTelemetry tracing. 
NSML execution is one of the use cases that motivated the module's extraction from the OpGraph package - and is covered from the token side.

## 2. Execution model

An NSML execution is a tree of tokens of type:

```java
RealmStateToken<MappingStep, MappingContext>
```

- **`MappingStep`** records a single rule firing as `(EObject source, MappingRule rule)`.
- **`MappingContext`** holds the in-progress mapping: target `ResourceSet`, source→target bindings map, accumulated diagnostics, provenance index.
- **`RealmStateToken`** combines `RealmToken` and `StateToken`. Writes to the context go through commands (`execute`/`apply`) so they obey the editing-domain invariants - typically an EMF Transaction editing domain wrapping the target `ResourceSet`. Reads of effectively-immutable parts of the context (bindings map at a given step) are safe directly via `getState()`.

The root token represents engine startup. Every rule firing is a `commit`. Joins - for example, a rule that depends on the outputs of two prior rules - are `merge` commits. The token tree at the end of execution is the complete provenance graph: every target element can be traced back through tokens to the rule firings and source elements that produced it.

```
root token ── commit ──▶ rule A on source X ──▶ rule B on source Y ──┐
                       └─ commit ──▶ rule C on source Z ─────────────┴──▶ merge ──▶ join rule D
```

## 3. Provenance

Every target `EObject` carries a reference back to the token that produced it - implemented as an EMF `Adapter`, an `EAnnotation`, or a side-index in the `MappingContext`, depending on whether the target model has its own annotation conventions. Two practical consequences:

- **"Why does this target element exist?"** is answered by walking token parents. The answer is structural, not reconstructed from logs.
- **Diff-aware re-execution.** If the source model changes, tokens whose source elements are unchanged are still valid and can be reused; only the affected subtree needs re-running. This is the same idea as Bazel's content-based incremental build, applied to model-to-model mapping. It also gives "what changed in the target if I change this in the source?" cheaply.

## 4. Async rule bodies and AI agents

A rule body can be synchronous or asynchronous. Synchronous rules use `RealmToken#execute` / `apply`; asynchronous rules - typically those that invoke an AI agent or another remote service - use `executeAsync` / `applyAsync`, which return `Mono<Void>` / `Mono<T>` and integrate naturally with Project Reactor pipelines.

Three properties matter for AI-backed rules and all come from the token substrate without any NSML-specific machinery:

1. **Asynchrony with backpressure.** Agents are I/O-bound. `applyAsync` returns a cold `Mono` so fan-out (`Flux.merge`, `Flux.concatMap`) is governed by Reactor's standard backpressure mechanics - the engine can cap concurrent agent calls without writing a thread pool.
2. **Retry from a known-good state.** A rule that gives a low-confidence answer, fails, or returns an invalid target produces a token like any other - but on the audit trail, not in the target model (writes via `RealmToken#execute` can be conditioned on a confidence check). Retries are siblings under the same parent token, with different prompts, temperatures, or models. The audit trail retains both the rejected and the accepted attempt.
3. **Observable cost and latency.** Each rule firing is a span. AI-backed rules emit child spans with OpenTelemetry gen-ai semantic-convention attributes (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reasons`). Aggregating across a mapping run yields the per-rule and total AI cost of producing the target model.

A worked example of an AI-backed rule:

```java
class ChooseTargetConceptRule implements MappingRule {

    @Override
    public Mono<Void> applyAsync(
            RealmStateToken<MappingStep, MappingContext> token,
            EObject source) {

        return token.applyAsync((self, ctx) -> agent
            .chooseTargetConcept(source, ctx)
            .doOnNext(answer -> ctx.recordTelemetry(self, answer))
            .flatMap(answer -> answer.confidence() >= 0.6
                ? Mono.just(answer.targetConcept())
                : Mono.error(new LowConfidenceException(answer))))
            .doOnNext(targetConcept -> self.execute((t, c) -> {
                var targetEObject = instantiate(targetConcept);
                c.bind(source, targetEObject);
                c.annotateProvenance(targetEObject, t);
            }))
            .then();
    }
}
```

Retry on `LowConfidenceException` is the engine's job, not the rule body's. The engine creates a sibling token from the parent with a higher-temperature (or different-model) variant of the rule, attempts again, and proceeds. The old branch stays in the token tree for audit.

## 5. Declarative and agentic rules - uniform shape

A deterministic NSML rule ("if `UMLClass`, create `OOPClass`") and an agentic rule ("ask the model which target concept fits this source") produce *identical* execution records - same lineage, same telemetry, same retry semantics. The only difference is in the rule body. That uniformity is the consequence of choosing tokens as the substrate: declarative and agentic mappings are not different kinds of system, only different kinds of rule.

The practical payoff: tooling that reads provenance, replays executions, computes cost, or compares two runs needs to know about tokens and rule metadata only - never whether a given step was deterministic or agentic. That distinction belongs in the rule body and nowhere else.

## 6. Parallelism

Independent rule firings - rules whose source elements are unrelated and whose target writes are non-conflicting - run in parallel:

```java
Flux.fromIterable(sourceElements)
    .flatMap(src -> token.applyAsync((t, ctx) -> applyRule(t, src, ctx)),
             concurrency)
    .collectList()
    .flatMap(childTokens -> Mono.just(
        token.merge(joinStep, asStateTokens(childTokens))));
```

Joins are explicit `merge` calls; their state is computed from the joining parents via `mapAsync`. The realm (the target `ResourceSet`'s editing domain) serializes writes regardless of how many parallel rule firings are in flight, so concurrency is bounded by I/O on the agent side, not by lock contention on the target model.

## 7. Persistence and resumption

When the `org.nasdanika.token.git` submodule is in play, each token is committed to a Git repository (element + state serialized as blobs, parents as Git parents). NSML execution becomes resumable: the engine can crash, restart, and pick up from the last committed token. Long-running mappings - especially those with expensive AI calls - get this property for free, and the audit trail becomes a real Git history that can be browsed with standard tooling.

## 8. Engine architecture

The engine itself is small. It does, in order:

1. Build a plan from the NSML mapping definition - which rules apply where, in what dependency order.
2. Walk the source model, producing a `Flux<MappingStep>` of pending rule firings.
3. For each step, fire the rule body against the current token (`execute` for sync, `applyAsync` for async / agentic), producing the next token.
4. Handle exceptions: on `LowConfidenceException` (or whichever class the rule declares retryable), branch from the parent with a variant; on hard failure, propagate.
5. At rule completion points that depend on multiple inputs, `merge`.
6. At end-of-run, the root-to-leaves token tree is the provenance graph; the target `ResourceSet` is the artifact; the OTEL trace tree is the run's observability record.

None of these steps is NSML-specific - they're the same shape any token-based execution engine would have. NSML supplies the rule definitions and the planner; the engine is a generic token walker.

## 9. Open questions specific to NSML

1. **Where does the provenance reference live?** EAnnotation per target element keeps everything in the target resource but pollutes its schema; an EMF Adapter is invisible to serializers but transient; a side-index in `MappingContext` is portable but extra plumbing. *Recommendation: side-index by default, with optional EAnnotation emission for downstream tools that need it.*
2. **Engine-level vs. rule-level retry policy.** Should low-confidence handling and retry budgets be declared in NSML rule metadata, or in the engine configuration? *Recommendation: both - rule metadata declares acceptable confidence thresholds and which exception classes are retryable; engine configuration controls overall retry budget and prompt-variation strategy.*
3. **Should the bindings map be `MappingContext` or a separate `RealmToken`?** If two unrelated rule subtrees write to disjoint parts of the target model, splitting the realm gives more parallelism. *Almost certainly not worth it for v1; revisit if write contention shows up under real load.*
4. **Cost-bounded execution.** Should the engine support a "stop at $X" budget for AI costs across a run? Easy to add as an OTEL `SpanExporter` or token-level handler that watches `gen_ai.usage.*` attributes and signals cancellation when exceeded. *Worth doing for production use; defer until a real use case demands it.*
5. **Caching agent calls.** A `MappingStep` plus a hash of the relevant `MappingContext` slice forms a natural cache key. If the same step recurs with the same inputs, the prior agent answer can be reused. *Worth designing carefully - silent reuse of stale model outputs is a footgun; opt-in per rule.*

## 10. Cross-references

- Token module design: [token-module-design.md](./token-module-design.md), in particular §6.3 (`RealmToken`), §6.4 (`TelemetryToken`), §8 (combined facets), §10.4 (`emf.transaction` submodule), §11 (mapping/transformation case study), §12 (rollback and retry).
- Note that `org.nasdanika.token.nsml` (§11.5 of the token doc) lives in the token module's repository, not in NSML's. NSML depends on it; the inverse is not true. This keeps NSML's dependency surface small for downstream consumers who only care about the language definition and not the execution engine.
