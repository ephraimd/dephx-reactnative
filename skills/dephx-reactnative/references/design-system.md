# Native composition and design system

## One foundation, project-owned identity

Load the app's canonical brand, copy, tokens and component rules. Examples are not
authority. Reuse its approved UI library through one shared component directory.
For a new app, agree the foundation before building screens. Gluestack can supply
controls; NativeWind supplies styling, not the interaction semantics of a UI kit.
Add only needed components and compose missing pieces from existing primitives.
Do not mix kits or hand-build a second Button/Input/Dialog family.

Use semantic color roles in both themes, shared spacing/radius/type variants and one
source for token values. If native APIs need resolved values, derive or validate them
against that source rather than maintain an unchecked second palette. Subscribe to
system appearance changes when system theme is supported. Persist through existing
storage; a theme preference alone does not justify a native storage dependency.

## Proportions and hierarchy

- Related actions use matching size variants; labels stay centered and wrap deliberately.
- Compact status badges are not oversized buttons. Interactive hit areas remain large
  enough for platform accessibility requirements; small radio visuals sit in labelled targets.
- Borders, selected, disabled, invalid and pending states come from shared primitives.
  State is never conveyed by color alone; expose role, label, value and accessibility state.
- Use consistent screen roles and gutters, adapting to phone/tablet dimensions instead of
  copying web width caps. Put the user's task before decoration. Motion must aid feedback
  and respect reduced motion; large textures, blur and animated backgrounds need evidence.
- Prefer existing native stack headers and back behavior. Use a shared custom header only
  when the product needs it; do not hide native headers throughout an app by default.

## Stable navigation and bounded records

Never put an independent task after a growing list. Controls, section navigation, funding
and configuration remain reachable regardless of record count. List pagination/recovery
belongs with records; a chat composer stays outside an independently scrolling transcript.
Use labelled task tabs/menu patterns, with a focused dialog for a short secondary task
and a screen for substantial authoring. Do not squeeze an entire editor into a modal.
Contextual shortcuts supplement a findable home. Preserve scope/filters when paging or
returning from detail; explain which records a total covers.

Use the existing virtualized list and backend pagination, not a ScrollView rendering
every row. Test zero, one, many records, long names, unbroken links and large text.
Keep screen-level search compatible with the backend; scan detection is a different
interaction and must not inherit a generic search debounce.

## Keyboard, safe areas and overlays

Use one shared overlay owner for dismissal, scroll, accessibility focus and opener
restoration. Do not stack duplicate modal/focus/keyboard handlers around a library
primitive. Menus align for reading and avoid viewport edges, including the final row.
Native overlay behavior must be tested rather than inferred from web portals.

Dialogs/sheets fit available space including safe areas and the visible keyboard.
The first and last fields, close control and footer actions remain reachable on a short
phone. Ensure focused fields scroll into view. Reserve bottom navigation/inset space;
test Android back, VoiceOver/TalkBack focus and returning to the opener. Tablets do not
automatically justify multiple columns inside a narrow sheet.

## Understandable tasks and safe drafts

Name actions for the user's work. Put custom-form editing where users create the thing
being registered or purchased; state built-in fields and question scope. Explain settings
through actual consequences. Use explicit Save where users need certainty; show unsaved,
saving, saved and failed states without distracting them. Do not invent a preview flow
when the product calls for saving and opening the actual published page.

Hydrate drafts by resource identity. A refetch must not replace dirty fields. Capture
the submitted snapshot at save; acknowledge that snapshot without overwriting later
edits. Draft A must not become resource B's draft. Preserve input on failed saves and
follow the existing discard/back-navigation policy.

Distinguish cold errors from blank records, stale retained data from fresh results and
unknown availability from Off. Labels must reflect actual operations: saving a draft
does not send it; storing a contact does not grant marketing consent. Display actual
server fees, including zero, separately from provider charges. Seller-owned surfaces
retain the product's actual ownership and disclosure rules.

## Evidence

Use `testing.md` for real runtime and release-performance checks. A shared component
fix needs representative consuming-screen checks, not one isolated attractive mockup.
