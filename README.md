# Incident.io Active Incidents - TRMNL Recipe

A TRMNL recipe that displays real-time incident management data from Incident.io on your TRMNL e-ink display.

## Features

- **Active Incident Count**: See how many incidents are currently open
- **Severity Breakdown**: Critical, High, Medium, and Low incident counts
- **Recent Incidents List**: View the most recent incidents with details
- **Color-Coded Severity**: Visual indicators for incident priority
- **Incident Duration**: See how long each incident has been active
- **Team Assignments**: View who's assigned to each incident
- **Four Layout Sizes**: Full, Half Horizontal, Half Vertical, and Quadrant

## Setup

### 1. Get Your Incident.io API Key

1. Log into your Incident.io dashboard
2. Go to **Settings** > **API Keys**
3. Create a new API key with read permissions
4. Copy the API key (starts with `inc_...`)

### 2. Install the Recipe

1. Upload this recipe to TRMNL as a public recipe
2. Users can install it from the TRMNL recipe library
3. During setup, users will be prompted to enter:
   - **Incident.io API Key**: Their API key
   - **Max Incidents to Display**: Number of incidents to show (default: 5)

### 3. Configure Refresh Interval

The recipe polls the Incident.io API every 15 minutes by default. You can adjust this in `src/settings.yml`:
- 15 minutes (default)
- 60 minutes
- 360 minutes (6 hours)
- 720 minutes (12 hours)
- 1440 minutes (24 hours)

## Local Development

### Prerequisites

You need either:
- Ruby with `trmnl_preview` gem: `gem install trmnl_preview`
- OR Docker: [Install Docker](https://docs.docker.com/get-docker/)

### Running Locally

1. Clone or download this repository and navigate to the recipe directory:
   ```bash
   cd incident-io
   ```

2. Copy the example config file and add your API key:
   ```bash
   cp .trmnlp.yml.example .trmnlp.yml
   # Edit .trmnlp.yml and add your API key
   ```

3. Start the development server:
   ```bash
   ./bin/dev
   ```

4. Open your browser to `http://localhost:4567`

5. The server will watch for changes and auto-rebuild templates

### Testing with Static Data

You can test layouts without making API calls by adding a `static_data` section to your `.trmnlp.yml` file. See `.trmnlp.yml.example` for the structure.

### Customizing Templates

Edit the `.liquid` files in `src/`:
- `full.liquid` - Full screen layout (800x480px)
- `half_horizontal.liquid` - Half screen horizontal
- `half_vertical.liquid` - Half screen vertical
- `quadrant.liquid` - Quarter screen

Changes are automatically rebuilt when you save files (if dev server is running).

## File Structure

```
incident-io/
├── .gitignore               # Git ignore file
├── .trmnlp.yml.example      # Example local dev config
├── README.md                # This file
├── claude.md                # Development notes and learnings
├── bin/
│   └── dev                  # Development server script
├── src/
│   ├── settings.yml         # Recipe configuration
│   ├── full.liquid          # Full screen template
│   ├── half_horizontal.liquid
│   ├── half_vertical.liquid
│   └── quadrant.liquid
├── _build/                  # Generated HTML (git ignored)
│   ├── full.html
│   ├── half_horizontal.html
│   ├── half_vertical.html
│   └── quadrant.html
└── example_data.json        # Sample API response

Note: .trmnlp.yml is not committed (contains API keys)
```

## API Details

- **Endpoint**: `https://api.incident.io/v2/incidents`
- **Method**: GET
- **Authentication**: Bearer token
- **Rate Limit**: 1200 requests/minute (far exceeds needs)

## Design

The plugin is optimized for e-ink displays with maximum contrast:
- **Text**: Black (#000000) on white background
- **Bold fonts** for better readability on e-ink
- **Severity indicators**: P1 (■), P2 (●), P3 (▲), P4/P5 (○)
- **Impact badges**: Shows affected teams, services, and user counts

## Publishing

To publish as a public recipe:

1. Test thoroughly with local development
2. Ensure all templates render correctly
3. Upload to TRMNL platform
4. Recipe ID will be assigned by TRMNL (update `id` in `settings.yml`)
5. Share with the community!

## Support

For issues or questions:
- TRMNL Documentation: https://help.usetrmnl.com/
- Incident.io API Docs: https://api-docs.incident.io/

## Future Enhancements

Potential additions for future versions:
- On-call schedule display (requires separate recipe or proxy server)
- Alert metrics
- Follow-up task tracking
- Incident trends/charts
- Custom filters for incident types

## License

Created for the TRMNL community. Modify and share freely!
