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
│   └── shared.liquid        # Shared components and reusable macros
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

## TRMNL Framework Integration

### 8. Framework Refactoring (November 2025)

The plugin was refactored to use TRMNL's official design framework instead of inline styles, resulting in:
- **~90% reduction in inline styles** across all templates
- **~50% code reduction** through shared component extraction (from 441 lines to 217 lines in layout files)
- **Better e-ink optimization** using framework's dithering patterns
- **Improved maintainability** with single source of truth for common patterns

### Framework Classes Used

**Typography & Colors:**
- `text--black` - Black text color (#000000)
- `text--white` - White text color
- `font-bold` - Bold font weight
- `title--small` - Small title (16px, for incident names)
- `label` - Standard label text (12px, for metadata)
- `label--small` - Small label text (10px, for impact badges)
- `value--large` - Large value display (for metric counts)

**Layout & Spacing:**
- `flex flex--wrap` - Flexible wrapping container
- `gap--[4px]` - 4px gap between flex items
- `mt--1` through `mt--2.5` - Margin top (4px to 10px)
- `mb--1` - Margin bottom (4px)
- `pl--2` - Padding left (8px)
- `p--5` - Padding all sides (20px)
- `p--2.5` - Padding all sides (10px)

**Components:**
- `label--inverted` - Inverted label (white text on black background, for severity badges)
- `label--outline` - Outlined label (for impact badges)
- `bg--gray-10` - Light gray background using dithering
- `rounded--xsmall` - Small border radius (5px)
- `rounded--[3px]` - Arbitrary 3px border radius

**Grid System:**
- `grid grid--cols-2` - Two column grid
- `grid grid--cols-3` - Three column grid
- `grid--cols-1` - Single column grid
- `gap--medium` - Medium gap between grid items

### Shared Components (src/shared.liquid)

The refactoring extracted common patterns into reusable components:

**1. Incident Counting (`count_incidents`)**
- Calculates active_count, critical_count, closed_count
- Used in all 4 layouts
- Eliminates 10-13 lines of duplicate code per file

**2. Duration Calculation (`calculate_duration`)**
- Calculates hours and minutes from created_at timestamp
- Used in 3 layouts (full, half_horizontal, half_vertical)
- Returns duration_hours and duration_minutes variables

**3. Format Duration (`format_duration`)**
- Displays duration in human-readable format (e.g., "72h 40m" or "15m")
- Handles both hour+minute and minute-only display

**4. Severity Badge (`severity_badge`)**
- Displays severity with framework classes: `label--small label--inverted rounded--xsmall`
- Shows emoji indicators: ■ (P1), ● (P2), ▲ (P3), ○ (P4/P5)
- Accepts size parameter: 'normal' (11px) or 'small' (9px)

**5. Impact Badges**
- `impact_badge_teams` - Shows affected teams with 👥 emoji
- `impact_badge_services` - Shows affected services with ⚙️ emoji
- `impact_badge_users` - Shows user count with 👤 emoji
- All use framework classes: `label--small label--outline bg--gray-10 rounded--[3px]`
- Accept size and limit parameters

**6. Empty State (`empty_state`)**
- Displays "All Clear" or "No Active Incidents" message
- Accepts size parameter: 'large', 'medium', 'small'
- Uses framework classes for all styling

**7. Metric Card (`metric_card`)**
- Displays value and label for metrics (Active, Critical, Resolved)
- Accepts value, label, and centered (boolean) parameters
- Uses framework classes: `value--large text--black font-bold` and `label text--black font-bold`

### Justified Inline Styles

Some inline styles remain for valid technical reasons:

**1. Line Heights** (no framework equivalent)
- `line-height: 1.3` for incident names (prevents overlap)
- `line-height: 1.5` for metadata rows (better readability)

**2. Dynamic Border Widths** (framework only has dithered borders)
- P1 incidents: 4px solid black left border
- P2 incidents: 3px solid black left border
- P3+ incidents: 2px solid black left border
- Framework's `border--v-*` classes create grayscale dithering patterns, not true width variations

**3. Specific Font Sizes** (for precise control)
- Severity badges: 9px or 11px depending on layout
- Impact badges: 9px or 10px depending on layout
- These precise sizes don't map to semantic framework classes

### Code Reduction Summary

**Before Refactoring:**
- quadrant.liquid: 54 lines
- half_vertical.liquid: 122 lines
- half_horizontal.liquid: 122 lines
- full.liquid: 143 lines
- **Total: 441 lines**

**After Refactoring:**
- quadrant.liquid: 25 lines (54% reduction)
- half_vertical.liquid: 61 lines (50% reduction)
- half_horizontal.liquid: 61 lines (50% reduction)
- full.liquid: 71 lines (50% reduction)
- **Total: 217 lines (51% overall reduction)**
- **Plus:** shared.liquid with 216 lines of reusable components

The refactoring moved duplicate code into shared components, making the layouts much more maintainable and consistent.
