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

The plugin has 5 layouts:
- **full.liquid** - Full screen with detailed incident list
- **half_horizontal.liquid** - Half screen, horizontal orientation
- **half_vertical.liquid** - Half screen, vertical orientation
- **quadrant.liquid** - Minimal quadrant view
- **shared.liquid** - Shared components should be kept here

All layouts should use the same data structure and filtering logic.

Changes within one layout should be reflected in another and centralised within shared.liquid if possible.

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

The plugin was refactored to use TRMNL's official design framework to meet marketplace requirements:
- **100% inline style elimination** - Zero inline styles in user content (only framework-required body style remains)
- **62% code reduction** through shared component extraction (from 441 lines to 165 lines in layout files)
- **Better e-ink optimization** using framework's dithering patterns and spacing
- **Improved maintainability** with single source of truth for common patterns

**Key Achievements:**
- Passed TRMNL marketplace validation (previously rejected for "too many inline styles")
- Created reusable `incident_card` and `metrics_header` components
- Standardized all 4 layouts to use consistent structure and components

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
- `mt--1`, `mt--1.5`, `mt--2`, `mt--2.5` - Margin top (4px to 10px)
- `mb--1`, `mb--1.5` - Margin bottom (4px to 6px)
- `pl--1.5`, `pl--2` - Padding left (6px to 8px)
- `pb--1.5`, `pb--2` - Padding bottom (6px to 8px)
- `p--2.5`, `p--5` - Padding all sides (10px to 20px)

**Item Emphasis Classes:**
- `item--emphasis-3` - Highest emphasis (for P1 incidents)
- `item--emphasis-2` - Medium emphasis (for P2 incidents)
- `item` - Standard item (for P3-P5 incidents)

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

**1. Incident Counting (inline logic)**
- Calculates active_count, critical_count, closed_count at the top of shared.liquid
- Variables available globally to all layouts
- Eliminates 10-13 lines of duplicate code per file

**2. Metrics Header (`metrics_header`)**
- **Usage:** `{% render "metrics_header", active: active_count, critical: critical_count, resolved: closed_count %}`
- Always displays 3 columns: Active, Critical, Resolved
- Uses framework grid classes: `grid grid--cols-3 gap--medium`
- Consistent across all 4 layouts (full, half_horizontal, half_vertical, quadrant)
- Single source of truth for metrics display

**3. Incident Card (`incident_card`)**
- **Usage:** `{% render "incident_card", incident: incident, truncate_length: 35 %}`
- Displays: incident name, status, severity badge, and created timestamp
- Handles severity emphasis automatically (P1 → item--emphasis-3, P2 → item--emphasis-2)
- Configurable truncate length (35 for full/half_vertical, 40 for half_horizontal)
- Used by all layouts that display incident lists
- Eliminates 20+ lines of duplicate code per incident

**4. Severity Badge (`severity_badge`)**
- **Usage:** `{% render "severity_badge", severity: incident.severity %}`
- Displays severity with framework classes: `label label--inverted rounded--xsmall ml--1`
- Shows emoji indicators: ■ (P1), ● (P2), ▲ (P3), ○ (P4/P5)
- No longer accepts size parameter (uses framework defaults)

**5. Impact Badges (legacy - not currently used)**
- `impact_badge_teams` - Shows affected teams with 👥 emoji
- `impact_badge_services` - Shows affected services with ⚙️ emoji
- `impact_badge_users` - Shows user count with 👤 emoji
- **Note:** Removed from all layouts to reduce clutter on e-ink display
- Components remain in shared.liquid for future use if needed

**6. Empty State (`empty_state`)**
- **Usage:** `{% render "empty_state", size: 'medium' %}`
- Displays "All Clear" or "No Active Incidents" message
- Accepts size parameter: 'large', 'medium', 'small'
- Uses framework classes for all styling

**7. Metric Card (`metric_card`)**
- **Usage:** `{% render "metric_card", value: active_count, label: 'Active', centered: false %}`
- Displays value and label for individual metrics
- Accepts value, label, and centered (boolean) parameters
- Uses framework classes: `value--large text--black font-bold` and `label text--black font-bold`
- Called by metrics_header component

**8. Duration Calculation (legacy - not currently used)**
- `calculate_duration` and `format_duration` templates remain in shared.liquid
- **Note:** Replaced with direct timestamp display (`{{ incident.created_at | date: "%b %d %H:%M" }}`)
- Reason: Liquid's "now" date filter doesn't work reliably in TRMNL's environment
- Showing created timestamp (e.g., "Oct 30 09:35") is more reliable and equally useful

