---
title: 'A Phase-Driven Model for Building Production AWS Systems with AI'
description: 'Lorem ipsum dolor sit amet'
pubDate: 'Mar 26 2026'
heroImage: '../../assets/phased-context-aware-sdlc.png'
---

Unstructured AI workflows collapse requirements, architecture, and implementation into a single prompt.

That ambiguity shows up directly in output:
- Infrastructure appears during requirements  
- Architecture shifts during implementation  
- Context grows while signal degrades  

This is not a prompting issue. It’s a lack of boundaries.

---

## Core Idea

**Context should be assembled, not written—and constrained by phase.**

Most AI workflows treat context as a static input.  
This approach treats it as something constructed dynamically from structured sources.

Instead of giving the model everything, the system:
- builds context incrementally  
- limits it to the current phase  
- injects only relevant domain knowledge  

The result is not just smaller prompts—it’s **bounded, more predictable behavior**.

---

## The Model

Every feature moves through a fixed sequence:

> Requirements → Architecture → Infrastructure → Implementation


The constraint is deliberate:

> The model is given only the context relevant to the current phase.

No architecture during requirements.  
No implementation during infrastructure.  

This is achieved through context isolation—not convention.

---

## System Design

### 1. Context Is Modular

Instead of a monolithic prompt, context is split into focused units:

- Core constraints (always loaded)  
- Phase definitions  
- Workflow rules  
- Domain guidelines  
- AWS service patterns  

Only a subset is assembled per run.

> Context becomes a build artifact, not an input.

It is constructed from structured sources based on phase and intent, rather than written ad hoc per interaction.

---

### 2. Context Is Phase-Bound

Each phase operates at a different level of abstraction:

| Phase          | Allowed Output                |
|----------------|------------------------------|
| Requirements   | Capabilities, constraints     |
| Architecture   | Flows, interfaces, schemas    |
| Infrastructure | CloudFormation, policies      |
| Implementation | Handlers, services, tests     |

The system withholds information that does not belong to the current phase.

This prevents phase leakage—the primary source of inconsistent outputs.

---

### 3. Context Is Service-Aware

AWS knowledge is not preloaded.

It is introduced incrementally based on feature intent.

Examples:
- “auth”, “login” → Cognito patterns  
- “api”, “endpoint” → API Gateway patterns  
- “queue”, “event” → SQS / EventBridge patterns  

This avoids contaminating early phases with implementation detail and keeps decisions scoped to the current level of abstraction.

> Infrastructure knowledge is introduced only when it becomes actionable.

---

### 4. State Lives in the Artifacts

State is not maintained in session memory. It is externalized in structured artifacts.

Each feature document captures:
- current phase  
- progress  
- decisions and changes  

This allows context to be rebuilt consistently across sessions without relying on hidden state.

The system remains stateless at runtime while preserving continuity through explicit, versioned artifacts.

---

### 5. Workflow Is Encoded in Context

Process constraints are embedded directly:

- No commits to `main`  
- Explicit approval before commit  
- Single-purpose changes only  

The model is guided by these constraints rather than inferring process implicitly.

---

## What Changes Across Phases

The same feature produces fundamentally different outputs depending on context.

### Requirements

Defines:
- functional scope  
- constraints  
- relevant AWS services (high level)  

No structure, no implementation.

---

### Architecture

Introduces:
- component boundaries  
- API contracts  
- data models  

Still no infrastructure.

---

### Infrastructure

Materializes:
- CloudFormation resources  
- security policies  
- deployment structure  

No application logic.

---

### Implementation

Completes:
- Lambda handlers  
- data access  
- validation and error handling  

At this point, most decisions are already made.

> Implementation shifts toward execution rather than exploration.

---

## Debugging with Change-Scoped Context

The same structure used for development can be applied to debugging.

Each feature introduces a bounded set of changes:
- infrastructure definitions  
- handlers and services  
- configuration  

When a regression appears after a feature is introduced, context can be limited to:
- the feature document  
- the current phase  
- files added or modified by that feature  

This reduces the debugging surface area to what actually changed.

---

### When This Works

This approach is effective when the issue is:
- introduced within the feature  
- contained within modified components  
- related to contracts, configuration, or local logic  

In these cases, limiting context improves signal and speeds up diagnosis.

---

### When It Does Not

Not all failures respect feature boundaries.

This approach breaks down when:
- the issue spans shared components  
- behavior emerges from multiple services  
- dependencies are indirect or implicit  

In these cases, context must expand beyond the feature.

---

### Practical Implication

Feature-scoped context is a useful default, not a guarantee.

It works best when paired with a simple escalation model:
- start with feature context  
- expand to system context if needed  

This mirrors how experienced engineers debug:  
start with what changed, then widen the scope.

---

## Observations

### Phase Separation Is the Lever

Most inconsistencies in AI output come from mixing abstraction levels.

Strict phase boundaries reduce:
- premature decisions  
- conflicting outputs  
- rework  

---

### Context Relevance > Context Size

Reducing context size helps, but that’s a side effect.

The real gain comes from:
> removing information that does not belong to the current phase

---

### Determinism Comes from Constraints

Flexible AI workflows feel powerful but introduce drift.

Constrained workflows:
- reduce ambiguity  
- improve repeatability  
- produce more predictable outcomes  

---

### State Must Be Explicit

When state is implicit, sessions reset and context must be reconstructed manually.

Externalizing state into artifacts:
- preserves continuity  
- enables reproducibility  
- makes progress inspectable  

---

## Closing

AI does not need better prompts.

It needs:
- clear phase boundaries  
- controlled context  
- explicit state  
- enforced workflows  

This is not a new paradigm—it is standard engineering discipline applied to AI interaction.

The outcome is not just better output. It is **predictable systems**.

> Good infrastructure is built in phases.  
> AI should be constrained to follow them.
