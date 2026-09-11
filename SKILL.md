---
name: enterprise-html-docs
description: "Use when creating styled enterprise HTML documents."
---

# Enterprise HTML Documents

## When to Use

- Creating a new styled HTML document (workflow doc, reference guide, report, meeting plan)
- Iterating on an existing HTML doc (adding sections, changing lifecycle, restyling)
- Adding GNT logo, Lucide icons, or Notion-style design to an HTML doc
- Generating print-friendly HTML + PDF from an existing HTML doc
- Any time the user sends an HTML file and asks for modifications

## Naming & Attribution Rules (MANDATORY)

- **Never mention "Iris-SPM"** anywhere in a generated document — no references, examples, code samples, or metadata.
- **Author/attribution is dynamic** — determine the author name from the Hermes agent user profile (e.g. the `user` memory store) or from what the user explicitly states. Include that name as author/attribution in the header metadata block or footer. If no name is available, ask the user before generating the document.
- **Document ID format (mandatory on every enterprise document):** `DOC-<TYPE>-<PROJECT>-YYYY-MMDD-NN`
  - Example: `DOC-CR-GE-2026-0828-01`
  - `TYPE` = document type code (e.g. CR = Change Request)
  - `PROJECT` = project short-form name (e.g. GE)
  - `YYYY-MMDD` = document date
  - `NN` = zero-padded sequence number, counted per document type + project combination (01, 02, ...)
- **Filename = Document ID + version suffix** — the file must be named `<DocumentID>-v<major>_<minor>_<patch>.<ext>`, e.g. `DOC-CR-GE-2026-0828-01-v1_0_0.html` (or `.pdf`/`.docx` matching the format). Do not use descriptive filenames. The version suffix uses underscores (e.g. `v1_0_1`) to stay filesystem-safe on all platforms.
- **Save location:** all enterprise documents go to `<HOME>/enterprise-document/<PROJECT>/` — a subfolder named after the `PROJECT` field from the Document ID (create the directory if missing). `<HOME>` is `/root` if it exists, otherwise `/home/ubuntu`. Example: `DOC-RES-SMARTROUTER-2026-0829-01-v1_0_0.html` is saved to `<HOME>/enterprise-document/SMARTROUTER/`. The base `enterprise-document/` directory should also be created if missing.

## Client-Facing Billing & CR Documents

For Change Requests, effort statements, and other billing-purpose documents:

- **Final figures only.** State billable hours/days directly. Do NOT include the derivation method — no allocation factors (e.g. "x 1/3"), no full/internal estimates, no before/after comparison columns. Internal math stays in chat reasoning; the deliverable shows only what the client is invoiced for. (These were iteratively stripped from the v1.0.0 CR — treat it as the template.)
- **Attribution:** Prepared By = the author name resolved from the Hermes agent user profile or as stated by the user — never an agent name (see Naming & Attribution Rules).
- **Same-version iterative edits:** content changes requested immediately after generation, before the doc is treated as final, keep the current version number — do not bump. Bump only when the user asks or the doc is re-delivered as a distinct iteration.
- **Structure that works:** CR overview (ID/type/project/status/source doc) → background & justification → requested change → billable breakdown → rationale → financial summary → approval sign-off block → glossary → version history.

**Editing & verification pattern for generated docs:** apply many content edits via `execute_code` with a list of `(old, new, expected_count)` tuples — `raise SystemExit` on any count mismatch — then run remnant checks (grep for removed phrases and old totals), an HTMLParser tag-balance pass, and an external-URL scan before delivering. This catches orphaned table rows and leftover phrases a visual pass misses. **Pitfall:** never name the document string `html` in a script that also does `from html.parser import HTMLParser` — the import rebinds the name to the stdlib module and `feed(html)` fails with `TypeError: can only concatenate str (not "module") to str`. Name it `doc`.

## Approval & Sign-Off Footer (MANDATORY)

