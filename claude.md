# Incident.io TRMNL Plugin — Development Notes

## Project Overview

TRMNL plugin (ID 175032) displaying Incident.io incidents on an e-ink display. Shows active, critical, and resolved incidents across four layout sizes.

## File Structure (trmnl_preview v0.3.0)

```
incident-io/
├── config.toml              # Local dev data source (points to data.json)
├── data.json                # Local test data (real API response, git-ignored)
├── views/                   # Symlink → src/ (gem expects templates here)
├── src/
│   ├── settings.yml         # Plugin metadata for trmnlp push
│   ├── shared.liquid        # Shared components, demo data, counting logic
│   ├── full.liquid          # Full screen — two-column layout
│   ├── half_horizontal.liquid  # Half horizontal — compact two-column
│   ├── half_vertical.liquid    # Half vertical — single-column list
│   └── quadrant.liquid         # Quadrant — metrics only
├── tmp/                     # Build output (git-ignored)
└── .gitignore
```

**Key differences from older gem versions:**
- `config.toml` replaces `.trmnlp.yml` for local data config
- `views/` symlink required (gem reads templates from `views/`, not `src/`)
- Build output goes to `tmp/`, not `_build/`
- `bin/dev` no longer exists — use `trmnlp serve` directly

## Build & Serve

```bash
# Local development (hot reload)
trmnlp serve

# Build static HTML
trmnlp build          # Output → tmp/

# Deploy to TRMNL
trmnlp login
trmnlp push           # Reads src/settings.yml for plugin metadata
```

**Data flow:**
- `trmnlp serve` reads `config.toml` → loads `data.json` as template context
- `trmnlp build` reads `config.toml` → loads `data.json` → outputs HTML to `tmp/`
- `trmnlp push` reads `src/settings.yml` for plugin ID, API config, custom fields

## Incident.io API

**Field names:**
- `incident_status` (not `status`)
- `incident_status.category` — `live`, `learning`, or `closed`
- `incident_status.name` — human-readable (e.g. "Investigating", "Resolved")
- `severity.name` — `P1` through `P5` (not "Critical", "High", etc.)

**Active = `live` + `learning`** (everything except `closed`).

**Polling config (in settings.yml):**
```yaml
polling_url: https://api.incident.io/v2/incidents?page_size={{ max_incidents }}
polling_headers: Authorization=Bearer {{ api_key }}
```

Headers use `key=value` format with `&` separating multiples — not `key: value`.

## TRMNL Framework v2

Docs: https://trmnl.com/framework/docs

**Categories:**
| Category | Topics |
|---|---|
| Foundation | Structure, Screen, View, Layout, Title Bar, Columns, Mashup |
| Arrangement | Size, Spacing, Gap, Flex, Grid, Aspect Ratio |
| Responsive | Responsive, Visibility |
| Styling | Background, Border, Rounded, Outline, Text, Image, Scale |
| Modulations | Overflow, Table Overflow, Clamp, Format Value, Fit Value, Content Limiter, Pixel Perfect |
| Elements | Title, Value, Label, Description, Divider |
| Components | Rich Text, Item, Table, Chart, Progress |

### Framework Classes Used in This Plugin

