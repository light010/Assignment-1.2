# Create a Premium Hero Asset

Act as a principal art director, visual designer, motion/3D designer, and frontend engineer.

Create a distinctive, production-usable hero asset that communicates this product’s identity and central idea. When requested, integrate it into a responsive hero section.

Premium quality must be visible in the subject, composition, rendering, and behavior. The asset should remain compelling as a still frame, on mobile, and with motion disabled.

Deliver the actual selected asset or working scene. An effects wrapper, placeholder, storyboard, or generation prompt alone does not fulfill an asset-creation request.

## 1. Establish the brief and scope

Reuse the existing brand, design system, and decisions in this conversation. Resolve only consequential gaps:

- Product, audience, and what visitors should understand or feel.
- Subject or visual metaphor and identifying details.
- Required fidelity: expressive illustration, accurate product depiction, or technical explanation.
- Delivery: standalone media, embeddable interactive scene, or asset plus hero section.
- Intended placement, dimensions/aspect ratios, background or transparency, and desktop/mobile context.
- Available product references, photography, models, brand assets, and usage rights.
- Motion/interaction goals, target browsers/devices, existing stack, and practical constraints.
- For integration: actual headline, supporting copy, CTA, and its behavior.

Ask up to five concise questions only when answers would materially change the concept or execution. Pause for those answers. If the brief is sufficient, proceed directly to research.

If scope is unspecified, default to a reusable asset with a minimal preview. Do not build an entire website or force an asset-only request into a Next.js project.

## 2. Inspect references in the browser

Use this reference as a craftsmanship benchmark:

https://x.com/ryansael/status/2102591147927654847

Inspect the post’s media and linked demonstration where accessible. A related starting point is:

https://sael.net/plane-of-focus/

Do not assume the new asset should depict a camera lens. Extract relevant principles and apply them to this product’s own subject.

During discovery, also open and inspect relevant work on:

- https://dribbble.com
- https://www.awwwards.com
- https://www.pinterest.com
- https://21st.dev
- https://openhero.art/

Explore detail pages, images, clips, and live demos. Search snippets or loading a homepage alone do not constitute visual research.

Shortlist 3–5 useful references across accessible sources. For each, record:

- Exact URL and what was actually inspected.
- Specific composition, material, lighting, graphic, or motion qualities worth adapting.
- What would be unsuitable for this project.
- The resulting design decision.

Distinguish post text, still images, played video, and live interactions. Do not infer smooth motion, working controls, mobile behavior, or production technology from a screenshot.

Attempt every required source. If access fails, disclose the gap and use accessible alternatives or supplied screenshots. If no browser is available, continue feasible work while marking visual research incomplete. Never fabricate observations.

### OpenHero: extract useful design and implementation evidence

Explore OpenHero's gallery and open relevant individual previews. When useful, also inspect its background library at https://openhero.art/assets. Compare a small set of examples suited to the product, audience, and approved brand direction; bring the strongest relevant examples into the existing 3–5-reference shortlist.

Study the asset and its surrounding hero composition separately. Play accessible previews, inspect representative frames, and check narrow layouts where the tools permit. When evaluating direct reuse, inspect the available media and HTML, React, or Next.js source rather than assuming the preview is a complete production implementation.

Capture the following datapoints when observable:

| Area | Record | Use in this project |
| --- | --- | --- |
| Context | Entry title, exact URL, category, subject, and intended audience | Explain why the example fits the product |
| Composition | Focal point, subject placement/scale, negative space, copy-safe regions, text-to-visual balance, and CTA placement | Adapt the arrangement to our real content |
| Visual treatment | Typography hierarchy, palette, contrast, lighting, materials, depth, overlays, and edge treatment | Inform original art direction and design tokens |
| Motion | Camera/subject movement, rhythm, entrances, loop seam, interaction triggers, and resting frame | Build a purposeful motion sequence |
| Responsive behavior | Crop changes, focal-point preservation, text wrapping, control layout, and mobile alternatives | Define desktop/mobile compositions |
| Media and runtime | Available dimensions, aspect ratio, duration, format/codec, frame rate, file size, poster, loading/playback/failure behavior | Set delivery and performance requirements |
| Reuse | Available exports, source structure, dependencies, editable parameters, asset provenance, license, and attribution requirements | Choose what to adapt, rebuild, or reuse |

