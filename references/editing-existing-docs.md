# Editing Existing HTML Docs — Session Notes (2026-08-20)

## Corruption pitfall: read_file round-trip
When removing sections from an existing HTML doc, the agent read the file with the read_file tool and wrote its returned `content` back to disk. read_file output carries line-number prefixes (`N|content`), so the saved file was corrupted with literal `N|` prefixes on every line — user reported "the layout of the html is wrong".

**Rule:** never write read_file's returned content back to disk. Edit via:
- the `patch` tool (find/replace), or
- a Python script using plain `open()` read → modify → write.

**Recovery recipe** (verified working):
```python
import re
p = '/root/<file>.html'
lines = open(p, encoding='utf-8').read().split('\n')
clean = [re.sub(r'^(\d+\|){1,2}', '', l) for l in lines]
open(p, 'w', encoding='utf-8').write('\n'.join(clean))
```
Then verify: first line is `<!DOCTYPE html>`, no `\n\d+\|` regex hits, tag balance (`<h2>`/`</h2>` etc. counts equal), removed sections absent.

Note: hermes_tools `read_file` may return a metadata dict without a `content` key in sandboxed execute_code contexts — another reason to prefer plain `open()` there.

## Scope rule for removal requests
David's "remove X" means literal scope only: delete exactly the named sections including their tables and callouts; no renumbering, restructuring, or version bump unless asked. (Example: removing "Compensation & Location", "What Success Looks Like", "Hiring Process" left a clean Nice to Have → Version History flow.)