### Layout Design Patterns

**Two-Column Layout** (full.liquid, half_horizontal.liquid):
- Left column: Active incidents (3 for full, 2 for half_horizontal)
- Right column: Closed incidents (3 for full, 2 for half_horizontal)
- Uses `grid grid--cols-2 gap--medium` for consistent spacing

**Single-Column Layout** (half_vertical.liquid):
- Shows active incidents only (up to 4)
- More vertical space for incident details

**Metrics-Only Layout** (quadrant.liquid):
- Shows only the 3-column metrics header
- No incident list

**Severity Visual Indicators:**
- Replaced dynamic border widths with framework `item--emphasis` classes
- P1 incidents: `item--emphasis-3` (highest visual priority)
- P2 incidents: `item--emphasis-2` (medium visual priority)
- P3-P5 incidents: Standard `item` class
- This approach uses framework's built-in emphasis patterns instead of custom inline styles

**Spacing Strategy:**
- Removed all `line-height` inline styles
- Use framework spacing classes: `mb--1`, `mb--1.5`, `mt--1`, `mt--2`, etc.
- Added `pb--1.5` to incident cards for better vertical separation
- Accept framework's default line-height for better compliance

### Code Reduction Summary

**Before Refactoring (Initial Version):**
- quadrant.liquid: 54 lines
- half_vertical.liquid: 122 lines
- half_horizontal.liquid: 122 lines
- full.liquid: 143 lines
- **Total: 441 lines**
- **Inline styles:** 28+ per layout

**After Full Refactoring (Current Version):**
- quadrant.liquid: 17 lines (69% reduction)
- half_vertical.liquid: 31 lines (75% reduction)
- half_horizontal.liquid: 54 lines (56% reduction)
- full.liquid: 63 lines (56% reduction)
- **Total: 165 lines (62% overall reduction)**
- **Plus:** shared.liquid with 275 lines of reusable components
- **Inline styles:** 0 (100% elimination - marketplace compliant)

**Key Improvements:**
1. Created `metrics_header` component - eliminated 4-8 lines per layout
2. Created `incident_card` component - eliminated 15-20 lines per incident
3. All layouts use same incident counting logic from shared.liquid
4. Consistent 3-column metrics across all layouts
5. Zero inline styles - uses only framework classes

The refactoring not only reduced code but made it fully TRMNL marketplace compliant.

### 9. Marketplace Compliance Learnings (November 2025)

**Problem:** Plugin rejected with "Markup uses too many inline styles, add more native Framework classes"

**Solution Process:**
1. **Identified inline styles:** Found 28+ style attributes in full.html (line-height, font-size, padding, border-left, background)
2. **Removed badge sizing:** Eliminated font-size and padding from all badge components (7 styles removed)
3. **Removed line-height:** Accepted framework defaults for line spacing (15 styles removed)
4. **Replaced dynamic borders:** Changed border-left to item--emphasis classes (5 styles removed)
5. **Added framework spacing:** Used pb--1.5, mb--1.5, mt--1, mt--2 for vertical spacing
6. **Final result:** Zero inline styles except framework-required body tag

**Key Takeaways:**
- TRMNL marketplace strictly enforces framework-first design
- Framework spacing classes (mb--, mt--, pb--, pl--) are sufficient for most layouts
- Item emphasis classes (item--emphasis-2, item--emphasis-3) replace dynamic border styling
- Accept framework defaults when possible (line-height, font-size) rather than customizing
- Component extraction helps identify and eliminate repeated inline styles

**Display Issues Encountered:**
1. **Text overlap:** Without line-height styles, text overlapped vertically
   - Solution: Added framework spacing classes (pb--1.5, mb--1, mt--1)
2. **Duration calculation failed:** Liquid's "now" filter didn't work in TRMNL
   - Solution: Display created timestamp instead (`{{ incident.created_at | date: "%b %d %H:%M" }}`)
3. **Impact badges too small:** Original 9-10px sizing was hard to read
   - Solution: Removed custom sizing, let framework handle it (cleaner appearance)

**Current Layout Structure:**
- All layouts show 3 metrics: Active, Critical, Resolved
- full.liquid & half_horizontal.liquid: Two-column layout (active left, closed right)
- half_vertical.liquid: Single-column layout (active only)
- quadrant.liquid: Metrics only (no incidents)
- All use shared components for consistency