Every enterprise document must include an **Approval & Sign-Off** table in the footer (before the Version History section). The table contains one row per approver role with columns: **Role | Name | Signature | Remarks | Date**. The `Signature` column is left blank (for wet/digital signing). The `Remarks` column is left blank (filled by the approver). The `Date` column is left blank (filled on sign-off).

The approver roles included depend on the **document type** (from the `TYPE` field in the Document ID). Use the mapping below as the **default maximum set** for each type, then apply the relevance filter (next subsection) to remove any role that has no direct stake in the document's subject matter. When a document spans multiple types, start with the union of all relevant roles, then filter. If the user specifies custom approver roles, use those instead.

| Document Type | TYPE Code | Required Approver Roles |
|---|---|---|
| Change Request | CR | Project Manager, Solution Architect, Client Sponsor |
| Technical Specification | TS | Solution Architect, Technical Lead, Project Manager |
| Business Analysis / Requirements | BA | Business Analyst, Project Manager, Solution Architect |
| Meeting Minutes / Decision Record | MM | Project Manager, Attendees (chair) |
| Project Plan / Schedule | PP | Project Manager, Client Sponsor |
| Risk Assessment | RA | Project Manager, Solution Architect, Risk Owner |
| Research / Investigation Report | RES | Solution Architect, Project Manager |
| Effort / Billing Statement | ES | Project Manager, Client Sponsor, Finance Approver |
| Architecture / Design Document | AR | Solution Architect, Technical Lead, Project Manager |
| Test Plan / Test Report | TP | Test Lead, Project Manager, Solution Architect |
| Release / Deployment Plan | RP | Release Manager, Solution Architect, Project Manager |
| Policy / Governance Document | POL | Governance Lead, Project Manager, Client Sponsor |
| General / Other | GEN | Project Manager, Solution Architect |

### Relevance Filter (MANDATORY)

The mapping table above provides the default role set per document type, but **not every role in the default set is necessarily relevant to the specific document being generated**. Before generating the sign-off table, evaluate each role against the document's actual content and scope. Include a role only if its responsibilities (see Role definitions below) intersect with the document's subject matter. Exclude any role that has no direct stake.

**How to determine relevance:**
- Read the document's sections and identify which stakeholder functions are actually involved (e.g., does the document involve technical architecture? financial figures? testing? deployment? governance?).
- For each role in the default set, ask: "Does this role's area of ownership appear in the document's content?" If no, exclude it.
- **Minimum:** every sign-off table must have at least one approver role. If filtering removes all roles, fall back to the full default set for that type and note the reason in chat.
- **Examples:**
  - A Change Request for a pure CSS restyling (no architecture impact) → exclude Solution Architect; keep Project Manager + Client Sponsor.
  - A Research Report on a non-technical business topic → exclude Solution Architect; keep Project Manager.
  - A Technical Specification for a deployment tool → include Solution Architect + Technical Lead; exclude Project Manager only if the doc has no timeline/scope content.
  - A Risk Assessment focused solely on schedule risk → exclude Solution Architect; keep Project Manager + Risk Owner.

**Role definitions:**
- **Project Manager (PM):** Owns timeline, scope, and delivery accountability.
- **Solution Architect (SA):** Owns technical design, feasibility, and architecture decisions.
- **Business Analyst (BA):** Owns requirements elicitation, analysis, and traceability.
- **Technical Lead:** Owns implementation approach and technical quality.
- **Client Sponsor:** Business owner funding the project; final approval authority.
- **Finance Approver:** Authorizes billing/invoicing figures (effort statements only).
- **Test Lead:** Owns test strategy, coverage, and quality gates.
- **Release Manager:** Owns deployment risk, rollback, and release coordination.
- **Governance Lead:** Owns policy compliance and audit trail.
- **Risk Owner:** Accountable for identified risk mitigation.

**HTML structure for the sign-off table:**
```html
<section id="approval-signoff">
  <h2><svg class="icon" ...>...</svg> Approval &amp; Sign-Off</h2>
  <table class="signoff-table">
    <thead>
      <tr><th>Role</th><th>Name</th><th>Signature</th><th>Remarks</th><th>Date</th></tr>
    </thead>
    <tbody>
      <tr><td>Project Manager</td><td></td><td></td><td></td><td></td></tr>
      <tr><td>Solution Architect</td><td></td><td></td><td></td><td></td></tr>
      <!-- ... one row per required role ... -->
    </tbody>
  </table>
</section>
```

