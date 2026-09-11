# Mermaid Diagram Size Verification

## When to Use

After rendering any HTML document containing Mermaid diagrams, run this check to verify diagrams are not too small (cramped/unreadable) or too big (clipping/overflowing their container).

## JS Size-Check Snippet

Inject this script at the end of the HTML `<body>` (after the Mermaid bundle + initialize scripts). It runs after Mermaid renders, measures each `.mermaid-wrapper` SVG, and writes a JSON report to a hidden `<div id="size-report">`.

```javascript
setTimeout(function() {
  var results = [];
  var wrappers = document.querySelectorAll('.mermaid-wrapper');
  wrappers.forEach(function(wrapper, i) {
    var svg = wrapper.querySelector('svg');
    if (!svg) { results.push({index: i, status: 'FAIL', reason: 'no SVG rendered'}); return; }
    var svgRect = svg.getBoundingClientRect();
    var wrapperRect = wrapper.getBoundingClientRect();
    var viewBox = svg.getAttribute('viewBox') || '';
    var vbParts = viewBox.split(/\s+/).map(parseFloat);
    var vbWidth = vbParts[2] || 0;
    var vbHeight = vbParts[3] || 0;
    var scaleX = svgRect.width / (vbWidth || 1);
    var scaleY = svgRect.height / (vbHeight || 1);

    var issues = [];
    // Check 1: too small (rendered width < 200px means text likely unreadable)
    if (svgRect.width < 200) issues.push('too_small_width:' + Math.round(svgRect.width) + 'px');
    // Check 2: too small height (rendered height < 80px)
    if (svgRect.height < 80) issues.push('too_small_height:' + Math.round(svgRect.height) + 'px');
    // Check 3: horizontal clipping (SVG wider than wrapper, scroll appears)
    if (wrapper.scrollWidth > wrapper.clientWidth + 1)
      issues.push('horizontal_clip:scroll=' + wrapper.scrollWidth + 'vs_client=' + wrapper.clientWidth);
    // Check 4: SVG overflows wrapper visually
    if (svgRect.width > wrapperRect.width + 1)
      issues.push('svg_overflows_wrapper:' + Math.round(svgRect.width) + 'vs' + Math.round(wrapperRect.width));
    // Check 5: extreme aspect ratio (width:height > 10:1 or height:width > 10:1)
    var ratio = svgRect.width / (svgRect.height || 1);
    if (ratio > 10) issues.push('extreme_wide_ratio:' + ratio.toFixed(1));
    if (ratio < 0.1) issues.push('extreme_tall_ratio:' + (1/ratio).toFixed(1));
    // Check 6: scale factor too small (viewBox much larger than rendered = shrunk too much)
    if (scaleX < 0.3 && vbWidth > 500) issues.push('over_shrunk:scale=' + scaleX.toFixed(2) + 'vbW=' + Math.round(vbWidth));

    results.push({
      index: i,
      status: issues.length === 0 ? 'PASS' : 'FAIL',
      issues: issues,
      renderedW: Math.round(svgRect.width),
      renderedH: Math.round(svgRect.height),
      viewBoxW: Math.round(vbWidth),
      viewBoxH: Math.round(vbHeight),
      scaleX: +scaleX.toFixed(2),
      scaleY: +scaleY.toFixed(2),
      wrapperW: wrapper.clientWidth,
      scrollW: wrapper.scrollWidth
    });
  });
  document.getElementById('size-report').textContent = JSON.stringify(results, null, 2);
}, 4000);
```

## How to Run

1. Temporarily inject the size-check script + `<div id="size-report" style="display:none;"></div>` into the HTML
2. Render in headless Chromium: `chromium-browser --headless=new --no-sandbox --disable-gpu --dump-dom --virtual-time-budget=12000 --window-size=1280,900 file://<path>`
3. Extract the JSON from the `#size-report` div in the output DOM (strip `<script>` blocks first to avoid Mermaid bundle false positives)
4. Verify every diagram has `status: "PASS"`; if any `FAIL`, fix the diagram and re-render

## Failure Thresholds

| Check | Threshold | Reason |
|---|---|---|
| `too_small_width` | rendered width < 200px | Text in nodes becomes unreadable |
| `too_small_height` | rendered height < 80px | Diagram is cramped flat |
| `horizontal_clip` | wrapper.scrollWidth > clientWidth | Diagram overflows container horizontally |
| `svg_overflows_wrapper` | SVG width > wrapper width | SVG spills outside its container |
| `extreme_wide_ratio` | width:height > 10:1 | Diagram is unnaturally stretched wide |
| `extreme_tall_ratio` | height:width > 10:1 | Diagram is unnaturally stretched tall |
| `over_shrunk` | scale < 0.3 AND viewBox > 500px | Large viewBox squeezed too small |

## Fixes for Common Failures

- **Too small / over-shrunk**: Remove `useMaxWidth: true` or set explicit `max-width` on the wrapper to prevent excessive shrinking. Alternatively, break a wide horizontal flowchart into a 2D grid layout (use subgraphs or `flowchart TD` with branches).
- **Horizontal clipping**: Ensure `.mermaid-wrapper` has `overflow-x: auto` so the diagram scrolls rather than clips. If the diagram is genuinely too wide, restructure it (e.g., split into multiple smaller diagrams).
- **Extreme aspect ratio**: Redesign the diagram layout — wide chains should wrap into rows; tall chains should use `flowchart LR` with branches or group levels into subgraphs.
- **Too narrow (tall diagrams)**: Add `min-width` to the wrapper or adjust the SVG `width` attribute after render via JS.
