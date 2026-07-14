# Presentation-Grade draw.io — Quality Conventions & Verification Loop

Distilled from iterative architect-review cycles on real enterprise presale diagrams. These rules exist because every one of them was violated once and caught by a human reviewer or a render check — the XML always looked fine.

## The Verification Loop (non-negotiable for client-facing diagrams)

**"Export succeeded" is not a gate. Pixels are the gate.**

1. Export: `draw.io --export --embed-diagram --format png --scale 2 --border 12 --output name.drawio.png name.drawio`
2. **Read the PNG** and inspect the whole diagram.
3. **Crop suspect regions** (PIL: `Image.open(png).crop((x1,y1,x2,y2))`) — arrowheads, edge labels, corridor areas — and Read the crops. Scale factor: `px = page_units × scale`, plus border.
4. Fix → re-export → re-crop. Repeat until clean.

Defects this loop catches that XML review never does: edges passing through boxes, labels overprinting arrows/borders/each other, arrowheads landing short of targets, broken icon placeholders.

## Edge Labels

- **Label-gap rule:** a label needs clear whitespace at least as wide as its rendered text (≈ `0.55 × fontSize × char_count` page units). **An offset cannot fix a gap narrower than the label — widen the layout instead.** Nudging just moves the collision.
- Always set `labelBackgroundColor=#ffffff` on labeled edges.
- **Placement controls:** edge geometry `x` (relative, −1…1 along the path) picks *which segment* the label sits on; `<mxPoint as="offset"/>` nudges from there. When two labels collide, separate by segment choice first, offset second.
- Plan label real estate up front: leave 30–60px gutters between containers *on the routes you know will carry labeled edges*; park long labels in open bands above/below containers.

## Edge Routing

- **Corridor routing:** run orthogonal edges through the deliberate gutters between containers. After adding waypoints, trace every segment: crossing a container *border* is fine; crossing a *card or text* is a defect.
- **Swimlane header bars are full-width text zones.** Any vertical exiting through a container's top crosses the header bar — choose a crossing x outside the (centered) title text span, or route out the side through an inter-container band.
- Leave ≥20px of straight final segment for the arrowhead; a bend too close to the target renders a broken-looking arrow.
- Stacked siblings: if an edge from box A must reach something directly below box B, consider **swapping A and B** so the edge drops through empty space instead of through B.

## Container & Card Styling (the ratified look)

- **Sub-containers get colored headers:** `swimlane;startSize=28;fillColor=#<header>;swimlaneFillColor=#ffffff;strokeColor=#<accent>;fontColor=#<dark-accent>;fontStyle=1;pointerEvents=0;`
- **Dominant/region container:** add `dashed=1;dashPattern=6 4;strokeWidth=2` and a tinted `swimlaneFillColor` (e.g. `#f5f9ff`).
- **Icon-on-left cards:** card style `rounded=1;align=left;spacingLeft=44;fontSize=11` with `<b>Name</b><br>short description`; icon is a *separate* `image` cell (~30×30) parented to the same container, positioned over the card's left padding.
- Consistent palette per lane family (e.g. channels green `#d5e8d4/#82b366`, platform blue `#dae8fc/#6c8ebf`, offline/jobs orange `#ffe6cc/#d79b00`, core purple `#d9c9f2/#5e35b1`, neutral gray).
- **No meta/provenance in client-facing diagrams:** no "per <person>", "draft", "generated", tool names, or internal cross-references. State facts neutrally.

## Icons

- Azure icons: `image;html=1;image=img/lib/azure2/<category>/<Name>.svg;`
- **Broken paths fail silently** — they render as a generic broken-image placeholder, not an error. **Probe first:** put candidate icons in a scratch .drawio, export, and look. Keep and reuse a verified list. Known-good azure2 paths:
  - `app_services/API_Management_Services.svg`, `app_services/Search_Services.svg`
  - `compute/Kubernetes_Services.svg`, `containers/Kubernetes_Services.svg`
  - `ai_machine_learning/AI_Studio.svg`, `ai_machine_learning/Cognitive_Services.svg`, `ai_machine_learning/Machine_Learning.svg`, `ai_machine_learning/Speech_Services.svg`
  - `databases/Azure_Cosmos_DB.svg`, `storage/Data_Lake_Storage_Gen1.svg`
  - `identity/Azure_Active_Directory.svg`, `security/Key_Vaults.svg`
  - `networking/ExpressRoute_Circuits.svg`, `networking/Firewalls.svg`, `networking/Web_Application_Firewall_Policies_WAF.svg`, `networking/Virtual_Networks.svg`
  - `management_governance/Monitor.svg`, `management_governance/Application_Insights.svg`
  - `analytics/Azure_Databricks.svg`, `analytics/Power_BI_Embedded.svg`
  - Known broken: `management_governance/Azure_Purview_Accounts.svg`, `web/Notification_Hubs.svg`

## Working With Review Feedback

- Reviewers flag *classes* of defect, not instances. When one label overlaps, audit **every** label the same way before resending — the verification loop, applied diagram-wide, is cheaper than another review round.
- When embedding exported PNGs into decks (python-pptx), compute placement from real image dimensions (aspect ratio), and re-render the deck page to verify fit — diagrams wider than ~2:1 overflow standard 16:9 content areas easily.

