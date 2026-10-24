# Tie Your Tie — Spec

A single-page site that shows, as a looping animation, how to tie three common knots. No filler text: the page has a headline, a knot selector, the animation, and two controls. The animation shows only the tie: no neck, collar or shirt.

---

## 1. Scope

**In**
- One page, one route
- Three knots: **Four-in-Hand**, **Half Windsor**, **Windsor**
- An animated line drawing for each knot that plays automatically and loops
- Controls: a **speed** setting and a **scrub slider**

**Out**
- Intro copy, step captions, a step counter, history, "about", footer links, cookie banners, analytics, sign-up, ads
- Video or GIF assets (the animation is drawn in code)
- Frameworks, build steps, external runtime dependencies

---

## 2. Page structure

Top to bottom, centred in one column:

```
            Tie Your Tie                 ← h1

  [ Four-in-Hand | Half Windsor | Windsor ]   ← selector

        ┌───────────────────────┐
        │                       │
        │      (animation)      │       ← square stage
        │                       │
        └───────────────────────┘
   ●━━━━━━━━━━━━○───────────────────     ← scrub slider
            0.5×   1×   2×               ← speed
```

- The whole layout fits one desktop viewport with no scrolling (target ≥ 700px viewport height).
- Column max width: 560px. Stage: square, `min(100%, 480px)`.
- On mobile (≥ 320px wide), the same order with 16px side gutters and no horizontal scroll.

---

## 3. Visual design

| Token | Light | Dark |
|---|---|---|
| `--bg` | `#F6F4EF` (warm paper) | `#141413` |
| `--ink` | `#1A1A1A` | `#ECEAE4` |
| `--muted` | `#8A867E` | `#7C786F` |
| `--tie` | `#6E1A2B` (Bordeaux) | `#9A2A3F` |
| `--tie-stripe` | `--tie` lightened ~14% | `--tie` lightened ~12% |
| `--tie-back` | `#3A3A3C` (plain dark gray) | `#4A4A4D` |
| `--line` | `#D9D5CC` | `#2C2B28` |

