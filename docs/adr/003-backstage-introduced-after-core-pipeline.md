# ADR-003: Introduce Backstage After the Core Pipeline Works Manually

## Status
Accepted

## Context
Backstage is the developer portal component of this platform, providing the Software
Catalog, Software Templates (scaffolding), and TechDocs. It is also, by a wide margin,
the most complex single tool in this project's stack: it is itself a full application
(a Node.js backend + React frontend) with its own plugin architecture, authentication
configuration, and YAML-based templating engine.

The original phase ordering placed Backstage relatively early (as "the portal"), which
would mean integrating a complex new tool against infrastructure (CI, registry, GitOps,
Kubernetes) that does not yet exist or is not yet proven to work.

## Decision
Build and manually verify the core pipeline first — Kubernetes deployment (Phase 3),
Helm packaging (Phase 4), CI (Phase 5), registry (Phase 6), and GitOps/Argo CD (Phase 7)
— using direct commands and manual repo creation, before introducing Backstage in
Phase 8. Backstage is then layered on top of infrastructure that is already known-good.

## Reasoning
1. **Isolates failure domains.** If something breaks after Backstage exists, we need to
   be able to answer "is this a Backstage integration problem, or is the underlying
   pipeline broken?" That question is only answerable cheaply if the underlying pipeline
   was already proven to work independently of Backstage.
2. **Backstage's value is integration, not creation.** Backstage's Software Templates
   don't invent CI/CD or Kubernetes deployment — they automate the *creation* of
   repositories that already follow a working, understood pattern. Building that pattern
   by hand first means the Backstage template in Phase 9 is automating something we
   already understand deeply, not something we're learning for the first time through
   Backstage's abstraction.
3. **Reduces simultaneous unknowns.** Backstage has its own significant learning curve
   (plugin config, catalog YAML, scaffolder actions, auth providers). Learning that at
   the same time as learning Kubernetes/Helm/GitOps would make debugging failures
   ambiguous and slower.

## Alternatives Rejected
- **Backstage first**: more closely mirrors the original problem statement's diagram
  ordering ("developer opens platform" is step one), but this is a diagram of the
  *end-state user experience*, not a recommended *build order*. Building in end-state
  UX order is a common mistake — build order should follow dependency and risk, not
  narrative order.

## Consequences
- The "developer self-service" experience won't be demonstrable until Phase 9, even
  though the underlying capability (deploy a service via golden path) exists earlier
  in manual form. This is acceptable and will be called out explicitly in portfolio
  documentation: "manual golden path proven in Phases 2–7, automated via Backstage in
  Phase 9."