**Styling:** Use the standard table style from the Design System. The `Name`, `Signature`, `Remarks`, and `Date` cells should have `min-height: 36px` (via `padding: 18px 12px`) to leave adequate signing space. The section sits as the **last content section** before Version History.

**Print consideration:** The sign-off table must fit on a single A4 page when printing. If the role count exceeds 6, add `page-break-inside: avoid` to the section to prevent splitting across pages.

## Design System (Notion-Style Light Mode)

Standard for enterprise HTML documents:

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

**Default sizing:** `height: 42px; max-width: 280px` (scale 1.5× to `63px / 420px` when the user asks for "increase by 1.5×"). Requests may specify 1.5×, 2×, or custom CSS (700px width, negative margins) — always apply the exact multiplier or CSS stated.

**Header layout:** Flexbox with logo left, metadata block right (title, version, date). Subtle bottom border separates header from H1.

**File size impact:** Base64 logo adds ~48 KB to the HTML. Acceptable for self-contained docs.

## Diagrams: Mermaid.js Standard (Inline Embedding)

Flowcharts, architecture diagrams, and sequence diagrams in HTML documentation and reports must use **Mermaid.js** with responsive container styling (`.mermaid-wrapper { display: flex; justify-content: center; overflow-x: auto; }`).

**Inline embedding is MANDATORY** — never load Mermaid from a CDN (`<script src=...>` violates Self-Contained Requirements and breaks offline `file://` use).

**Embed procedure:**
1. Source bundle: `/root/assets/mermaid.min.js` (mermaid@10, ~3.3 MB). If missing, download once: `curl -sL -o /root/assets/mermaid.min.js https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js`
2. Inline it as `<script>` content via Python `str.replace()` with a placeholder marker (same pattern as the base64 logo — never paste 3.3 MB through the patch tool).
3. Defensive escape: replace `</script` with `<\/script` in the bundle before inlining (mermaid@10 currently has zero occurrences, but other versions may not).
4. Initialize AFTER the bundle script — use the `mermaid.initialize()` config from guardrail 4 below.

**File size impact:** inline Mermaid adds ~3.4 MB to the HTML. Only embed when the document actually contains diagrams; skip entirely for diagram-free documents.

**Verified (headless chromium, mermaid@10):** inline bundle renders flowcharts to SVG — `data-processed="true"`, `aria-roledescription="flowchart-v2"`, quoted node labels and `|label|` edge labels all render.

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

Strict SemVer-style versioning applies to HTML documents:

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

Documents run locally via `file://` URLs on Windows. The HTML must be fully self-contained — all JS, CSS, and content inline; no external files of any kind:
- **No external CSS** — all styles inline in `<style>` block
- **No external JS** — all scripts inline in `<script>` blocks, including the Mermaid.js library (embed the full bundle inline — see Diagrams section; never a CDN `<script src>`)
- **No external HTML** — no `<iframe>`, `<object>`, or `<embed>`; the document is a single standalone `.html` file
- **No CDN fonts** — use system font stack, not Google Fonts `@import` or `<link>`
- **No external images** — embed as base64 data URIs (logos, icons)
- **Exception:** SVG XML namespace `http://www.w3.org/2000/svg` is required and harmless
- Add cache-bust meta tags: `<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate" />`

## Verification Checklist

