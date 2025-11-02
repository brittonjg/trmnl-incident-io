# Claude Development Notes - Incident.io TRMNL Plugin

## Project Overview
This is a TRMNL plugin that displays Incident.io incidents on an e-ink display. The plugin shows active incidents, critical incidents, and resolved incidents with various layout options.

## Key Technical Learnings

### 1. Data Structure Differences

**Important:** The Incident.io API uses different field names than what might be expected:

- ✅ Use `incident_status` NOT `status`
- ✅ Use `incident_status.category` NOT `status.category`
- ✅ Use `incident_status.name` NOT `status.name`

**Severity Naming:**
- The API uses **P1, P2, P3, P4, P5** naming (not "Critical", "High", "Medium", "Low")
- P1 = Highest priority/Critical
- P2 = High
- P3 = Medium
- P4/P5 = Lower priority

**Status Categories:**
- `live` - Active incidents being worked on
- `learning` - Post-incident learning/documentation phase (still considered "active")
- `closed` - Resolved incidents

### 2. Liquid Template Limitations

**Critical Issue:** Liquid's `where` filter does NOT work with nested object properties.

❌ **Does NOT work:**
```liquid
{% assign live_incidents = incidents | where: "incident_status.category", "live" %}
```

✅ **Works:**
```liquid
{% assign active_count = 0 %}
{% for incident in incidents %}
  {% if incident.incident_status.category != "closed" %}
    {% assign active_count = active_count | plus: 1 %}
  {% endif %}
{% endfor %}
```

**Solution:** Use manual counting loops instead of `where` filters when accessing nested properties.

### 3. Build vs Serve Data Sources

- **`trmnlp serve`** (dev server) - Uses `.trmnlp.yml` for local testing
- **`trmnlp build`** - Uses `src/settings.yml` for building static HTML

Both files need to have matching data structures!

### 4. Display Optimization for E-ink

**Best Practices Learned:**
- Use **black (#000000)** for maximum contrast on e-ink displays
- Make text **bold** for better readability
- Font sizes that work well:
  - Main incident names: 16px
  - Meta information: 12px
  - Small labels: 10-11px
- Use `line-height: 1.3-1.5` to prevent text cutoff
- Allow text to wrap lengthways rather than truncating with `white-space: nowrap`
- Remove unnecessary fields (reference numbers, assignee names) to reduce clutter

### 5. Layout Structure

The plugin has 4 layouts:
- **full.liquid** - Full screen with detailed incident list
- **half_horizontal.liquid** - Half screen, horizontal orientation
- **half_vertical.liquid** - Half screen, vertical orientation
- **quadrant.liquid** - Minimal quadrant view

All layouts should use the same data structure and filtering logic.

### 6. Production Checklist

Before uploading to TRMNL:

1. ✅ Remove `strategy: static` from `src/settings.yml`
2. ✅ Ensure `src/settings.yml` and `.trmnlp.yml` have matching data structures
3. ✅ Update all layouts with correct field names (`incident_status` not `status`)
4. ✅ Use manual counting loops for nested property filtering
5. ✅ Run `trmnlp build` to generate production HTML files
6. ✅ Test with real API data structure (P1-P5 severities)
7. ✅ Verify counts show correctly (Active, Critical, Resolved)

### 7. Common Issues & Solutions

**Problem:** Counts showing as 0
- **Cause:** Using `where` filter with nested properties
- **Solution:** Use manual for loops with if statements

**Problem:** Text getting cut off at bottom
- **Cause:** Font too large or no line-height
- **Solution:** Reduce font size to 10-12px, add `line-height: 1.4`

**Problem:** Incident names overlapping
- **Cause:** Too many elements (reference, emoji, name) on one line
- **Solution:** Simplify to just incident name, remove reference numbers

**Problem:** Data not updating after changes
- **Cause:** Forgot to run `trmnlp build`
- **Solution:** Always rebuild after template changes

## File Structure

```
incident-io/
├── .trmnlp.yml              # Local dev configuration & test data
├── src/
│   ├── settings.yml         # Production settings & static data
│   ├── full.liquid          # Full screen layout template
│   ├── half_horizontal.liquid
│   ├── half_vertical.liquid
│   ├── quadrant.liquid
│   └── shared.liquid        # Shared components (currently empty)
├── _build/                  # Generated HTML files (created by trmnlp build)
│   ├── full.html
│   ├── half_horizontal.html
│   ├── half_vertical.html
│   └── quadrant.html
└── bin/
    └── dev                  # Dev server startup script
```

## API Configuration

**Polling URL:**
```
https://api.incident.io/v2/incidents?page_size={{ max_incidents }}
```

The `page_size` parameter limits how many incidents are returned from the API. This should match the `max_incidents` custom field value.

**Authorization:**
```
Authorization=Bearer {{ api_key }}
```

**IMPORTANT:** TRMNL uses `key=value` format for headers, NOT `key: value`!
- Use `=` to separate key and value (not `:`)
- Use `&` to separate multiple headers
- Example: `Authorization=Bearer {{ api_key }}&Content-Type=application/json`

The `api_key` custom field is interpolated into the Bearer token.

## Deployment

To push to TRMNL server:
```bash
trmnlp push
```

Make sure you're logged in first:
```bash
trmnlp login
```

## Active Incident Calculation

Active incidents = `live` + `learning` status categories (excludes `closed`)

Example from test data:
- 2 incidents with `category: "live"`
- 1 incident with `category: "learning"`
- 21 incidents with `category: "closed"`
- **Active count = 3**
