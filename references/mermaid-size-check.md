# Mermaid Diagram Size Verification

## When to Use

After rendering any HTML document containing Mermaid diagrams, run this check to verify diagrams are not too small (cramped/unreadable) or too big (clipping/overflowing their container).

## The Root Cause of Tiny Diagrams

When `.mermaid-wrapper` uses `display: flex`, the child `<pre class="mermaid">` has no intrinsic width and shrinks to content size (~300px). The SVG's `width: 100%` then resolves against the shrunken pre, producing tiny diagrams despite `useMaxWidth: true` being set correctly in the Mermaid config.

**The fix is CSS, not Mermaid config:** set `width: 100% !important` on `.mermaid-wrapper .mermaid`. Do NOT remove `useMaxWidth: true` to compensate — that causes overflow. The correct combination is:

```css
.mermaid-wrapper .mermaid { font-size: 15px !important; width: 100% !important; }
.mermaid-wrapper svg { max-width: 100% !important; height: auto !important; }
```

```javascript
mermaid.initialize({
  startOnLoad: true,
  theme: "default",
  securityLevel: "loose",
  flowchart: { useMaxWidth: true, htmlLabels: true, curve: "basis", nodeSpacing: 50, rankSpacing: 60 }
});
```

## CDP-Based Measurement (Reliable on Constrained VMs)

When Playwright and `browser_exec` fail with OOM on the 3.3 MB inline Mermaid bundle, use direct CDP via Python websockets:

1. Launch Chromium with remote debugging:
   ```python
   subprocess.Popen(['chromium-browser', '--headless', '--no-sandbox', '--disable-gpu',
     '--disable-dev-shm-usage', '--remote-debugging-port=9222', 'about:blank'])
   time.sleep(3)
   ```
2. Get the page websocket URL from `http://localhost:9222/json` (use `localhost`, not the IP — secret redaction mangles bracketed IP literals in urllib).
3. Connect via `websockets.connect(page_ws)`, send `Page.navigate` to the file URL, wait 8–10s for Mermaid render.
4. Call `Runtime.evaluate` with a JS snippet that reads `getBoundingClientRect()` on each SVG and its parent `<pre>` and card container.
5. Compare: SVG width should be ≤ card width (no overflow) and ≥ 80% of card width (not undersized).

## JS Measurement Snippet

```javascript
(() => {
  const svgs = document.querySelectorAll('.mermaid-wrapper svg');
  const results = [];
  svgs.forEach((svg, i) => {
    const rect = svg.getBoundingClientRect();
    const pre = svg.parentElement;
    const preRect = pre.getBoundingClientRect();
    const cardRect = pre.parentElement.parentElement.getBoundingClientRect();
    results.push({
      index: i,
      svgWidth: Math.round(rect.width),
      svgHeight: Math.round(rect.height),
      preWidth: Math.round(preRect.width),
      cardWidth: Math.round(cardRect.width),
      bodyClientWidth: document.body.clientWidth,
      dataProcessed: pre.getAttribute('data-processed'),
      viewBox: svg.getAttribute('viewBox')
    });
  });
  return JSON.stringify(results);
})()
```

## Pass/Fail Criteria

| Check | Pass Condition | Reason |
|---|---|---|
| Diagram rendered | `data-processed === "true"` | Mermaid processed the diagram |
| No overflow | `svgWidth <= cardWidth` | SVG stays within container |
| Not undersized | `svgWidth >= cardWidth * 0.8` | SVG fills most of the container |
| Pre fills container | `preWidth >= cardWidth - 50` | The `<pre>` element is not shrunken |

## Fixes for Common Failures

- **SVG too small (300px in a 700px card)**: Add `width: 100% !important` to `.mermaid-wrapper .mermaid`. The `<pre>` flex child is shrinking to content width. This is the most common failure.
- **SVG overflows container**: Ensure `useMaxWidth: true` in Mermaid config AND `max-width: 100% !important` on `.mermaid-wrapper svg`. Do NOT set `max-width: none`.
- **Horizontal clipping**: Ensure `.mermaid-wrapper` has `overflow-x: auto`. If the diagram is genuinely too wide, restructure it (split into multiple smaller diagrams, change `flowchart LR` to `flowchart TD`).
- **Extreme aspect ratio (width:height > 10:1)**: Redesign the diagram layout — wide chains should wrap into rows; use subgraphs or change flow direction.
