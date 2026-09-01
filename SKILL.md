---
name: enterprise-html-docs
description: "Use when creating styled enterprise HTML docs for David."
---

# Enterprise HTML Documents

## When to Use

- Creating a new styled HTML document (workflow doc, reference guide, report, meeting plan)
- Iterating on an existing HTML doc (adding sections, changing lifecycle, restyling)
- Adding GNT logo, Lucide icons, or Notion-style design to an HTML doc
- Generating print-friendly HTML + PDF from an existing HTML doc
- Any time David sends an HTML file and asks for modifications

## Naming & Attribution Rules (MANDATORY)

- **Never mention "Iris-SPM"** anywhere in a generated document — no references, examples, code samples, or metadata.
- **Every document must mention "Liew Wei Sung"** — include as author/attribution in the header metadata block or footer.
- **Document ID format (mandatory on every enterprise document):** `DOC-<TYPE>-<PROJECT>-YYYY-MMDD-NN`
  - Example: `DOC-CR-GE-2026-0828-01`
  - `TYPE` = document type code (e.g. CR = Change Request)
  - `PROJECT` = project short-form name (e.g. GE)
  - `YYYY-MMDD` = document date
  - `NN` = zero-padded sequence number, counted per document type + project combination (01, 02, ...)
- **Filename = Document ID + version suffix** — the file must be named `<DocumentID>-v<major>_<minor>_<patch>.<ext>`, e.g. `DOC-CR-GE-2026-0828-01-v1_0_0.html` (or `.pdf`/`.docx` matching the format). Do not use descriptive filenames. The version suffix uses underscores (e.g. `v1_0_1`) to stay filesystem-safe on all platforms.
- **Save location:** all enterprise documents go to `/root/enterprise-document/` (create the directory if missing).

## Client-Facing Billing & CR Documents

For Change Requests, effort statements, and other billing-purpose documents:

- **Final figures only.** State billable hours/days directly. Do NOT include the derivation method — no allocation factors (e.g. "x 1/3"), no full/internal estimates, no before/after comparison columns. Internal math stays in chat reasoning; the deliverable shows only what the client is invoiced for. (David iteratively stripped these from the v1.0.0 CR — treat it as the template.)
- **Attribution:** Prepared By = "Liew Wei Sung" — never an agent name (see Naming & Attribution Rules).
- **Same-version iterative edits:** content changes requested immediately after generation, before the doc is treated as final, keep the current version number — do not bump. Bump only when the user asks or the doc is re-delivered as a distinct iteration.
- **Structure that works:** CR overview (ID/type/project/status/source doc) → background & justification → requested change → billable breakdown → rationale → financial summary → approval sign-off block → glossary → version history.

**Editing & verification pattern for generated docs:** apply many content edits via `execute_code` with a list of `(old, new, expected_count)` tuples — `raise SystemExit` on any count mismatch — then run remnant checks (grep for removed phrases and old totals), an HTMLParser tag-balance pass, and an external-URL scan before delivering. This catches orphaned table rows and leftover phrases a visual pass misses. **Pitfall:** never name the document string `html` in a script that also does `from html.parser import HTMLParser` — the import rebinds the name to the stdlib module and `feed(html)` fails with `TypeError: can only concatenate str (not "module") to str`. Name it `doc`.

## Design System (Notion-Style Light Mode)

David's standard for enterprise HTML documents:

