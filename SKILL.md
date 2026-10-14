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

## New Document Build Procedure

1. **Resolve inputs:** author name from the Hermes user profile (ask if absent), DOC ID (`DOC-<TYPE>-<PROJECT>-YYYY-MMDD-NN`), and create `<HOME>/enterprise-document/<PROJECT>/` if missing.
2. **Author in scratch:** write the full HTML to the scratch dir (`/root/.hermes/cache/scratch/`) via `write_file` with two placeholder markers in the source: `LOGO_B64` inside the logo `<img src="data:image/svg+xml;base64,...">` and `MERMAID_JS` as the sole content of the Mermaid `<script>` tag (skip the Mermaid marker entirely for diagram-free docs). Never paste 48 KB of base64 or 3.3 MB of JS through `write_file`/`patch`.
3. **Inject and save final:** in one `execute_code` script, read the logo SVG and mermaid bundle, `assert doc.count('LOGO_B64') == 1` (and same for `MERMAID_JS`) before `str.replace()`, defensively replace `</script` with `<\/script` in the JS bundle, write the final file to the project dir, then run the full batch verification (see Verification Checklist item 9) in the same script.
4. **Render-verify locally:** copy the final HTML to `/root/<name>.html` (snap Chromium reads `/root` fine; only `~/.hermes/cache` is unreadable), run `chromium-browser --headless=new --no-sandbox --disable-gpu --disable-dev-shm-usage --dump-dom` for structural checks and `--screenshot=/root/<name>.png --window-size=1280,<height>` for visual inspection, feed the screenshot to `vision_analyze`, then delete the probe files from `/root`.
5. **Sloptrim pre-delivery de-AI pass:** before delivering the final document, run the sloptrim skill to remove AI-writing patterns from the prose (see *Sloptrim Pre-Delivery Pass* below). This is mandatory for all documents containing authored prose paragraphs (skip for pure data tables, forms, or fillable templates with no narrative content).

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
- **Jira-based cost reports:** when the user provides a Jira ticket export and asks for a development cost/effort report, use the ticket-type-based man-day estimation model in `references/jira-cost-estimation.md` — type weights (Story=8, Task=4, Sub-Task=2, Epic=0 MD), blended daily rate, and the standard report section structure for ES documents.

- **RE (hours-based) cost estimates:** when the user gives a Rough Estimate with per-role hours (BE/FE/BA/QC + overhead rows) and a man-day rate, use the hours exactly as given: cost = hours ÷ 8 × rate. ES sections: Exec Summary → Basis & Exclusions (third-party/cloud fees) → By Discipline → By Feature Group (direct only, shared overhead shown separately) → Top 10 Drivers → Detailed line items → Illustrative team/timeline (only if the RE leaves it blank; label it illustrative) → Tech Stack → Financial Summary → Sign-off (PM, Client Sponsor, Finance) → Glossary → Version History. Show man-days to 3 decimals (e.g. 428.125). Rounding to 2 decimals breaks how the totals reconcile. Reference: `/root/enterprise-document/EMP/DOC-ES-EMP-2026-1012-01-v1_0_0.html`.

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
| Performance / Salary Assessment | EVAL | Project Manager, HR Lead (Compensation & Benefits), Finance Approver |
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