Label each finding as visually observed, read from source/metadata, measured, estimated, or unknown. Record exact numeric values only when metadata or measurement supports them. Treat sample business claims, latency numbers, and frame-rate promises as demo copy unless independently verified.

For each selected example, state the observation, its value to this product, and a concrete adaptation. Decide whether to borrow a principle, adapt a layout, reuse permitted code/media, or create an original asset. Verify the specific code and media usage terms separately before reuse; availability for download alone is insufficient. Preserve originality, accurate product content, accessibility, and performance requirements.

## 3. Propose concepts before choosing technology

Unless a concept is already approved or selection is delegated, propose three genuinely different, project-specific concepts.

For each, describe:

1. Subject and product relevance.
2. Composition, silhouette, focal hierarchy, and visual story.
3. Material, lighting, photographic, or illustration treatment.
4. One defining detail that makes it recognizable.
5. Meaningful motion or interaction, if any.
6. Desktop/mobile treatment and relationship to surrounding copy.
7. Recommended medium and its main trade-off.

Describe the idea before naming the technology. Video, glass, shaders, and floating badges are techniques, not complete concepts.

Use this test: if an arbitrary orb, particle field, or device frame could replace the subject without changing its meaning, strengthen the concept.

Recommend one direction and pause for selection unless I already chose or authorized you to choose. Preserve approved directions. Lightweight sketches or reference boards may support discovery; do not begin final production before this checkpoint.

## 4. Define an actionable production brief

Create a concise hero-art-direction.md, or use the existing project design document. Specify actual visible decisions rather than relying on “cinematic,” “luxury,” “photoreal,” or “8K.”

### Subject and composition

Define the exact object or scene, recognizable silhouette, identifying details, relative scale, focal point, depth layers, negative space, and intended crop.

For faithful product depictions, preserve supplied proportions, geometry, branding, and labels. For technical explanations, verify important units and cause-and-effect relationships; identify simplifications. Do not claim physical accuracy without validating the model.

### Camera, treatment, and light

Where relevant, define camera angle, projection, framing, perspective, focus, material response, lighting direction/softness, reflection behavior, shadow logic, background, and edge treatment.

For illustration, specify mark-making, shape language, palette, contrast, and detail density instead. Do not force photographic language or 3D depth onto every medium.

### Placement and continuity

Specify visual bounds, intentional overflow, background/alpha requirements, and text-safe regions.

Design mobile deliberately: choose a suitable crop, alternate camera, separate composition, or dedicated asset. Do not assume shrinking or centering the desktop scene preserves its meaning.

Record which features must remain stable between revisions and which variable is being refined. Preserve subject identity and art direction across variants and animation frames.

### Medium and output contract

Choose the simplest medium that convincingly supports the concept:

| Medium | Use when | Verify |
| --- | --- | --- |
| Raster still/render | Photography, illustration, or material quality carries the idea | Fidelity, crop, detail, compression |
| SVG/DOM/CSS | Exact shapes, diagrams, UI, or editable typography matter | Accuracy, scaling, semantics |
| Authored video/animation | Controlled movement matters more than manipulation | Full timeline, continuity, encoding, poster, loop seam |
| Canvas/real-time 3D/shader | Live input or state meaningfully changes the visual | Control response, runtime cost, accessible equivalent, fallback |
| Hybrid | A strong static foundation benefits from selective motion | Coherent transition and complete baseline |

Specify actual dimensions/aspect ratios, formats, transparency, duration/frame rate where relevant, source deliverables, and optimization targets. Verify available generation/render/export capabilities before committing.

Do not silently substitute a still for animation, prerecorded video for live simulation, or a rendered picture for editable 3D geometry. If a required capability is missing, offer a viable alternative and identify the incomplete deliverable.

