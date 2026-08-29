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

```python
def strip_icon_in_heading(text):
    """Remove <svg class="icon"...>...</svg> blocks, handling nested SVGs."""
    out = []
    i = 0
    while i < len(text):
        if text[i:i+18] == '<svg class="icon"':
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

### Known Warnings (Non-Fatal)
- `Ignored 'gap:6mm' at 61:24, unknown property` — weasyprint doesn't support CSS `gap` on flex. Replace with `margin` on children. Layout still renders, just without flex gap spacing.
- Large base64 images may produce memory warnings on very large documents.

### Verification

```bash
pdfinfo output.pdf
# Check: Pages count, Page size (should be 595.276 x 841.89 pts = A4), PDF version
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
            if text[i:i+18] == '<svg class="icon"':
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