## Icon Density — default to using icons

**Every named service/component gets an icon if a verified one exists.** A text-only card reads as a placeholder; an icon+label card reads as finished. Don't reserve icons for "important" boxes — apply them uniformly across a diagram, including secondary/on-prem boxes when a reasonable icon exists (generic server/database/desktop shapes are fine for legacy or unspecified systems). Only skip an icon when nothing sensible applies (e.g. an abstract "EDW" placeholder with no canonical logo).

Icons should be **visually dominant, not a decorative badge** — size them to roughly 40-60% of the card's shorter dimension, not a small corner mark. If a reviewer says "make the logo bigger," they mean a step change (1.4-1.8×), not a nudge.

## Transparent Backgrounds

**Export client-facing / deck-bound PNGs with `-t` (transparent).** A white background is invisible in draw.io's own canvas but becomes a hard visible box the moment the image lands on a slide with any template background, texture, or corner branding/watermarks — exactly the kind of defect the pixel-verification loop should catch (check it when reading the exported PNG: does it carry a white rectangle that will fight the slide background?). Default to transparent for anything destined for a deck or doc; only export opaque when the image is viewed standalone.

## Density & Legibility — the real lever for "hard to read" feedback

When a diagram will be scaled down to fit a fixed physical size (a slide), the rendered size of every icon and letter is governed by `physical_size / page_units` — **not** by the fontSize or icon size written in the XML alone. This means the single highest-leverage fix for "text is too small on the slide" is usually **shrinking the page's total footprint** (cutting dead space), not just bumping fontSize:

- Figure out which dimension of the diagram is *binding* against the target physical box: compare the diagram's aspect ratio to `width_max / height_max`. If the diagram is wider (relatively) than the target box, it's width-bound — shrinking page **width** increases the effective on-slide scale; shrinking page height does nothing for legibility there (it only changes how much of the vertical ceiling gets used). If the diagram is taller-relative, it's height-bound and the lever flips.
- Compacting a diagram (tighter cards, shorter labels, smaller gaps) on the *binding* axis is worth more than any single fontSize increase, because it raises the effective size of *everything* — icons, text, line weights — simultaneously.

**Practical order of operations when asked to "make it more readable" or "condense":**
1. Bump icon size (see Icon Density above).
2. Condense every card's text to keyword phrases — drop connective words and secondary clauses; a card label is not a sentence.
3. Shrink card padding so the border hugs the icon+text (no more than ~15-20% dead margin).
4. Reduce inter-card / inter-row gaps modestly.
5. Recompute the page's total width/height from the new tighter layout — this compaction, not the individual tweaks, is what makes the exported image read bigger once placed on a slide at a fixed physical size.

**Tightening a gap can turn a previously-safe label into a border collision.** The label-gap rule (min clear width ≈ `0.55 × fontSize × char_count`) was satisfied at the *old* gap size — after compaction, re-check every labeled edge that crosses a gap you shrank, not just the ones you intentionally touched. This is a common self-inflicted defect from density passes: shrinking a 30px gap to 22px can leave a label with no room even though nothing about the label itself changed.

**Risk-managing dense diagrams with many hardcoded edge waypoints:** a full relayout (moving every container/card position) requires recomputing every absolute waypoint and re-verifying every corridor — expensive and error-prone on diagrams with many `<Array as="points">` edges. When the diagram is edge-waypoint-heavy, prefer a **low-risk pass**: enlarge icons and condense text *in place* (same card x/y/width/height, just bigger icon + tighter spacingLeft), which touches nothing edges depend on. Only reflow container/card positions when the ask specifically requires shrinking the overall footprint (e.g. to fit a hard physical-size ceiling) — and budget time to re-verify every affected waypoint when you do.

## Client Review Lessons (recurring feedback patterns)

Feedback from an actual client architecture-diagram review round that generalizes to most stakeholder reviews of dense technical diagrams:

- *"Make each service's logo bigger"* → icons were decorative-sized relative to the card; fix is Icon Density above, applied aggressively, not incrementally.
- *"Shorten the text to common/frequent keywords"* → cards had full clauses instead of keyword phrases; fix is text condensation (step 2 above).
- *"The border should hug close to the logo and text"* → cards had generous dead padding around a small icon+short label; fix is tightening card height/padding to content (step 3 above).
- *"Reduce spacing between components"* → perceived over-spacing was often actually a symptom of small icons/dense text inside oversized cards, not literally-too-large gaps between cards — enlarging icons and condensing text can resolve this complaint even without touching inter-card gaps. Check both: measure actual gaps against the label-gap rule before assuming they need to shrink.
- *"Cutting content down makes the deck easier to read immediately"* — the standing bias when a reviewer flags density: cut before you shrink fontSize, and cut before you add more explanatory sub-clauses. Every diagram destined for a slide should default toward fewer words and bigger icons over more precise/complete prose.

## Pairing With the drawio MCP

The MCP tools (`open_drawio_xml`, `search_shapes`) open diagrams in the app for interactive viewing/shape lookup. They do not replace the loop: the CLI export + PNG-read is the only channel that shows you what the client will see. Author XML → (optionally) open via MCP for a live look → **always** gate on the exported PNG.
