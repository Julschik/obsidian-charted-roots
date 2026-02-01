# Entity Profile Views

Planning document for entity Profile Views feature.

**Status:** 📋 Planning

**Related:** [#239](https://github.com/banisterious/obsidian-charted-roots/discussions/239) (Control Center modularization), [#240](https://github.com/banisterious/obsidian-charted-roots/discussions/240) (Dockable sidebar views)

---

## Overview

Profile Views are entity-specific `ItemView` subclasses that provide comprehensive, scrollable views for deep work on a single entity (person, place, event, source, organization). Each entity type has its own registered view with a distinct display name and icon. Identity fields appear at the top, followed by collapsible sections for related data — all inline and editable.

**Motivation:**
- Current workflows are entity-type-centric (People tab, Events tab, Sources tab), but real user workflows cut across entity types — adding a birth event to a person requires jumping between People, Events, and Sources, losing context along the way
- The v0.20.0 dockable Browser views solve the window management problem but preserve the context-switching problem
- Profile Views keep all related data visible together in collapsible sections, enabling deep work on a single entity without tab-hopping
- Separate view types per entity enable natural multitasking — a person profile docked alongside an event profile, each clearly labeled

---

## Architecture

### View types

Five registered `ItemView` subclasses, one per entity type:

| View type | Display name | Icon | Sections |
|-----------|-------------|------|----------|
| `PersonProfileView` | Person profile | `user` | Identity → Family → Events → Sources → Media → Data Quality |
| `PlaceProfileView` | Place profile | `map-pin` | Identity → Events at location → Sources → Media → Map preview |
| `EventProfileView` | Event profile | `calendar` | Identity → Participants → Sources → Media → Place link |
| `SourceProfileView` | Source profile | `book-open` | Identity → Referenced facts → Persons cited → Media |
| `OrganizationProfileView` | Organization profile | `building` | Identity → Members → Events → Sources → Media |

### Shared infrastructure

All five views extend shared base utilities that provide:
- Collapsible section rendering with chevron toggle
- Breadcrumb bar rendering and navigation history
- State persistence (entity cr_id, expanded/collapsed sections)
- Debounced refresh on vault changes (2s)
- Scroll-to-section support

### Section renderers

Section renderers are standalone functions (e.g., `renderProfileFamilySection()`, `renderProfileEventsSection()`) following the existing tab renderer pattern from Phase 1. Sections shared across entity types (Sources, Media) use the same render function.

---

## Navigation

### Entry points

- **Context menu** on entity notes: "Open person profile", "Open place profile", etc.
- **Browser views**: Click-through from People/Places/Events/Sources/Organizations Browser views
- **Command palette**: "Open person profile", "Open place profile", etc.

### Cross-entity navigation

Clicking a related entity of a **different type** (e.g., a source in a Person profile) opens that entity's Profile View. Because each entity type has its own view type, this naturally opens in a separate pane rather than replacing the current view. This enables the side-by-side workflows that motivated this feature.

### Same-type navigation

Clicking a related entity of the **same type** (e.g., a child in a Person profile) navigates the current view in-place with breadcrumb history. Modifier-click (Ctrl/Cmd) opens a new instance instead.

### Breadcrumb bar

Shows navigation path within a single view instance (e.g., "John Smith → Mary Smith → Elizabeth Smith"). Clicking a breadcrumb navigates back.

### Relationship to existing views

Profile Views are an **additional option**, not a replacement for existing behavior. Users who prefer working in raw markdown continue to do so. Browser views preserve their current "open note" behavior, with Profile Views available as an alternative entry point.

---

## Save behavior

Follows the Obsidian Properties UI pattern:
- **Simple fields** (name, dates, occupation): save on blur
- **Relationship additions** (spouse, parent, source): save on picker confirmation
- **Relationship removals**: save immediately with confirmation prompt for destructive changes
- No explicit Save button — changes write immediately to frontmatter
- This matches Obsidian's convention of immediate persistence and avoids crash-risk from deferred saves

---

## Reusable components

The existing codebase provides strong building blocks:

| Component | Location | Use in Profile Views |
|-----------|----------|---------------------|
| `FamilyGraphService` | `src/core/family-graph.ts` | Load person, resolve family relationships |
| `PlaceGraphService` | `src/core/place-graph.ts` | Load/resolve place entities |
| `EventService` | `src/events/services/event-service.ts` | Load events for a person/place |
| `SourceService` | `src/sources/services/source-service.ts` | Load/manage sources |
| `EvidenceService` | `src/sources/services/evidence-service.ts` | Research coverage %, fact sourcing |
| `ProofSummaryService` | `src/sources/services/proof-summary-service.ts` | Proof notes, conflict tracking |
| `MediaService` | `src/core/media-service.ts` | Resolve media linked to entity |
| `renderPersonTimeline()` | `src/events/ui/person-timeline.ts` | Chronological event display |
| `PersonPickerModal` | `src/ui/person-picker.ts` | Browse & select person |
| `PlacePickerModal` | `src/ui/place-picker.ts` | Browse & select place |
| `SourcePickerModal` | `src/sources/ui/source-picker-modal.ts` | Browse & select source |
| `EventPickerModal` | `src/events/ui/event-picker-modal.ts` | Browse & select event |
| Collapsible section pattern | `src/sources/ui/create-source-modal.ts` | Chevron toggle, expand/collapse |
| `Setting` class | Obsidian API | Form field rendering |

---

## CSS naming

- Shared: `cr-profile`, `cr-profile__section`, `cr-profile__section-header`, `cr-profile__breadcrumb`
- Entity-specific: `cr-person-profile`, `cr-place-profile`, `cr-event-profile`, `cr-source-profile`, `cr-org-profile`
- Section content: `cr-profile__identity`, `cr-profile__family`, `cr-profile__events`, `cr-profile__sources`, `cr-profile__media`, `cr-profile__data-quality`

---

## Implementation phases

### Phase 1 — Read-only Profile Views

- Implement shared profile view infrastructure (collapsible sections, breadcrumb, state persistence, debounced refresh)
- Register all five view types
- Implement section renderers for each entity type (read-only display)
- Reuse existing components: `renderPersonTimeline()` for events, `MediaService` for thumbnails, `EvidenceService` for data quality
- Click-through navigation between entity profiles
- Context menu and command palette entry points
- State persistence (which entity, which sections expanded)

### Phase 2 — Inline editing

- Edit identity fields with save-on-blur
- Add/remove relationships via existing picker modals
- Create events and sources inline from within the profile
- Immediate frontmatter persistence
- Undo support for accidental edits

### Phase 3 — Polish and integration

- Mini table-of-contents / section jump links in header for long profiles
- Browser view "Open profile" integration (click row → open profile)
- Keyboard navigation between sections
- Mobile-responsive layout (collapse sections by default, touch-friendly targets)
- Performance optimization (lazy-render sections on expand)

---

## Design decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| View registration | Separate view type per entity | Clear tab labels, natural multitasking, consistent with existing plugin pattern |
| Shared code | Base class or shared utilities | Avoid duplication for sections, breadcrumb, state persistence |
| Section layout | Vertical scroll with collapsible sections | Matches Obsidian Properties pattern, works on all screen sizes |
| Section renderers | Standalone functions | Reusable, testable, consistent with existing tab renderer pattern |
| Save model | Save on blur / on picker confirm | Matches Obsidian conventions, avoids crash-risk from deferred saves |
| Browser relationship | Profile is additional option, not replacement | Preserves existing workflow for users who prefer raw markdown |
| Cross-entity links | Open in separate pane (different view type) | Enables side-by-side workflows that motivated this feature |
| Same-entity links | Navigate in-place with breadcrumb | Avoids proliferating tabs of the same type |
