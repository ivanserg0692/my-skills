---
name: ecommerce-frontend-design
description: Use the project's documented design and API research when planning, implementing, reviewing, or improving the e-commerce storefront in app/frontend, including shared storefront UI. Does not apply to unrelated backend work or standalone admin UI.
---

# E-commerce Frontend Design

Use the existing storefront design documentation as the basis for frontend decisions. Keep design rules and API findings in their original documents rather than copying them into this skill.

## Read the Relevant Design Basis

Start with the [design entrypoint](../../../app/frontend/docs/design/README.md) to establish scope, status, and evidence date. Then read only the material relevant to the task:

- [Page map](../../../app/frontend/docs/design/page-map.md): routes, page purpose, UI components, endpoint dependencies, and full/partial/missing API support. Read the affected page entries before planning or changing a user flow.
- [Design concept](../../../app/frontend/docs/design/design-concept.md): tokens, themes, responsive layouts, component choices, state ownership, and implementation sequence. Read the relevant sections before changing storefront presentation or shared UI.
- [API analysis](../../../app/frontend/docs/design/api-analysis.md): response shapes, authorization, catalog behavior, cart/order semantics, existing frontend integration, and known gaps. Read the relevant contracts before integrating data or changing interactions.
- [References](../../../app/frontend/docs/design/references.md): visual sources, provenance, and access limitations. Consult when choosing or extending a visual solution.

For visual changes, inspect the saved [prototype board](../../../app/frontend/docs/design/prototypes/storefront-light-dark.png) with an image viewer. Columns show home, catalog, and product; rows show Light and Dark. Apply the documented corrections and API limitations alongside the image. It is a raster concept with demonstration content, not a pixel-exact specification or an implemented interface.

Resolve links relative to this skill directory. For commands run from the application repository root, the documentation is under `frontend/docs/design/`; the frontend has its own Git repository. If required documents or images are missing, report the missing material instead of inventing its contents.

## Apply the Basis to the Requested Change

- Follow the documented violet accent, Inter typography, rounded cards, restrained borders/shadows, container width, and responsive grid. Account for Light, Dark, and System according to the documented theme approach and actual implementation state.
- All storefront layouts must be responsive across desktop, tablet, and mobile widths. Reflow grids, navigation, filters, and page sections to fit the available space rather than merely shrinking the desktop layout. Keep text readable and controls usable by touch, and prevent unintended horizontal page overflow.
- Preserve existing working Next.js/React/Refine components, routing, localization, providers, and integrations where practical. Use the documented distinction between composed storefront pages and Refine resource workflows when choosing components.
- Check installed dependencies before selecting UI libraries. A recommendation for Tailwind, shadcn/ui, Lucide, or next-themes does not mean it is installed or authorize installation.
- Match visible controls to verified data and capabilities. Do not infer popularity, reviews, variants, product galleries, payment, or delivery support from prototype illustrations. Use the affected page's support level and documented limitations to shape the UI.
- Treat API research as a dated observation. Verify affected endpoints, DTOs, controllers, Gateway routes, and access rules against current source before relying on them. Use safe local requests when useful; do not mutate business data merely to validate a design.
- When current source differs from the report, explain the discrepancy and its effect on the requested change. Do not silently preserve a stale contract or implement a speculative backend feature.
- Explicit user instructions and subsequently approved changes take precedence over this design basis. Explain material departures from the documented direction; do not expand a small frontend task into a redesign.

## Keep storefront query logic cohesive

When a storefront component mixes URL handling, query coordination, and JSX, group the related state and behavior by responsibility. In the catalog, a local URL hook reads and writes filter and pagination parameters, and a local data hook coordinates Refine requests and exposes the data, loading states, and retry actions used by the view. The component owns rendering and UI interactions such as opening and closing the filter drawer. Keep hooks in the component file while their scope is local.

Pass resource names as props to storefront list components that are used across routes or with different resources. Let Refine resolve the provider from the registered resource's `meta.dataProviderName` unless a caller needs an explicit override. Keep resource-specific card and detail components focused on their existing use cases.

## Keep Scope and Validation Explicit

The saved documentation and prototypes do not authorize implementation, backend changes, or new dependencies. Apply the project's change-approval workflow to the concrete task, using existing approval when it already covers the proposed work.

For an implementation or review, identify the affected pages, API contracts, reused components, and remaining blockers. For layout changes, verify the changed UI at desktop, tablet, and mobile widths in both themes when preview tools are available; report any unverified sizes or states. Include relevant loading, empty, error, and authentication states. Follow project verification restrictions: use targeted checks and do not run frontend builds.

If an approved change alters a documented route, contract, or design decision, identify the corresponding design documentation update and handle it within the authorized scope. Keep observed behavior, proposed work, and unsupported capabilities distinguishable.
