# Print Pipeline Reference

## A4 @page CSS Template

Replace the screen `<style>` block with this print-optimized version:

```css
<style>
  @page{
    size:A4;
    margin:18mm 14mm 20mm 14mm;
    @top-left{content:element(hdr);}
    @bottom-center{content:counter(page) " of " counter(pages);}
  }
  @page :first{margin-top:8mm}

  :root{
    --ink:#1A1A2E;--muted:#6B7280;--line:#D1D5DB;--accent:#5333FF;
  }
  *{box-sizing:border-box;margin:0;padding:0}
  html,body{
    font-family:-apple-system,BlinkMacSystemFont,'Segoe UI','Helvetica Neue','Source Sans 3',system-ui,sans-serif;
    color:var(--ink);background:#FFFFFF;line-height:1.45;font-size:9.5pt;
  }
  body{padding:0;max-width:none;margin:0}

  header.gnt-header{
    position:running(hdr);
    display:flex;align-items:center;justify-content:space-between;
    padding:0 14mm 4mm;border-bottom:0.5pt solid var(--line);margin-bottom:4mm;
    font-size:8pt;color:var(--muted);
  }
  .gnt-logo{height:9mm;width:auto;max-width:60mm;display:block}
  .gnt-meta{text-align:right;line-height:1.3;font-size:7.5pt}
  .gnt-meta strong{display:block;font-size:8.5pt;color:var(--ink);font-weight:600}

  h1{font-size:22pt;font-weight:700;letter-spacing:-0.01em;margin:0 0 4mm;page-break-after:avoid}
  h2{
    font-size:13pt;font-weight:600;margin:8mm 0 2mm;
    border-bottom:1pt solid var(--accent);padding-bottom:1.5mm;
    page-break-after:avoid;break-after:avoid;
  }
  h3{font-size:10.5pt;font-weight:600;margin:5mm 0 1.5mm;page-break-after:avoid}
  p{font-size:9pt;line-height:1.5;margin:1.5mm 0 2mm;color:#1F2937}
  .subtitle{color:var(--muted);font-size:8.5pt;margin:2mm 0 4mm}
  code{font-family:'JetBrains Mono',ui-monospace,Menlo,monospace;background:#F3F4F6;padding:0.4mm 1.2mm;border-radius:1.5pt;font-size:85%}

  .icon{display:none!important}

  .legend{display:flex;gap:6mm;flex-wrap:wrap;margin:3mm 0 5mm;padding:2mm 3mm;background:#F9FAFB;border:0.5pt solid var(--line);border-radius:2pt;page-break-inside:avoid}
  .legend-item{display:flex;align-items:center;gap:1.5mm;font-size:8pt}
  .dot{width:2mm;height:2mm;border-radius:50%}

  .diagram-card{background:#FFFFFF;border:0.5pt solid var(--line);border-radius:2pt;padding:3mm;margin:2mm 0 4mm;page-break-inside:avoid;break-inside:avoid}
  .diagram-card svg{width:100%;height:auto;display:block;max-height:140mm}

  table{width:100%;border-collapse:collapse;margin:2mm 0 5mm;font-size:7.5pt;page-break-inside:auto}
  th{background:#F3F4F6;color:var(--ink);padding:1.5mm 2mm;text-align:left;font-weight:600;border:0.5pt solid var(--line);font-size:8pt}
  td{padding:1.5mm 2mm;border:0.5pt solid var(--line);vertical-align:top;color:#1F2937}
  tr{page-break-inside:avoid;break-inside:avoid}

  .note{
    background:#FFFBEB;border-left:1.5pt solid #F59E0B;
    padding:2.5mm 3mm;border-radius:1pt;margin:2mm 0 4mm;
    font-size:8.5pt;color:#1F2937;page-break-inside:avoid;break-inside:avoid;
  }
  .note > .icon{display:none!important}

  .meta{font-size:7pt;color:var(--muted);margin-top:6mm;font-style:italic;border-top:0.5pt solid var(--line);padding-top:3mm}
</style>
```

## Icon Stripping (Depth-Counting Regex)

Naive regex `<svg class="icon">.*?</svg>` fails on nested SVGs (Lucide icons wrapped in an outer container SVG). Use this depth-counting approach instead:

