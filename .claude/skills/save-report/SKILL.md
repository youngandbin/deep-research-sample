---
name: save-report
description: |
  Save a deep research report from an Opik trace. Given a trace ID, extracts the final_report
  from the chart_generation_and_improvement span, saves it as a markdown file in reports/,
  downloads all chart images to images/, converts image URLs to local paths, and updates README.md.
argument-hint: "<trace-id>  Opik trace ID"
---

# Save Deep Research Report from Opik Trace

Extracts a deep research report from an Opik trace, saves it locally with downloaded images,
and updates the README index.

## Inputs

Parse the Opik trace ID from `$ARGUMENTS`.
- If no trace ID is provided, ask the user.

---

## Step 1: Fetch trace data from Opik

Use the Opik MCP tool `get-trace-by-id` to retrieve the trace.

```
get-trace-by-id(trace_id=<TRACE_ID>)
```

### Trace data structure

A trace contains an array of spans. Each span has:
```json
{
  "id": "...",
  "name": "chart_generation_and_improvement",
  "type": "general",
  "input": {...},
  "output": {
    "chart_images": {"hash1": "https://...png?...", ...},
    "final_report": "# Report Title\n\n..."
  }
}
```

**The `final_report` lives in the span named `chart_generation_and_improvement`**, inside
`output.final_report`. It is NOT necessarily the last span in the array — always locate it
by span name.

If `get-trace-by-id` does not return span-level data directly, use other Opik MCP tools
(e.g. `list-traces`) to retrieve the span list for the trace.

The `final_report` value is a markdown string.

---

## Step 2: Determine filename and save the report

Derive the filename from the first `# ` heading in the final_report markdown.

- Strip the `# ` prefix.
- Remove any subtitle after `:` (e.g. `# Analysis Report: Subtitle` → `Analysis Report`).
- Remove characters invalid in filenames (`/`, `\`, `?`, `*`, `<`, `>`, `|`, `"`).
- If the title exceeds 50 characters, shorten it to the key terms.

Save the final_report content to `reports/<filename>.md`.

---

## Step 3: Download images and convert paths

Find all `![...](https://...)` image references in the saved markdown file.

For each image URL:
1. Extract the filename using the `chart_YYYYMMDD_HHMMSS_N.png` pattern.
2. Download the image to `images/<filename>`.
3. Replace the URL in the markdown with `../images/<filename>`.

If the URL does not match the chart pattern, use the last path segment (without query string)
as the filename.

```python
import os, re, urllib.request

with open(report_path, 'r', encoding='utf-8') as f:
    content = f.read()

def replace_img(m):
    alt, url = m.group(1), m.group(2)
    fname_match = re.search(r'(chart_\d+_\d+_\d+\.png)', url)
    if fname_match:
        fname = fname_match.group(1)
    else:
        fname = url.split('/')[-1].split('?')[0]
    local_path = os.path.join('images', fname)
    if not os.path.exists(local_path):
        urllib.request.urlretrieve(url, local_path)
    return f'![{alt}](../images/{fname})'

new_content = re.sub(r'!\[([^\]]*)\]\((https?://[^)]+)\)', replace_img, content)

with open(report_path, 'w', encoding='utf-8') as f:
    f.write(new_content)
```

---

## Step 4: Update README.md

Append a new row to the report table in `README.md`.

- Percent-encode the full filename (including Korean characters) for a working GitHub link.
- For the description column, extract all `## ` headings from the final_report (excluding `## 출처`)
  and list them joined by `, `.

```python
import re, urllib.parse

headings = re.findall(r'^## (.+)', final_report, re.MULTILINE)
headings = [h for h in headings if h.strip() != '출처']
description = ', '.join(headings)

encoded = urllib.parse.quote(filename)
new_row = f'| [{title}](reports/{encoded}) | {description} |'
```

Insert the new row after the last row in the table.

---

## Step 5: Summary

Report back to the user:
- Path of the saved report file
- Number of images downloaded
- Whether README.md was updated

---

## Important

- **`final_report` location**: Found in the `chart_generation_and_improvement` span's `output.final_report`. Always search by span name, not by array index.
- **Image relative paths**: Reports live in `reports/`, so image paths must be `../images/<filename>`.
- **README link encoding**: Use `urllib.parse.quote()` on the full Korean filename so links work on GitHub.
- **File conflict**: If a report with the same name already exists, ask the user before overwriting.
- **Skip existing images**: Do not re-download if the same filename already exists in `images/`.