**Typography:**
- `text--black` — black text (#000000, max e-ink contrast)
- `label` — standard label (12px)
- `label--small` — small label (10px)
- `value--large` — large value display (metric counts)

**Layout & Spacing:**
- `layout layout--col gap--space-between` — full-height column layout
- `grid grid--cols-1`, `grid--cols-2`, `grid--cols-3` — grid columns
- `gap--small`, `gap--medium` — grid/flex gaps
- `mt--1` through `mt--2.5` — margin top
- `mb--1`, `mb--1.5` — margin bottom
- `pl--1.5` — padding left
- `pb--1.5` — padding bottom
- `p--2.5`, `p--5` — padding all sides
- `flex flex--wrap gap--[4px]` — wrapping flex with custom gap

**Components:**
- `item` — standard card container
- `item--emphasis-2` — medium emphasis (P2 severity)
- `item--emphasis-3` — high emphasis (P1 severity)
- `label--inverted` — inverted label (white on black, used for severity and impact badges)
- `rounded--xsmall` — small border radius
- `rounded--[3px]` — arbitrary border radius
- `content` — item content wrapper
- `title_bar` — bottom title bar with logo and status

**Modulations:**
- `data-clamp="1"` — runtime text clamping (single line, replaces Liquid `truncate`)

## Shared Components (src/shared.liquid)

### Demo Data

When `api_key == "0"` (marketplace preview mode), 4 hardcoded demo incidents are created using `parse_json`. This provides realistic screenshots for the marketplace listing.

### Incident Counting

Inline logic at the top of `shared.liquid` — loops through `all_incidents` and sets:
- `active_count` — incidents where `category != "closed"`
- `closed_count` — incidents where `category == "closed"`
- `critical_count` — active incidents where `severity.name == "P1"`

### metrics_header

```liquid
{% render "metrics_header", active: active_count, critical: critical_count, resolved: closed_count %}
```

3-column grid showing Active, Critical, and Resolved counts. Used by all layouts.

### incident_card

```liquid
{% render "incident_card", incident: incident %}
{% render "incident_card", incident: incident, show_impact: true %}
```

Displays incident name (clamped to 1 line via `data-clamp="1"`), status, severity badge, and created timestamp. Severity drives emphasis: P1 → `item--emphasis-3`, P2 → `item--emphasis-2`, P3+ → plain `item`.

Optional `show_impact: true` renders impact badges (teams, services, users) from `custom_field_entries`.

### severity_badge

```liquid
{% render "severity_badge", severity: incident.severity %}
```

Displays severity with emoji indicator (■ P1, ● P2, ▲ P3, ○ P4+) using `label--inverted`.

### Impact Badges

Three badge components for custom field data:
- `impact_badge_teams` — affected teams
- `impact_badge_services` — affected services
- `impact_badge_users` — user count

All use `label--inverted`. Only render when the matching custom field has values.

### empty_state

```liquid
{% render "empty_state", size: 'large' %}   <!-- full layout -->
{% render "empty_state", size: 'medium' %}  <!-- half_vertical -->
{% render "empty_state", size: 'small' %}   <!-- compact views -->
```

### metric_card

```liquid
{% render "metric_card", value: count, label: 'Active', centered: false %}
```

Individual metric display. Called by `metrics_header`.

### title_bar

```liquid
{% render "title_bar", active: active_count %}
```

Bottom bar with Incident.io logo, plugin name, and dynamic status ("1 Active Incident" / "3 Active Incidents" / "All Clear").

### Duration Templates (legacy)

`calculate_duration` and `format_duration` remain in `shared.liquid` but aren't used. Liquid's `"now"` filter doesn't work reliably in TRMNL's environment, so we display the created timestamp directly.

## Layout Design

| Layout | Columns | Active shown | Closed shown | Empty state size |
|---|---|---|---|---|
| full | 2 | 3 | 3 | large |
| half_horizontal | 2 | 2 | 2 | inline |
| half_vertical | 1 | 4 | 0 | medium |
| quadrant | — | 0 | 0 | — |

All layouts render `metrics_header` at the top and `title_bar` at the bottom.

## Marketplace Review Fixes (March 2026)

Mario's 6 items from TRMNL marketplace review — all implemented:

1. **Demo data** — added `api_key == "0"` block in `shared.liquid` with 4 demo incidents using `parse_json`
2. **Empty state sizes** — `'large'` for `full.liquid`, `'medium'` for `half_vertical.liquid`
3. **`label--outline` → `label--inverted`** — updated on all impact badges
4. **Removed `font-bold`** — stripped from `empty_state` and `metric_card` components
5. **`data-clamp="1"`** — replaces Liquid `truncate` filter in `incident_card` for runtime text clamping
6. **Removed `truncate_length`** — parameter no longer passed to any `incident_card` render calls

## Liquid Limitations

**Nested property filtering doesn't work:**
```liquid
{% assign filtered = items | where: "nested.property", "value" %}  ← BROKEN
```

Use manual loops instead:
```liquid
{% for item in items %}
  {% if item.nested.property == "value" %}
    {% assign count = count | plus: 1 %}
  {% endif %}
{% endfor %}
```

## Common Issues

| Problem | Cause | Fix |
|---|---|---|
| Counts show 0 | `where` filter with nested props | Manual counting loops |
| `Encoding::InvalidByteSequenceError` | Non-ASCII chars in `data.json` | Strip smart quotes, em-dashes |
| Text overlap | Missing spacing after removing inline styles | Framework spacing classes (`mb--1`, `pb--1.5`) |
| Duration calc fails | Liquid `"now"` unreliable in TRMNL | Show `created_at` timestamp instead |
| Marketplace rejection | Inline styles | Framework classes only — zero inline styles |

## E-ink Display Tips

- Black text (`text--black`) for max contrast
- Use framework typography — don't set custom font sizes
- `data-clamp` for text truncation (works at runtime on the device)
- Less clutter = better readability. Cut fields that aren't essential.
- Use `item--emphasis` classes for visual priority, not custom borders