| Token | Value | Usage |
|---|---|---|
| `--bg` | `#FFFFFF` | Page background |
| `--surface` | `#F7F7F5` | Cards, callouts, table headers |
| `--ink` | `#1A1A2E` | Headings, primary text |
| Body text | `#37352F` | Paragraphs (Notion's near-black) |
| `--muted` | `#6B7280` | Subtitles, meta, captions |
| `--line` | `#E5E7EB` | Table borders, dividers |
| `--line-soft` | `#EDECE9` | Card borders, subtle dividers |
| `--accent` | `#5333FF` | Links, icon highlights, traceability arrows |
| `--hover` | `#F1F1EF` | Table row hover |

**Font stack:** `-apple-system, BlinkMacSystemFont, "Segoe UI", "Helvetica Neue", "Source Sans 3", system-ui, sans-serif` — no CDN import, works under `file://` on Windows.

**Code font:** `'JetBrains Mono', ui-monospace, SFMono-Regular, Menlo, monospace` with `background: rgba(135,131,120,0.15); color: #EB5757; border-radius: 3px`.

**Callout style:** `background: #F7F6F3; border-left: 3px solid var(--warn); padding: 14px 16px; border-radius: 4px` — Notion's signature left-border callout with icon + content in a flex row.

**Table style:** `border-collapse: collapse; border: 1px solid var(--line-soft); border-radius: 6px; overflow: hidden` — th has `background: var(--surface)`, td has `border-bottom: 1px solid var(--line-soft)`, row hover `background: var(--hover)`.

**Card style:** `background: var(--surface); border: 1px solid var(--line-soft); border-radius: 8px; padding: 20px; overflow-x: auto` for diagram containers.

## Inline Lucide Icons

Embed hand-tuned Lucide-style SVG icons inline next to headings and in legends. No CDN.

**Spec:** 24×24 viewBox, `fill="none"`, `stroke="currentColor"`, `stroke-width="2"`, `stroke-linecap="round"`, `stroke-linejoin="round"`.

**CSS:**
```css
.icon{width:1em;height:1em;display:inline-block;vertical-align:-0.125em;stroke:currentColor;fill:none;stroke-width:2;flex-shrink:0}
h2 > .icon{color:var(--accent);width:24px;height:24px}
h3 > .icon{color:var(--muted);width:16px;height:16px}
```

**H2 pattern:** `<h2 id="sec-N"><svg class="icon" ...>...</svg> Section Title</h2>`

**Common icons by section type:**
- Overview/intro → file-text
- Timeline/schedule → calendar
- Flow/process → arrow-right, workflow
- Traceability → link
- People/stakeholders → users
- Risk/warning → alert-triangle
- History/version → clock, history
- Glossary → book-open, message-square
- Storage → database
- Checkmarks → check-circle

**Pitfall — nested SVGs:** Lucide icons contain a single `<svg>` root. If you wrap them in an outer `<svg class="icon">` container (as some templates do), you get nested SVGs that break HTML parser balance. Always use a single `<svg>` element per icon. If stripping icons for print, use depth-counting regex (see `references/print-pipeline.md`).

## GNT Logo Embedding

**Source files:**
- Light mode: `/root/assets/GNT_logo_clean_lightmode.svg` (36 KB)
- Dark mode: `/root/assets/GNT_logo_clean_darkmode.svg` (36 KB)

**Selection rule:** Use lightmode variant for white/light backgrounds (`#FFFFFF`, `#FAFAFE`). Use darkmode variant for dark backgrounds (`#0A0A0A`, `#1A1A2E`).

**Embedding technique:** Base64-encode the SVG file and embed as `<img src="data:image/svg+xml;base64,...">`. This keeps the HTML self-contained (no external file dependency, works under `file://`).

```python
import base64
with open('/root/assets/GNT_logo_clean_lightmode.svg','rb') as f:
    b64 = base64.b64encode(f.read()).decode('ascii')
# Inject into HTML: <img src="data:image/svg+xml;base64,{b64}" class="gnt-logo" />
```

**Default sizing:** `height: 42px; max-width: 280px` (scale 1.5× to `63px / 420px` when David asks for "increase by 1.5×"). David has requested 1.5×, 2×, and custom CSS (700px width, negative margins) in different sessions — always apply the exact multiplier or CSS he specifies.

**Header layout:** Flexbox with logo left, metadata block right (title, version, date). Subtle bottom border separates header from H1.

**File size impact:** Base64 logo adds ~48 KB to the HTML. Acceptable for self-contained docs.

## Diagrams: Mermaid.js Standard

David mandates that flowcharts, architecture diagrams, and sequence diagrams in HTML documentation and reports use **Mermaid.js** (`<script src="https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js"></script>`) with responsive container styling (`.mermaid-wrapper { display: flex; justify-content: center; overflow-x: auto; }`).

### Critical Mermaid Syntax Guardrails (Avoid "Syntax error in text")
1. **Always quote node labels**: Enclose labels in double quotes, e.g. `A["Step 1: Go Build"] --> B{"Passed?"}`.
2. **Never embed raw square brackets inside labels**: Bracket characters `[` and `]` without quotes or nested inside unescaped strings clash with Mermaid node delimiters. Use descriptive unnested text (e.g. `tags: wiki-ref, golang`) or HTML entities (`&#91;` and `&#93;`).
3. **Subgraph syntax**: Use explicit IDs and quoted titles, e.g. `subgraph L1["L1: Local Context"]`.
4. **Initialization config**: Initialize with `securityLevel: "loose"`, `htmlLabels: true`, and `useMaxWidth: true`:
   ```javascript
   mermaid.initialize({
     startOnLoad: true,
     theme: "dark", // or default matching theme
     securityLevel: "loose",
     flowchart: { useMaxWidth: true, htmlLabels: true, curve: "basis" }
   });
   ```

## Versioning Convention

David follows strict SemVer-style versioning for HTML documents:

| Change Type | Version Bump | File Action | Example Filename |
|---|---|---|---|
| Major restructure / role model change | Major (v1→v2) | New file | `DOC-CR-GE-2026-0828-01-v2_0_0.html` |
| New section, lifecycle variant, glossary | Minor (v1.5→v1.6) | **New file**, preserve previous | `DOC-CR-GE-2026-0828-01-v1_6_0.html` |
| Typo fix, table correction, CSS tweak | Patch (v1.6.0→v1.6.1) | **Rename** existing file | `DOC-CR-GE-2026-0828-01-v1_6_1.html` |

- **Rules:**
  - ALL previous versions preserved on disk — never delete or overwrite
  - Minor bumps create a new file; the prior version stays untouched
  - Patch bumps rename the existing file (no new file)
  - Lifecycle variants (e.g., 8-week vs 24-week) are first-class versions — preserve both
  - Always add a Version History table (§N) as a section in the document itself
  - Always include a Glossary of Terms table when producing enterprise architecture, comparison matrices, or technical specifications
  - Update the `<title>` tag, meta footer, and version history table on every version change

## Print-Friendly HTML + PDF Pipeline

See `references/print-pipeline.md` for the full A4 @page CSS template and weasyprint command.

**Key steps:**
1. Copy the latest screen-version HTML
2. Replace `<style>` block with print-optimized CSS (A4 @page, smaller fonts, page-break rules)
3. Strip inline Lucide SVG icons (they're too small to print legibly) — use the depth-counting regex in `references/print-pipeline.md`
4. Add `@page` rules: `size: A4; margin: 18mm 14mm 20mm 14mm` with running header and page counter
5. Add `page-break-inside: avoid` on table rows, diagram cards, and callouts
6. Add `break-after: avoid` on h2/h3 headings
7. Render: `weasyprint input.html output.pdf`
8. Verify with `pdfinfo output.pdf` (check page count, page size = A4)

**weasyprint quirks:**
- Ignores CSS `gap` property on flex containers — use `margin` instead
- `running()` header works with `position: running(hdr)` + `@top-left { content: element(hdr) }`
- Page counter: `@bottom-center { content: counter(page) " of " counter(pages) }`
- Max SVG diagram height should be constrained to ~140mm to fit within A4 margins

## Self-Contained Requirements

David runs HTML locally via `file://` URLs on Windows. The HTML must be fully self-contained:
- **No external CSS** — all styles inline in `<style>` block
- **No external JS** — any scripts must be inline
- **No CDN fonts** — use system font stack, not Google Fonts `@import` or `<link>`
- **No external images** — embed as base64 data URIs (logos, icons)
- **Exception:** SVG XML namespace `http://www.w3.org/2000/svg` is required and harmless
- Add cache-bust meta tags: `<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate" />`

## Verification Checklist

Before delivering any HTML version:
1. **Tag balance:** Run HTML parser, confirm zero unclosed tags and zero mismatches
2. **Self-contained:** `grep -oE 'https?://[^"]*' file.html` — only `http://www.w3.org/2000/svg` should appear
3. **Version markers:** Title tag, meta footer, and version history table all show the new version
4. **Section count:** Count `<h2>` tags — matches expected section count
5. **Content integrity:** Key phrases from prior version still present (no accidental content loss)
6. **No placeholder text:** Search for `PLACEHOLDER`, `TODO`, `XXX` — none should remain
7. **File size:** Note the size (base64 logo adds ~48 KB)

## Pitfalls

1. **Nested SVG icons** — wrapping a Lucide `<svg>` inside another `<svg class="icon">` creates nesting that breaks HTML parser balance. Use a single `<svg>` per icon. When stripping for print, naive regex (`<svg class="icon">.*?</svg>`) leaves orphan `</svg>` tags — use depth-counting instead.

2. **weasyprint `gap` warning** — weasyprint ignores `gap` on flex. Replace with `margin` on children. Non-fatal (just a warning) but layout won't match screen version exactly.

3. **Base64 logo injection** — don't try to inline 48 KB of base64 in a `patch` tool call. Write the HTML with a `PLACEHOLDER_B64` marker, then use Python `str.replace()` to inject the actual base64 string.

4. **Version history bookend pattern** — bold the first version (v1.0.0) and the current version in the history table; intermediate versions stay regular weight. This visually anchors "where we started" and "where we are now."

5. **Copying from wrong source** — when restoring a prior version (e.g., 24-week from 8-week), always copy from the original cached file (`doc_XXXXXXXX_*.html`), not from a derived version that may have accumulated changes.

6. **Icon stripping leaves `</svg>` orphans** — after regex-stripping `<svg class="icon">...</svg>` from headings, check for and clean any leftover `</svg>` tags at the start of heading text: `re.sub(r'(<h[23][^>]*>)\s*</svg>\s*', r'\1', content)`.