**Off-by-one trap:** the needle `'<svg class="icon"'` is 17 characters — slice `text[i:i+18]` never matches and the strip silently removes NOTHING. Always slice `text[i:i+17]`. When icons are single non-nested `<svg>` roots (the common case for inline Lucide icons), a plain `re.sub(r'<svg class="icon"[^>]*>.*?</svg>', '', html, flags=re.DOTALL)` is sufficient and easier to get right; use depth-counting only when nested SVGs are actually present.

```python
def strip_icon_in_heading(text):
    """Remove <svg class="icon"...>...</svg> blocks, handling nested SVGs."""
    out = []
    i = 0
    while i < len(text):
        if text[i:i+17] == '<svg class="icon"':
            depth = 1
            j = text.find('>', i) + 1
            while j < len(text) and depth > 0:
                if text[j:j+6] == '<svg ' or text[j:j+5] == '<svg':
                    depth += 1
                    j = text.find('>', j) + 1
                elif text[j:j+6] == '</svg>':
                    depth -= 1
                    j += 6
                else:
                    j += 1
            i = j  # skip past the balanced </svg>
        else:
            out.append(text[i])
            i += 1
    return ''.join(out)

# Apply within h2/h3 blocks:
import re
def clean_heading(match):
    tag = match.group(1)
    inner = match.group(2)
    cleaned = strip_icon_in_heading(inner)
    return f'<{tag}>{cleaned}</{tag}>'

content = re.sub(r'<(h2|h3)>(.*?)</\1>', clean_heading, content, flags=re.DOTALL)

# Clean orphan </svg> tags left at start of heading text:
content = re.sub(r'(<h[23][^>]*>)\s*</svg>\s*', r'\1', content)
```

## weasyprint Command

```bash
weasyprint input.html output.pdf
```

### Known Warnings
- `Ignored 'display: grid'` — WeasyPrint does NOT support CSS grid at all. Grid containers are silently ignored and children stack vertically. This breaks stats grids, feature card layouts, and comparison grids. **Fix:** convert `display: grid; grid-template-columns: repeat(auto-fit, minmax(Npx, 1fr)); gap: M` to `display: flex; flex-wrap: wrap` + `.child { flex: 1 1 Npx; margin: 0.375rem; }` on each child.
- `Ignored 'gap: M'` — WeasyPrint doesn't support `gap` on flex either. Replace with `margin` on children (half the gap value on each side).
- `Invalid media type '(max-width: 640px)'` — WeasyPrint ignores `@media` queries with feature syntax. Responsive rules won't apply. Either inline the mobile styles or wrap them in `@media screen`.
- Large base64 images may produce memory warnings on very large documents.

### Grid → Flexbox Conversion Pattern

Before running weasyprint, scan the HTML for `display: grid` and convert each instance:

```python
import re

# Pattern: display: grid; grid-template-columns: repeat(auto-fit, minmax(Npx, 1fr)); gap: Mrem;
# Convert to: display: flex; flex-wrap: wrap;
# Then add to children: flex: 1 1 Npx; margin: 0.375rem;

# Example conversion (do via patch tool or execute_code):
# BEFORE:
#   .stats-grid {
#     display: grid;
#     grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
#     gap: 0.75rem;
#     margin-bottom: 1.5rem;
#   }
# AFTER:
#   .stats-grid {
#     display: flex;
#     flex-wrap: wrap;
#     margin-bottom: 1.5rem;
#   }
#   .stats-grid .stat { flex: 1 1 160px; margin: 0.375rem; }
```

Also convert `display: flex; gap: M` patterns — replace `gap` with `margin` on children:
```python
# BEFORE: .header .meta { display: flex; flex-wrap: wrap; gap: 0.75rem; }
# AFTER:  .header .meta { display: flex; flex-wrap: wrap; }
#         .header .meta > * { margin-right: 0.75rem; }
```

### Dark-Theme HTML → PDF (No Print CSS Override)

When converting a dark-themed HTML document (GitHub-style `#0D1117` background) to PDF via weasyprint, do NOT add a `@media print` block that overrides to white backgrounds. WeasyPrint renders dark themes correctly as-is. A print override only changes `body` and `.card` backgrounds but leaves CSS variables (`--text-muted`, `--border`) and colored table cells (`td.green`, `td.warn`, `td.red`) with their dark-theme values — producing invisible text on white. Remove any existing `@media print` block before rendering.

### Running Header Pattern (avoid logo clipping and title collision)

