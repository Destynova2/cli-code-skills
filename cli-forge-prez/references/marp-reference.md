# Marp Syntax & Techniques

> **When to read:** Step 3 (Generate Marp Markdown).

---

## Basic structure

```markdown
---
marp: true
theme: uncover
paginate: true
backgroundColor: #0f0f23
color: #e0e0e0
---

# <!-- fit --> Title that auto-scales

Subtitle or one-liner

<!-- Speaker notes go here. The audience never sees this.
     Remind yourself: start with the hook, not the agenda. -->

---

## Assertion heading — a full sentence

- Max 5 bullet points
- Max 6 words per point on visual slides

<!-- Notes for this slide. What to say, not what to show. -->

---
```

Slides are separated by `---` on its own line.

## Frontmatter options

```yaml
---
marp: true
theme: uncover          # default | gaia | uncover
paginate: true          # page numbers
header: "Project Name"  # top-left on every slide
footer: "April 2026"   # bottom-left on every slide
backgroundColor: #0f0f23
color: #e0e0e0
style: |
  section {
    font-family: 'Inter', 'SF Pro', sans-serif;
  }
  pre {
    background: #1e1e3f;
    border-radius: 8px;
  }
  section.lead h1 {
    color: #ffcc00;
  }
---
```

## Auto-scaling headings

```markdown
# <!-- fit --> This heading fills the slide width
```

Works only on `#` headings. Extremely useful for title slides and single-message slides.

## Slide-local directives

Override any global setting for a single slide with `_` prefix:

```markdown
<!-- _backgroundColor: #1a1a2e -->
<!-- _color: #e94560 -->
<!-- _class: lead -->
```

Classes: `lead` (centered, large), `invert` (flipped colors).

## Background images

```markdown
![bg](image.png)                     # full bleed
![bg brightness:0.4](image.png)      # darken for text overlay
![bg blur:5px](image.png)            # blur
![bg left:40%](image.png)            # 40/60 split, image on left
![bg right:30%](image.png)           # 70/30 split, image on right
![bg brightness:0.5 blur:3px](x.png) # chained filters
```

### Multiple backgrounds (triptych)

```markdown
![bg](before.png)
![bg](during.png)
![bg](after.png)
```

Creates side-by-side images. Useful for before/during/after comparisons.

## Two-column layout

No plugin needed — inline HTML works in Marp:

```markdown
<div style="display: flex; gap: 2em;">
<div style="flex: 1;">

### Problem
- Deploys take 45 min
- 2 outages / month

</div>
<div style="flex: 1;">

### Solution
- Deploys take 3 min
- 0 outages in 6 months

</div>
</div>
```

## Code blocks

```markdown
​```rust
// Highlight the key lines with comments or keep blocks short
fn proxy(req: Request) -> Response {
    let signed = sign_ecdsa(&req);  // ← this is the point
    forward(signed)
}
​```
```

**Max 8 lines per code block.** Highlight 2-3 lines that matter. Put the rest in speaker notes.

## Speaker notes

```markdown
<!--
This is a speaker note. The audience sees nothing.
Remind yourself: pause here for 3 seconds.
Anticipated question: "Why not mTLS?" — answer: cert rotation in air-gapped.
-->
```

In HTML export, press `P` to toggle presenter view.

## Mermaid diagrams

Marp supports Mermaid via `@marp-team/marp-core` or as inline SVG. For skill usage, prefer generating the diagram with `cli-forge-schema` and embedding as an image or using Marp's HTML mode:

```markdown
<div class="mermaid">
graph LR
    Client --> Proxy --> LLM
</div>
```

Or embed the SVG directly from `cli-forge-schema` output.

## Dark theme — recommended defaults

Conference projectors in bright rooms render dark-on-light poorly. Dark backgrounds with light text project better.

```yaml
backgroundColor: #0f0f23
color: #e0e0e0
style: |
  section {
    font-family: 'Inter', sans-serif;
  }
  h1, h2 {
    color: #ffcc00;
  }
  a {
    color: #58a6ff;
  }
  code {
    color: #00ff88;
  }
  pre {
    background: #1a1a2e;
    border-radius: 8px;
    padding: 0.8em;
  }
  blockquote {
    border-left: 4px solid #ffcc00;
    padding-left: 1em;
    color: #aaaaaa;
  }
```

## Presenter view & navigation

- **Arrow keys** → navigate slides
- **P** → toggle presenter notes (HTML export)
- **F** → fullscreen
- **ESC** → overview grid

## CLI usage

```bash
# HTML standalone (one file, no dependencies)
npx @marp-team/marp-cli slides.md -o slides.html

# With custom theme
npx @marp-team/marp-cli slides.md --theme theme.css --html -o slides.html

# Preview with live reload
npx @marp-team/marp-cli slides.md --preview

# Build all decks in a directory
npx @marp-team/marp-cli --input-dir talks/ --html -o dist/

# PDF (requires Chrome/Edge/Firefox — CI only, not in sandbox)
npx @marp-team/marp-cli slides.md --pdf -o slides.pdf
```