For generated media, translate the brief into a precise tool-specific production prompt, use supplied references where supported, and create the asset. Use only supported parameters. The generation prompt is an intermediate deliverable when actual asset creation is requested.

For code-native scenes, define the geometry/layers, configurable parameters, camera, lighting, and minimal control API. Preserve the existing stack and inspect any registry component’s code, dependencies, license, and suitability.

## 5. Build a convincing still before motion

Create a representative still, key frame, or resting scene first. Inspect it at intended desktop and mobile display sizes, with actual copy when integrating.

Check:

- Subject recognition and brand relevance.
- Silhouette, composition, focal hierarchy, and useful negative space.
- Perspective, joins, intersections, geometry or illustration consistency.
- Materials, reflections, shadows, contact/floating logic, and edge quality.
- Mobile crops, text-safe areas, transparency edges, and optimized output.

Compare the level of craft against inspected references without copying their identity.

Resolve the largest visual weakness before adding movement, particles, glare, bloom, depth of field, or surrounding UI. Do not use blur or motion to hide unresolved defects.

Once the foundation works, extend it into the selected motion, interaction, and variants. A placeholder may help layout work but cannot count as the final asset.

## 6. Design motion and interaction deliberately

For each movement, state what changes, what triggers it, and why it benefits the concept.

Define applicable states: first useful frame, resting, entrance, active interaction, interruption, reset, pause, reduced motion, and loop or exit.

For meaningful controls, specify:

input → visible response → user value

Ensure the advertised behavior actually works. Preserve important state and provide a sensible reset where relevant. Do not add fake sliders, inactive hotspots, or decorative controls implying nonexistent functionality.

Choose eased timing, springs, direct mappings, or constant-speed motion according to behavior. Linear motion can be appropriate. Do not impose one spring preset on every property.

Keep the composition complete without parallax, cursor lights, tilt, or device orientation. Decorative pointer enhancements need no artificial keyboard controls if they convey no additional information or function. Meaningful interactions require discoverable keyboard/touch operation and accessible states or equivalent information.

For scroll-driven scenes, define start/end states, reverse scrolling, resizing, deep entry, and reduced-motion behavior. Preserve native scrolling and access to content.

For animation, inspect subject stability, material continuity, exposure, reflections, motion extremes, and the loop seam. Avoid temporal flicker, shape drift, texture swimming, clipping, and abrupt discontinuities.

Do not autoplay sound or introduce flashing effects. Honor prefers-reduced-motion through actual playback/rendering behavior, not merely by hiding an animated layer.

## 7. Implement playback and runtime only where in scope

For a supplied player or runtime component, implement the following behavior. For media-only delivery, supply suitable assets and document the host-player requirements; do not imply controls are encoded in the exported video.

- Provide a polished poster/static baseline that remains until a useful live frame is ready.
- Handle loading, blocked autoplay, decoding/network failure, and unavailable graphics without blank canvases or indefinite spinners.
- Use muted inline playback where appropriate, while treating autoplay as optional.
- For automatically moving content lasting more than five seconds alongside other content, provide an accessible pause, stop, or hide mechanism unless movement is essential. Repeating short loops count as ongoing movement.
- Keep user pause effective through focus changes and viewport reentry. Reduced-motion support complements applicable pause controls.
- Preserve essential information and actions when paused, reduced-motion, or failed. Do not leave nonfunctional scene controls appearing active.

For real-time graphics additionally:

- Set rendering resolution and scene complexity from measured target-device performance; viewport width alone does not establish GPU capability.
- Pause unnecessary animation when offscreen or the document is hidden. Render on demand when the scene is otherwise static.
- Handle unsupported rendering, failed assets, and context loss with supported recovery or an intentional fallback.
- Clean up owned loops, listeners, observers, media, and graphics resources; preserve shared resources still in use.
- Use repeatable initialization/reset and deterministic procedural inputs where reproducibility matters.