Do NOT put the full screen header (logo `<img>` + metadata flex block) in a `position:running(hdr)` element — WeasyPrint renders the running element inside the page margin box, where a wide logo gets clipped and the block collides with page-1 body content. Instead:

- Keep the logo + metadata header as a normal in-body block at the top of page 1 (it doubles as the document frontmatter).
- Use a **text-only** `@top-center` string running header for pages 2+ (doc ID · version · title · classification), and suppress it on the first page: `@page :first{@top-center{content:none}}`.
- Keep the page counter in `@bottom-center` on all pages.

### Wide-Table Overflow (10+ columns)

Under `table-layout:auto`, a table with many columns can exceed the printable width and silently clip the last columns past the right margin — WeasyPrint will not warn. For any table wider than ~8 columns:

- Set `table-layout:fixed` and add an explicit `<colgroup>` with percentage widths summing to 100%.
- Drop the font size one step (e.g. 7.5pt → 6.8pt) and shorten long cell values and headers ("Meets Expectations" → "Meets Exp.", "Band Min–Max (VND)" → "Band Min–Max").
- Add `word-wrap:break-word; overflow-wrap:break-word` on `td` so long names wrap instead of forcing width.

### Verification

```bash
pdfinfo output.pdf
# Check: Pages count, Page size (should be 595.276 x 841.89 pts = A4), PDF version
```

Name the PDF with the source HTML's document ID and version (`<DocumentID>-v<major>_<minor>_<patch>.pdf`) and save it beside the HTML versions in the project folder.

**Visual verification (two passes, both required):** `pdfinfo` checks page count and size, not layout. After it passes:

1. **Structure pass (contact sheet):** `pdftoppm -png -r 60` all pages, stitch with PIL into a contact sheet (4 columns × ceil(N/4) rows), feed to `vision_analyze` — catches blank pages, missing page counters, gross breakage.
2. **Layout pass (per-page, high-res):** `pdftoppm -png -r 110` the first 2–3 pages, any page holding the widest table, and the final page; feed each to `vision_analyze` asking specifically about overlapping elements, margin overflow, clipped columns, and header/title collision. A 60-dpi contact sheet CANNOT see subtle overlaps or a table column clipped past the right margin — a contact-sheet-only pass has approved PDFs that the user then rejected for layout defects.

```bash
pdftoppm -png -r 110 -f 1 -l 3 output.pdf /tmp/hi   # high-res layout check
```

## Full Pipeline Script

```python
import re, base64

def generate_print_html(source_html_path, output_html_path):
    """Convert a screen-version HTML to print-friendly A4 HTML."""
    with open(source_html_path) as f:
        src = f.read()

    # 1. Replace style block with print CSS (use the template above)
    print_css = open('print_style.css').read()  # or inline the CSS from above
    new = re.sub(r'<style>.*?</style>', f'<style>{print_css}</style>', src, count=1, flags=re.DOTALL)

    # 2. Strip Lucide icons from headings
    def strip_icon_in_heading(text):
        out = []
        i = 0
        while i < len(text):
            if text[i:i+17] == '<svg class="icon"':
                depth = 1
                j = text.find('>', i) + 1
                while j < len(text) and depth > 0:
                    if text[j:j+6] == '<svg ' or text[j:j+5] == '<svg':
                        depth += 1
                        j = text.find('>', j) + 1
                    elif text[j:j+6] == '</svg>':
                        depth -= 1
                        j += 6
                    else:
                        j += 1
                i = j
            else:
                out.append(text[i])
                i += 1
        return ''.join(out)

    def clean_heading(match):
        tag = match.group(1)
        inner = match.group(2)
        return f'<{tag}>{strip_icon_in_heading(inner)}</{tag}>'

    new = re.sub(r'<(h2|h3)>(.*?)</\1>', clean_heading, new, flags=re.DOTALL)
    new = re.sub(r'(<h[23][^>]*>)\s*</svg>\s*', r'\1', new)

    # 3. Update title
    new = new.replace('v1.7.0', 'v1.8.0 (Print)')

    with open(output_html_path, 'w') as f:
        f.write(new)

    return output_html_path

# Usage:
# generate_print_html('doc_v1_7_0.html', 'doc_v1_8_0_print.html')
# Then: weasyprint doc_v1_8_0_print.html doc_v1_8_0.pdf
```