- Dark mode follows `prefers-color-scheme`.
- **Headline:** serif display face (e.g. *Instrument Serif* or *Cormorant Garamond*, Google Fonts), ~56px desktop / 40px mobile, normal weight, slight negative tracking.
- **UI text:** one sans face (e.g. *Inter*), 13–14px, `--muted` until active.
- **Illustration style:** the tie alone, with 1.3px `--ink` outlines. No neck, collar or shirt, and no gradients or drop shadows. The stage fades out at the bottom, where the hanging blades run out of frame.
- **Favicon:** a small Bordeaux tie, as an inline SVG data URI.
- **Tie fabric:** the front face is Bordeaux `--tie` with a **subtle diagonal stripe** in `--tie-stripe`. Stripes are thin (≈ 15% of the blade width), evenly spaced, at roughly 45° to the blade, in the classic repp direction (running from the wearer's upper left to lower right). The back face is plain dark gray `--tie-back` with no stripes, so you can tell at a glance which side of the fabric is showing.
- **Motion:** eased (ease-in-out) between steps; no bounce or overshoot.
- Plenty of whitespace; no borders around sections. The stage has no frame.

---

## 4. Knots and steps

All knots are drawn **mirror view** (as the wearer sees themselves in a mirror), starting with the wide blade on the wearer's right. Each step is one keyframe; the animation tweens between them.

The step descriptions below are **authoring notes for the keyframes only** and are not shown on the page.

### Four-in-Hand — 6 steps
1. Wide end long, on the right
2. Cross wide over narrow
3. Wrap behind
4. Cross over the front again
5. Up through the neck loop
6. Down through the front, tighten

### Half Windsor — 8 steps
1. Wide end long, on the right
2. Cross wide over narrow
3. Wrap behind
4. Up through the neck loop
5. Down to the same side
6. Across the front
7. Up through the neck loop
8. Down through the front, tighten

### Windsor — 9 steps
1. Wide end long, on the right
2. Cross wide over narrow
3. Up through the neck loop
4. Down to the left
5. Behind, to the right
6. Up through the neck loop, down right
7. Across the front
8. Up through the neck loop
9. Down through the front, tighten

> Step sequences should be checked against a reference diagram before the keyframes are drawn; the order above is the standard one, but the drawing is the authority.

---

## 5. Animation model

### 5.1 Representation
The tie is **one continuous ribbon**, not two separate shapes. That keeps the fabric continuous through every wrap.

- **Spine:** an ordered list of N points (N ≈ 48) from the narrow tip to the wide tip.
- **Width profile:** fixed per spine index; it tapers from the narrow tip (~1.6 units) to the wide tip (~3.6 units), with pointed ends.
- **Depth:** each spine point has a `z` value (front/behind). The renderer uses it to decide what draws over what, so wraps and pass-throughs read correctly.
- **Face:** each spine point has a `face` flag (front/back of the fabric). It switches between the striped front and the plain back lining where the blade twists.
- **Neck loop:** the band that goes around the (invisible) neck is part of the ribbon. Its back arc shows the gray back face.

### 5.2 Keyframes
- Each keyframe is hand-authored as a short list of waypoints `[x, y, z, flags, widthScale]` along the tie. Flags mark the back face, fold edges, and the two points pinned at the two sides of the neck band.
- On load, each keyframe is smoothed (centripetal Catmull-Rom) and resampled into N = 180 evenly spaced fabric points. Every keyframe has the same N, so any two can be interpolated point by point.
- Some keyframes are in-between poses (`sub`). The motion passes through them, but they don't count as steps and have no hold.
- Stretches at `z ≤ -9` are not drawn. The finished knots use this to hide fabric buried inside the knot.

### 5.3 Interpolation
- Between consecutive steps, `x` and `y` interpolate per point with ease-in-out.
- `z` and `face` are discrete and switch at the midpoint of the transition (or at a per-point threshold, so a pass-through visibly "threads" along the blade).
- Optional: Catmull-Rom smoothing of the spine before the outline is built, so the ribbon edges stay smooth at any frame.

### 5.4 Rendering
- An inline `<svg>` with a fixed `viewBox` (e.g. `0 0 100 100`).
- The ribbon is split into segments wherever `z` changes. Segments are painted in z-order, and each segment is drawn as a filled polygon with an outline.
- **Stripes are attached to the fabric, not the screen.** On front-face segments, each stripe is drawn as a thin quad running from the left edge at one distance along the spine to the right edge a little further along. That gives the diagonal, and the stripes move, bend and stretch with the blade. A fixed SVG `<pattern>` is not used, because the stripes would stay still while the tie moved underneath them. Stripes are clipped to their segment, and the segment outline is drawn on top of them.
- One `requestAnimationFrame` loop; the loop stops when the tab is hidden (`visibilitychange`).

### 5.5 Timeline
- A knot's timeline is the sum of its transitions, plus `hold` time on each step (default 600ms; 1200ms on the final step).
- Base transition duration: 900ms at 1×.
- Playback position is a single number `t ∈ [0, 1]` across the whole knot. The slider, its step dots and the renderer all read from `t`.
- At `t = 1` the animation holds, fades the stage briefly (~200ms), and restarts from `t = 0`.

---

## 6. Controls and behaviour

### 6.1 Knot selector
- A segmented control with three options; exactly one is active.
- Implemented as a `radiogroup` (arrow keys move between options).
- Switching knots cross-fades the stage (~250ms), resets `t = 0`, and keeps the current speed.
- The selected knot is reflected in the URL hash (`#four-in-hand`, `#half-windsor`, `#windsor`) so it can be linked. If the hash is missing or unknown, the page shows Four-in-Hand.

### 6.2 Autoplay
- The animation starts playing on load with no interaction and is always playing, except while the user is scrubbing.
- If `prefers-reduced-motion: reduce` is set, it does **not** autoplay or resume: it shows step 1 and the user drives it with the slider. Transitions then become cross-fades instead of motion.

### 6.3 Scrub slider
- A native `<input type="range">`, styled minimally: thin `--line` track, `--ink` fill to the thumb, small round thumb.
- Its value is bound to `t` (0–1000 internally for smooth dragging).
- **Tick marks** at each step boundary, as small dots under the track.
- Dragging pauses playback and renders the exact frame under the thumb. Holding the thumb still holds that frame.
- **On release, autoplay resumes immediately from that point.** There is no separate play/pause control.
- Keyboard: ← / → move to the previous / next step; Shift + ← / → move in fine increments. Playback pauses while the keys are in use and resumes 1.5s after the last key press.

### 6.4 Speed
- A three-option segmented control: **0.5× · 1× · 2×**. The default is 1×.
- Changing speed takes effect immediately and keeps the current position in the animation.

---

## 7. Accessibility
- Colour contrast meets WCAG AA for all text and controls in both themes.
- Every control is reachable and operable by keyboard; focus rings are visible (2px `--ink` outline, 2px offset).
- The SVG has `role="img"` and an `aria-label` naming the knot (for example, "Animation: tying a Half Windsor").
- The slider has `aria-valuetext` giving the step, for example "Step 3 of 8".

---

## 8. Technical
- **Deliverable:** a single `index.html` with inline CSS and JS. The only external request is the Google Fonts stylesheet, and system fonts are the fallback.
- No frameworks, no build step, and it runs by opening the file.
- Target: current Chrome, Safari, Firefox, and iOS/Android browsers.
- Budget: < 60 KB uncompressed, excluding fonts; a steady 60fps on a mid-range phone.

---

## 9. Acceptance criteria
1. On load, the headline, selector, stage and controls are visible without scrolling on a 1280×800 and a 390×844 viewport.
2. The Four-in-Hand animation starts playing within 500ms of load, with no user action.
3. All three knots animate through every listed step and loop.
4. The ribbon stays continuous in every frame, and its over/under layering is correct at every wrap and pass-through.
5. Stripes appear only on the front face and move with the fabric; the back face is plain.
6. Dragging the slider shows the matching frame immediately. Releasing it resumes playback from that frame.
7. 0.5× and 2× play at half and double the 1× duration (±5%).
8. Switching knots resets to step 1 and updates the URL hash. Reloading with a hash opens that knot.
9. With reduced motion set, nothing autoplays.
10. The only text on the page is the headline, the knot names and the speed labels.