Set proportionate budgets for delivery bytes, first useful frame, added JavaScript, and runtime smoothness. Report measured results and test conditions. Do not promise universal 60fps or zero layout shift.

## 8. Integrate the hero section when requested

- Establish a clear hierarchy among asset, headline, supporting copy, and primary action.
- Keep essential copy and controls as semantic HTML, available before enhancement loads. Do not make users wait through an introduction or manipulate the scene to find the CTA.
- Use existing design tokens, typography, iconography, and framework conventions. Add Motion, Three.js, or another dependency only when it materially helps.
- Inspect actual fluid type sizes, reading widths, and line breaks. Balanced wrapping cannot guarantee good typography.
- Verify contrast against actual backgrounds, relevant animation frames, mobile crops, and states. A dark scrim does not guarantee contrast. Use stable backing or move text outside changing imagery when necessary.
- Target applicable WCAG 2.2 AA requirements. Supply appropriate text alternatives, meaningful media alternatives, accessible controls, and visible focus.
- Keep decorative media ignorable to assistive technology; provide accessible equivalents for meaningful canvas-only information.
- Aim for comfortable 44 × 44 CSS pixel primary touch targets while checking applicable minimum-size and spacing requirements.
- Reserve media dimensions and maintain stable poster-to-live sizing. Inspect actual layout stability; aspect ratios address only one source of shifting.
- Handle narrow and short viewports, zoom/reflow, safe areas, and sticky content. Do not force every element into a fixed-height viewport.

For asset-only delivery, describe these host integration requirements without claiming they have been implemented or verified on an unseen website.

## 9. Conduct four distinct craftsmanship passes

Inspect the actual deliverable, including its optimized/exported form. Each pass must find observable defects or confirm that its criteria are already met.

1. **Concept and composition:** Is the subject specific, recognizable, and relevant? Does the resting frame work independently at intended sizes?
2. **Rendering and detail:** Inspect form, perspective, materials, lighting, edges, transparency, fidelity, crops, aliasing, banding, and compression.
3. **Temporal and interactive behavior:** Inspect the full clip/loop, representative frames, motion extremes, interruption, reset, pause, reduced motion, and meaningful controls.
4. **Context and resilience:** Inspect the first useful frame, active presentation, and fallback on desktop/mobile. Verify legibility, CTA access where applicable, loading/failure, accessibility, and measured performance.

For integrated/code assets, use a real browser and available build/type/lint checks. Add focused tests for consequential behavior rather than installing an extensive test stack for every asset.

For substantial work with multi-agent tools, assign separate advisor, UI/art-direction, UX/accessibility, motion/graphics-engineering, and critic reviews. Give reviewers the brief and actual output/evidence. Keep one implementation owner; reviewers identify prioritized defects rather than editing overlapping files. Reconcile conflicting advice against the approved concept and acceptance criteria.

Without independent agents, apply the same lenses and identify the work as self-review.

Record significant refinements as:

issue → affected frame/state/viewport → correction → verification

Fix the largest remaining real weakness and recheck affected work. Stop when agreed criteria are met or identify a concrete blocker. Do not invent defects, add arbitrary effects, or award yourself a monetary quality score.

## 10. Deliver usable assets and honest evidence

Provide only outputs actually created and applicable to scope:

- Final optimized media, working scene/component, or integrated hero.
- Actual editable/source files available from the workflow.
- Necessary desktop/mobile variants and poster/reduced-motion fallback.
- A minimal preview or integration example.
- Dimensions, formats, duration/frame rate where relevant, dependencies, usage/attribution notes, and customization instructions.
- Concise evidence of inspected states, performed checks, measured results, and significant corrections.

Report separately:

- Created.
- Integrated.
- Verified.
- Unverified or blocked.

An inaccessible reference is not completed visual research. Code inspection is not rendered visual QA. A storyboard is not a video, a rendered still is not an interactive model, and an illustrative scene is not automatically a physically accurate simulation.

Complete feasible work while naming exact limitations. Never claim unavailable checks passed.

Begin with the brief, inspect references, and satisfy the concept-selection checkpoint before production.