Before delivering any HTML version:
1. **Tag balance:** Run HTML parser, confirm zero unclosed tags and zero mismatches
2. **Self-contained (dependencies only):** Scan for external CSS/JS/font/HTML dependencies — `grep` for `<link`, `<script src=`, `@import`, `fonts.googleapis`, `<iframe`, `<object`, `<embed`. These must NOT appear. Citation hyperlinks in the references section (e.g. `<a href="https://developer.apple.com/...">`) are allowed — the check targets *render dependencies*, not *citation links*. The only non-citation external URL should be `http://www.w3.org/2000/svg`.
3. **Version markers:** Title tag, meta footer, and version history table all show the new version
4. **Section count:** Count `<h2>` tags — matches expected section count
5. **Content integrity:** Key phrases from prior version still present (no accidental content loss)
6. **No placeholder text:** Search for `PLACEHOLDER`, `TODO`, `XXX` — none should remain (the `PLACEHOLDER_B64` logo marker must have been replaced)
7. **File size:** Note the size (base64 logo adds ~48 KB)
8. **Approval & Sign-Off table:** Confirm the `#approval-signoff` section exists with only the approver roles that are **relevant to the document's actual content** (per the Relevance Filter in the Approval & Sign-Off Footer section — not blindly the full default set from the mapping table). Each row must have all 5 columns (Role, Name, Signature, Remarks, Date) with Name/Signature/Remarks/Date left blank. Verify that every included role's area of ownership appears in the document's content, and that no irrelevant role was included.
9. **Batch verification:** Run all checks above in a single `execute_code` script — HTMLParser tag-balance pass, regex URL scan, `re.findall` for placeholders, section/reference/logo counts, a grep for `Iris-SPM`, and an **author-attribution resolved check** (grep for placeholder tokens like `AUTHOR_NAME`, `{{author}}`, `TBD`, or an empty `Prepared By` field — the author name must be a concrete resolved value, not a placeholder). Raise on any mismatch. This catches issues a visual pass misses and is faster than running each check separately.
10. **Browser render:** After the script checks pass, render in a real browser (Playwright headless Chromium via `execute_code` if `browser_exec` is unavailable) at 1280px viewport, take a full-page screenshot, and feed to `vision_analyze` for layout/contrast/clipping inspection. Do not claim verification unless a real render completed.

## Pitfalls

1. **Nested SVG icons** — wrapping a Lucide `<svg>` inside another `<svg class="icon">` creates nesting that breaks HTML parser balance. Use a single `<svg>` per icon. When stripping for print, naive regex (`<svg class="icon">.*?</svg>`) leaves orphan `</svg>` tags — use depth-counting instead.

2. **weasyprint `gap` warning** — weasyprint ignores `gap` on flex. Replace with `margin` on children. Non-fatal (just a warning) but layout won't match screen version exactly.

3. **Base64 logo injection** — don't try to inline 48 KB of base64 in a `patch` tool call. Write the HTML with a `PLACEHOLDER_B64` marker, then use Python `str.replace()` to inject the actual base64 string.

4. **Version history bookend pattern** — bold the first version (v1.0.0) and the current version in the history table; intermediate versions stay regular weight. This visually anchors "where we started" and "where we are now."

5. **Copying from wrong source** — when restoring a prior version (e.g., 24-week from 8-week), always copy from the original cached file (`doc_XXXXXXXX_*.html`), not from a derived version that may have accumulated changes.

6. **Icon stripping leaves `</svg>` orphans** — after regex-stripping `<svg class="icon">...</svg>` from headings, check for and clean any leftover `</svg>` tags at the start of heading text: `re.sub(r'(<h[23][^>]*>)\s*</svg>\s*', r'\1', content)`.

7. **Self-contained URL check false positive on citation links** — the old check `grep -oE 'https?://[^"]*' file.html` flags legitimate reference hyperlinks (e.g. `<a href="https://developer.apple.com/...">`) as violations. Scan for *render dependencies* (`<link`, `<script src=`, `@import`, Google Fonts URLs) instead of all HTTP URLs. Citation links in the references section are allowed.

8. **Mermaid DOM-grep false positives** — the inline Mermaid bundle (3.3 MB of minified JS) contains string literals like `"Syntax error in text"` and CSS class names like `error-icon`, so grepping the RAW rendered DOM returns false positives. Strip `<script>` blocks first, then verify the rendered body: `aria-roledescription="flowchart` (don't match the full value — it's `flowchart-v2`), `data-processed="true"`, and real render errors via `<g class="error-icon"`.