**Selection rule:** Use lightmode variant for white/light backgrounds (`#FFFFFF`, `#FAFAFE`). For dark backgrounds (`#0A0A0A`, `#1A1A2E`, `#0D1117`), do NOT use the darkmode SVG directly — it has a full A4 canvas viewBox (`0 0 21000 29700`) with the logo in a tiny corner, making it invisible at 42px height. Instead, use the lightmode SVG (tight viewBox `0 0 7826 1663`) with `fill:black` replaced by `fill:#E6EDF3` (or the dark theme's text color) to make the black text elements visible on dark backgrounds:

```python
import base64
with open('/root/assets/GNT_logo_clean_lightmode.svg', 'r') as f:
    svg = f.read()
svg = svg.replace('.fil3 {fill:black;fill-rule:nonzero}', '.fil3 {fill:#E6EDF3;fill-rule:nonzero}')
b64 = base64.b64encode(svg.encode('utf-8')).decode('ascii')
```

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

Flowcharts, architecture diagrams, and sequence diagrams in HTML documentation and reports must use **Mermaid.js** with responsive container styling. The wrapper and pre element must both be sized correctly to prevent the SVG from shrinking or overflowing:

```css
.mermaid-wrapper { display: flex; justify-content: center; overflow-x: auto; margin-bottom: 16px; padding: 12px 0; }
.mermaid-wrapper .mermaid { font-size: 15px !important; width: 100% !important; }
.mermaid-wrapper svg { max-width: 100% !important; height: auto !important; }
```

The `width: 100%` on `.mermaid` (the `<pre>` element) is MANDATORY — without it, the flex child shrinks to content width (~300px) and the SVG's `width: 100%` resolves against that shrunken parent instead of the full card width.

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

## Theme Variants (Dark ↔ Light)

A restyled copy of an existing document (same content, other theme) is a **minor version bump** — new file (e.g. `v1_1_0`), prior version preserved, version-history row added describing the variant. Convert programmatically, never by hand:

1. Swap the `:root` token block (dark `#0D1117/#161B22/#1C2129/#E6EDF3/#8B949E/#30363D` ↔ light `#FFFFFF/#F7F7F5/#F1F1EF/#1A1A2E/#6B7280/#E5E7EB`), keeping `--accent` and mapping semantic colors to the target palette's variants (e.g. `--good` `#3FB950`→`#059669`, `--bad` `#F85149`→`#DC2626`, `--info` `#58A6FF`→`#2563EB`).
2. Swap the body text color (`#E6EDF3` ↔ `#37352F`), inline-code color (`#FF7B72` dark ↔ `#EB5757` light), and any hard-coded `rgba(...)` tag/badge backgrounds and gradient stops to the target palette's equivalents.
3. Re-embed the logo for the target theme (light bg: lightmode SVG as-is; dark bg: lightmode SVG with `fill:black` → theme text color — see GNT Logo Embedding). Do this by re-injecting base64 over a marker, not by editing the existing data URI.
4. Update version markers in `<title>`, header meta, and footer, and add the version-history row (bold the current version, unbold the previous).
5. Verify with a **dark-remnant check**: grep the converted file for every hex token of the source theme — any survivor means a hard-coded color outside the token block. Then run the full verification battery (tag balance, self-contained, content integrity) and a real render; confirm content is identical apart from style, version markers, and logo.

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

**Pre-flight enterprise compliance gate (MANDATORY before PDF generation):** Before running weasyprint on any HTML document, verify it has ALL enterprise-required elements: GNT logo (base64-embedded, correct light/dark variant), frontmatter header (DOC ID, author, date, version, status), Approval & Sign-Off section, Glossary, and Version History. If any are missing, add them to the HTML first, then generate the PDF. Generating a PDF without these elements produces a non-compliant deliverable that will be rejected.

**Key steps:**
1. **Enterprise compliance check** — verify GNT logo, frontmatter, sign-off, glossary, version history are present in the HTML; add any missing elements before proceeding
2. Copy the latest screen-version HTML
3. Replace `<style>` block with print-optimized CSS (A4 @page, smaller fonts, page-break rules)
4. Strip inline Lucide SVG icons (they're too small to print legibly) — use the depth-counting regex in `references/print-pipeline.md`
5. Add `@page` rules: `size: A4; margin: 18mm 14mm 20mm 14mm` with running header and page counter
6. Add `page-break-inside: avoid` on table rows, diagram cards, and callouts
7. Add `break-after: avoid` on h2/h3 headings
8. **Convert CSS grid → flexbox** — WeasyPrint does not support `display: grid` at all (not just `gap`); grids are silently ignored and children stack vertically. Replace `display: grid; grid-template-columns: repeat(auto-fit, minmax(Npx, 1fr)); gap: M` with `display: flex; flex-wrap: wrap` + `flex: 1 1 Npx; margin: M/2` on children. See `references/print-pipeline.md` for the full conversion pattern.
9. Render: `weasyprint input.html output.pdf`
10. Verify with `pdfinfo output.pdf` (check page count, page size = A4)

**weasyprint quirks:**
- Does NOT support `display: grid` at all — the entire grid layout is silently ignored and children stack vertically. This is NOT harmless when grids carry critical layout (stats grids, feature cards, comparison grids). Convert all `display: grid` to `display: flex; flex-wrap: wrap` with `flex: 1 1 <min>px` on children before rendering.
- Ignores CSS `gap` property on flex containers — use `margin` on children instead
- Ignores `@media` queries with feature syntax like `(max-width: 640px)` — wrap responsive rules in `@media screen` or inline them
- `running()` header works with `position: running(hdr)` + `@top-left { content: element(hdr) }`
- Page counter: `@bottom-center { content: counter(page) " of " counter(pages) }`
- Max SVG diagram height should be constrained to ~140mm to fit within A4 margins

## Sloptrim Pre-Delivery Pass

Enterprise documents represent professional deliverables. AI-generated prose carries recognizable patterns (AI vocabulary, significance inflation, signposting, transition clusters, canonical slop phrases) that undermine credibility. The sloptrim skill (installed at `~/.hermes/skills/productivity/sloptrim/`) removes these patterns before delivery.

**Mandatory for:** all documents with authored prose paragraphs (research reports, change requests, specifications, assessments, plans).
**Skip for:** pure data tables, fillable forms with no narrative, templates with placeholder text only.

### Procedure

Run this as a single `execute_code` script after step 4 (render-verify) and before delivery:

1. **Extract prose from HTML:** strip `<script>` and `<style>` blocks, then extract text from `<p>`, `<li>`, `<td>`, and callout elements using regex. Write extracted text to a temp file.
   ```python
   import re
   with open(html_path) as f:
       html = f.read()
   html_clean = re.sub(r'<(script|style)[^>]*>.*?</\1>', '', html, flags=re.DOTALL)
   paras = re.findall(r'<(?:p|li|td)[^>]*>(.*?)</(?:p|li|td)>', html_clean, re.DOTALL)
   text = '\n\n'.join(re.sub(r'<[^>]+>', '', p).strip() for p in paras if re.sub(r'<[^>]+>', '', p).strip())
   ```

2. **Score with detect.py:** run `python3 "$DETECT" /tmp/extracted_text.txt` where `DETECT=~/.hermes/skills/productivity/sloptrim/scripts/detect.py`. Read `ai_tell_score` and `ai_tell_band` from the `_metrics` block.
   - Score ≤ 20 (clean): skip rewrite, proceed to delivery. The prose is already human-quality.
   - Score 21-40 (light tells): targeted edits to flagged spans only, then re-score.
   - Score > 40 (mixed/heavy/pervasive): full rewrite of prose paragraphs toward the plain/clear style profile (business prose = conservative mode), then re-score.

3. **Rewrite prose in-place within the HTML:** replace each flagged paragraph's text content using `str.replace()` with `(old, new, expected_count)` assertions. Never run `detect.py --clean` on the HTML file — it normalizes characters in tag attributes and can break the document. Rewrite prose manually (the LLM does this, not the script).

4. **Re-score the cleaned extracted text:** extract again, run detect.py, confirm score dropped to ≤ 25 (light tells or clean). If still > 40, do a second pass (max 2 loops per the sloptrim skill).

5. **Verify HTML integrity after rewrite:**
   - Tag balance: run HTMLParser, confirm zero unclosed/mismatched tags (see Verification Checklist item 1, skip void elements).
   - Enterprise elements: DOC ID, author name, sign-off section, version history, glossary, callout blocks all present.
   - Numbers preserved: every number, date, percentage, unit, proper noun, and hyphenated technical term from the original must survive the rewrite. Grep for key figures.
   - Word count: output must not exceed 1.25x the input word count (sloptrim cuts, never grows).

6. **Deliver:** if all checks pass, the document is ready. If any check fails, fix and re-verify before delivery.

### What gets removed

The 71-pattern catalogue targets: AI vocabulary (leverage, crucial, transformative, delve, tapestry, showcase, robust, paradigm, harness, holistic, seamless, unlock, unprecedented, multifaceted, elevate, etc.), compulsive intro hooks ("In today's fast-paced world"), signposting ("Let's explore"), editorial interjections ("It's important to note that"), transition clusters (Additionally/Furthermore stacking), significance inflation, canonical slop phrases ("stay ahead of the curve", "drive meaningful impact"), sentence-length monotony, and 60+ other patterns.

### What gets preserved

All factual content: numbers, units, dates, proper nouns, citations, hyphenated technical terms, attributed claims, hedges, rankings, placeholders (`[Name]`, `{{field}}`), and the document's HTML structure, CSS, scripts, and enterprise compliance elements.

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
4. **Section count:** Count `<h2>` tags — matches expected section count. Derive the expected count from the authored structure, not a guess: every content section PLUS the Approval & Sign-Off, Glossary, and Version History sections each contribute one `<h2>`. Asserting a guessed number fails on a correct document and wastes a debug cycle.
5. **Content integrity:** Key phrases from prior version still present (no accidental content loss)
6. **No placeholder text:** Search for `PLACEHOLDER`, `TODO`, `XXX` — none should remain (the `PLACEHOLDER_B64` logo marker must have been replaced)
7. **File size:** Note the size (base64 logo adds ~48 KB)
8. **Approval & Sign-Off table:** Confirm the `#approval-signoff` section exists with only the approver roles that are **relevant to the document's actual content** (per the Relevance Filter in the Approval & Sign-Off Footer section — not blindly the full default set from the mapping table). Each row must have all 5 columns (Role, Name, Signature, Remarks, Date) with Name/Signature/Remarks/Date left blank. Verify that every included role's area of ownership appears in the document's content, and that no irrelevant role was included.
9. **Batch verification:** Run all checks above in a single `execute_code` script — HTMLParser tag-balance pass, regex URL scan, `re.findall` for placeholders, section/reference/logo counts, a grep for `Iris-SPM`, and an **author-attribution resolved check** (grep for placeholder tokens like `AUTHOR_NAME`, `{{author}}`, `TBD`, or an empty `Prepared By` field — the author name must be a concrete resolved value, not a placeholder). Raise on any mismatch. This catches issues a visual pass misses and is faster than running each check separately.
10. **Browser render:** After the script checks pass, render in a real browser (Playwright headless Chromium via `execute_code` if `browser_exec` is unavailable) at 1280px viewport, take a full-page screenshot, and feed to `vision_analyze` for layout/contrast/clipping inspection. Do not claim verification unless a real render completed.
    **Local `file://` render fallback:** cloud-hosted browser tools cannot open local `file://` URLs (blocked as private/internal address). Render locally with system Chromium instead: `chromium-browser --headless=new --no-sandbox --disable-gpu --disable-dev-shm-usage --dump-dom 'file://<path>'` for structural checks (grep the DOM dump for h2 count, signoff-table, logo class), and `--screenshot=/root/<name>.png --window-size=1280,<height>` for visual inspection. Write probe HTML and screenshots directly under `/root/` — snap Chromium reads and writes `/root` paths without redirection, so the private-tmp workaround below only applies to `/tmp` targets. Snap-confined Chromium writing to `/tmp/<name>.png` redirects into its private namespace (`/tmp/snap-private-tmp/snap.chromium/tmp/<name>.png`) and may report "Permission denied" at the requested path while succeeding silently — glob the private-tmp path and copy. Capture tall pages by setting an explicit large `--window-size` height (there is no full-page flag), then crop regions with PIL and feed the crops to `vision_analyze`.
11. **Mermaid diagram size check:** If the document contains Mermaid diagrams, inject the size-check JS snippet (see `references/mermaid-size-check.md`) into the rendered HTML, re-render in headless Chromium, and extract the JSON report from `#size-report`. Every diagram must return `status: "PASS"`. If any diagram returns `FAIL`, fix the diagram layout (restructure wide chains, adjust wrapper sizing, change flow direction) and re-render until all pass. Common failures: too small (rendered width < 200px), horizontal clipping (scroll overflow), extreme aspect ratio (> 10:1), over-shrunk (scale < 0.3 on a large viewBox).
12. **Sloptrim de-AI verification:** if the document contains authored prose (step 5 of the build procedure), confirm the sloptrim pre-delivery pass ran and the re-scored `ai_tell_score` is ≤ 25. Verify all enterprise elements (DOC ID, author, sign-off, version history, glossary) and key numbers survived the rewrite. Confirm HTML tag balance is still zero errors after the prose was modified in-place. If the document has no authored prose (pure data/forms), this step is skipped.

## Pitfalls

1. **Nested SVG icons** — wrapping a Lucide `<svg>` inside another `<svg class="icon">` creates nesting that breaks HTML parser balance. Use a single `<svg>` per icon. When stripping for print, naive regex (`<svg class="icon">.*?</svg>`) leaves orphan `</svg>` tags — use depth-counting instead.

2. **weasyprint does NOT support CSS grid** — `display: grid` is silently ignored entirely, causing children to stack vertically. This is NOT a harmless warning when grids carry critical layout (stats grids, feature cards, comparison matrices). Convert all `display: grid; grid-template-columns: repeat(auto-fit, minmax(Npx, 1fr)); gap: M` to `display: flex; flex-wrap: wrap` + `flex: 1 1 Npx; margin: M/2` on children before rendering. Also replace `gap` on flex containers with `margin` on children. See `references/print-pipeline.md` for the full conversion pattern.

3. **Base64 logo injection** — don't try to inline 48 KB of base64 in a `patch` tool call. Write the HTML with a `PLACEHOLDER_B64` marker, then use Python `str.replace()` to inject the actual base64 string.

4. **Version history bookend pattern** — bold the first version (v1.0.0) and the current version in the history table; intermediate versions stay regular weight. This visually anchors "where we started" and "where we are now."

5. **Copying from wrong source** — when restoring a prior version (e.g., 24-week from 8-week), always copy from the original cached file (`doc_XXXXXXXX_*.html`), not from a derived version that may have accumulated changes.

6. **Icon stripping leaves `</svg>` orphans** — after regex-stripping `<svg class="icon">...</svg>` from headings, check for and clean any leftover `</svg>` tags at the start of heading text: `re.sub(r'(<h[23][^>]*>)\s*</svg>\s*', r'\1', content)`.

7. **Self-contained URL check false positive on citation links** — the old check `grep -oE 'https?://[^"]*' file.html` flags legitimate reference hyperlinks (e.g. `<a href="https://developer.apple.com/...">`) as violations. Scan for *render dependencies* (`<link`, `<script src=`, `@import`, Google Fonts URLs) instead of all HTTP URLs. Citation links in the references section are allowed.

8. **Mermaid bundle string-literal false positives in multiple checks** — the inline Mermaid bundle (3.3 MB of minified JS) contains string literals like `"Syntax error in text"`, CSS class names like `error-icon`, and HTML fragments like `<iframe style=...`. These trigger false positives in BOTH the self-contained dependency scan (grepping for `<iframe`/`<object`/`<embed`) AND the rendered-DOM verification (grepping for `error-icon`). Strip `<script>` blocks from the document string before running any grep-based check — for the self-contained scan, for the Mermaid render verification, and for placeholder scans. After stripping, verify the rendered body via: `aria-roledescription="flowchart` (don't match the full value — it's `flowchart-v2`), `data-processed="true"`, and real render errors via `<g class="error-icon"`.

9. **HTMLParser void-element false positives in tag-balance check** — the `handle_endtag` callback fires for self-closing void elements written with `/>` syntax (`<meta ... />`, `<img ... />`, `<input ... />`), producing "Unmatched close" errors that are false positives. If the stack is empty after processing all content, the HTML is well-formed. Fix: skip void elements in `handle_endtag` — check `tag not in ('meta','img','input','br','hr','link','area','base','col','embed','source','track','wbr')` before appending an error. Alternatively, treat "Unmatched close" entries for void-element tag names as non-errors when the stack is empty.

10. **Placeholder scan false positives from embedded content** — the verification step that greps for `TBD`, `PLACEHOLDER`, `TODO`, etc. matches substrings in three sources: (a) base64-encoded content (logos, embedded images) — `xml:space="preserve"` base64-encodes to a sequence containing `tbd`; (b) the inline Mermaid bundle (see pitfall 8); (c) **Jira status values** in document content — the status `"TO DO"` matches the `TODO` token case-insensitively. Strip `data:[^;]+;base64,[A-Za-z0-9+/=]+` and `<script>` blocks from the document string, and use case-sensitive matching for `TODO` (or exclude known status values) before running `re.findall` for placeholder tokens — same principle as pitfall 7 (citation-link false positives) and pitfall 8 (Mermaid DOM-grep false positives).

11. **`@media print` in dark-themed HTML destroys contrast in weasyprint PDF** — a `@media print { body { background: #fff; color: #000; } .card { background: #f5f5f5; } }` block forces white backgrounds and black text, but does NOT override dark-theme CSS variables (`--text-muted`, `--border`, colored `td.green`/`td.warn`/`td.red` classes). The result: muted text, table borders, and colored cells become invisible on white. Remove the `@media print` block entirely from dark-themed HTML before running weasyprint — the dark theme CSS renders correctly as-is in PDF output.

12. **Chromium OOM with large inline JS bundles** — on constrained VMs (≤4 vCPU / 16 GB), the 3.3 MB inline Mermaid bundle can crash headless Chromium during full-page screenshot rendering with `V8 process OOM (Failed to reserve virtual memory for CodeRange)`. Both Playwright and `chromium-browser --screenshot` fail; `browser_exec` may also fail with `CDP response channel closed`. **Fallback hierarchy:** (a) `chromium-browser --headless --no-sandbox --disable-gpu --disable-dev-shm-usage --dump-dom 'file://...path...' > /tmp/dom_dump.txt` still works for structural verification (grep for h2 count, `signoff-table`, `gnt-logo`, `role-cell`). (b) For diagram size measurement specifically, launch Chromium with `--remote-debugging-port=9222`, connect via Python `websockets` to the page-level CDP endpoint from `http://localhost:9222/json`, navigate with `Page.navigate`, wait for render, then call `Runtime.evaluate` with a JS snippet that reads `getBoundingClientRect()` on each SVG. This CDP approach handles the 3.3 MB bundle without the OOM that kills Playwright's multi-process model. Do not loop on screenshot retries — switch to DOM-dump or CDP immediately after the first OOM crash.

13. **Mermaid `<pre>` flex-child shrinking** — when `.mermaid-wrapper` uses `display: flex`, the child `<pre class="mermaid">` has no intrinsic width and shrinks to content size (~300px). The SVG's `width: 100%` then resolves against the shrunken pre, producing tiny diagrams despite `useMaxWidth: true`. Fix: set `width: 100% !important` on `.mermaid-wrapper .mermaid`. Do NOT remove `useMaxWidth: true` to compensate — that causes overflow instead. The correct combination is `useMaxWidth: true` (Mermaid config) + `width: 100%` (CSS on pre) + `max-width: 100%` (CSS on SVG).

14. **Angle-bracket placeholders in code/command content** — writing `<prompt>`, `<name>`, `<session>`, `<id>`, `<slug>` literally inside `<code>` or table cells makes the HTMLParser tag-balance check treat them as open tags, failing with confusing `mismatch: expected slug, got code` errors. Entity-escape every placeholder angle bracket at authoring time: `<prompt>`.

15. **Entity injection via execute_code** — when a post-build fix must insert `<`/`>` entities, build them with `chr(38) + 'lt;'` concatenation inside the executed code. Literal `<` strings written into the code payload can arrive pre-normalized, so `str.replace('<prompt>', '<prompt>')` silently no-ops and the tag-balance failure repeats identically. Always print the replacement count and assert it is non-zero.

17. **Vision model false-positive clipping reports**: on tall screenshot crops, vision_analyze can report "clipped" text and columns that aren't actually clipped. Check against the DOM before changing anything. Inject an onload script that writes `scrollWidth` vs `clientWidth` for the page and every `.tw` table wrapper into a `<pre id=probe>`, then `--dump-dom` and grep. Snap Chromium can't read files under `~/.hermes/cache`, so write the probe file to `/root/` and delete it afterwards.

18. **detect.py on raw HTML gives inaccurate scores** — running `detect.py` directly on a `.html` file works but produces inflated character counts and diluted pattern detection because HTML tags, CSS, and attributes are scanned as text. Always extract prose from `<p>`, `<li>`, `<td>` elements first, then run detect.py on the extracted plain text. The raw-HTML score runs ~20-30% lower than the extracted-text score, masking severe AI-slop problems.

19. **detect.py --clean destroys HTML structure** — the `--clean` flag normalizes characters across the entire file including tag attributes, class names, and data URIs. Never run `--clean` on an HTML file. Rewrite prose in-place using `str.replace()` with count assertions, then re-score the extracted text.

20. **Sloptrim rewrite must not change HTML structure** — when replacing AI-slop prose with clean text, use exact string matching on the text content between tags. Never alter tag names, attributes, class names, IDs, or nesting. After every rewrite, run the HTMLParser tag-balance check (Verification Checklist item 1) and verify enterprise elements (DOC ID, sign-off, version history, glossary) are intact. A prose rewrite that breaks HTML structure is a failed delivery, no matter how clean the text scores.

21. **Sloptrim word-count limit applies to prose, not full HTML** — the 1.25x word-count ceiling is measured on extracted prose text, not the HTML file size. The HTML file may shrink (sloptrim cuts prose) or stay similar (CSS/JS/tables unchanged). Measure word count on the extracted text before and after the rewrite.

16. **Full-page screenshot height guessing** — headless Chromium `--screenshot` has no full-page flag; a `--window-size` height larger than the document renders the surplus as blank white. Treat an all-white bottom crop as "window taller than document — shrink and re-render", not a render failure; one or two shrink iterations converge. Also run build scripts with `exec(open(path).read())` — `%run` is IPython magic and raises SyntaxError in the execute_code kernel.
