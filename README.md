# trmnl-plugin-wienerlinien

**Abfahrtsmonitor** — a [TRMNL](https://usetrmnl.com) plugin that shows live Wiener Linien public-transport departures on your e-ink display.

Configure up to four stops by their RBL/Steig number and the device shows the next three departures per line as time of day (e.g. `23:42 · 23:49 · 23:56`). Zero backend required: TRMNL polls the [Wiener Linien open-data realtime endpoint](https://www.wienerlinien.at/open-data) directly and renders the four screen layouts via Liquid templates.

> **Unofficial plugin** — not affiliated with, endorsed by, or connected to Wiener Linien GmbH & Co KG. The Wiener Linien name and logo belong to their owners.
>
> **No data is stored or transmitted to third parties.** TRMNL polls the public Wiener Linien open-data endpoint directly. Nothing is sent anywhere else.

## What it shows

Per configured stop (up to 4):

- **Line** (e.g. `U2`, `82A`, `1`)
- **Direction** (e.g. `→ Karlsplatz`)
- **Stop name** (resolved automatically from the RBL)
- **Next 3 departures** as time of day (`HH:mm`), using realtime data when available and falling back to the schedule otherwise

Time of day is used rather than a countdown because the e-ink device only refreshes every 15 minutes — a "2 min" countdown would already be stale by the next refresh.

## How it works

```
┌────────────┐    polls    ┌──────────────────────────┐   renders    ┌──────────────┐
│   TRMNL    │ ──────────► │  Wiener Linien OGD       │ ───────────► │  Liquid      │
│  (polling) │             │  /ogd_realtime/monitor   │              │  templates   │
└────────────┘             └──────────────────────────┘              └──────┬───────┘
                                                                            │
                                                                      ┌─────▼──────┐
                                                                      │  e-ink BMP │
                                                                      │  on device │
                                                                      └────────────┘
```

A single GET to `https://www.wienerlinien.at/ogd_realtime/monitor?rbl=…&rbl=…` returns all configured stops in one shot. The templates iterate `data.monitors[]` and render whatever the API returned — empty slots in the config are silently skipped.

## Installation

*To be filled in once the plugin is published to the [TRMNL marketplace](https://usetrmnl.com/plugins).*

## Configuration

| Field | Type | Purpose |
|---|---|---|
| `rbl_1` | String | RBL/Steig number for stop 1 (required) |
| `rbl_2` | String | RBL/Steig number for stop 2 (optional) |
| `rbl_3` | String | RBL/Steig number for stop 3 (optional) |
| `rbl_4` | String | RBL/Steig number for stop 4 (optional) |

### Finding RBL numbers

An RBL (a.k.a. *Steig* or *StopID*) identifies **one stop + one line + one direction**. So "U2 at Krieau toward Karlsplatz" is one RBL; "U2 at Krieau toward Seestadt" is a different RBL.

The Wiener Linien open-data dump is at <https://www.wienerlinien.at/open-data>. The three CSVs you'll touch are served directly from `wienerlinien.at/ogd_realtime/doku/ogd/`:

- [`wienerlinien-ogd-haltepunkte.csv`](https://www.wienerlinien.at/ogd_realtime/doku/ogd/wienerlinien-ogd-haltepunkte.csv) — RBL/Steig per platform, with stop name and DIVA (stop-area ID)
- [`wienerlinien-ogd-linien.csv`](https://www.wienerlinien.at/ogd_realtime/doku/ogd/wienerlinien-ogd-linien.csv) — line names (U2, 82A, 1, …) ↔ LineID
- [`wienerlinien-ogd-fahrwegverlaeufe.csv`](https://www.wienerlinien.at/ogd_realtime/doku/ogd/wienerlinien-ogd-fahrwegverlaeufe.csv) — joins LineID + StopID + direction

Two practical ways to find the RBL you want:

**A. Discovery via the realtime endpoint (fastest).** Look up the *DIVA* (one number per station — much easier than per-platform) by grepping the haltepunkte CSV by stop name, then query the realtime monitor with `diva=…`. Every monitor in the response includes its RBL and `towards`, so you can read off which RBL is which direction:

```bash
# 1. Find the DIVA for the stop name
curl -s https://www.wienerlinien.at/ogd_realtime/doku/ogd/wienerlinien-ogd-haltepunkte.csv | grep -i 'krieau'
# 4258;60201893;Krieau;...   ← DIVA is 60201893

# 2. See which RBLs serve which direction (only currently-active lines appear)
curl -s "https://www.wienerlinien.at/ogd_realtime/monitor?diva=60201893" \
  | jq '.data.monitors[] | {rbl: .locationStop.properties.attributes.rbl, line: .lines[0].name, towards: .lines[0].towards}'
# {"rbl":4258,"line":"U2","towards":"Karlsplatz"}
# {"rbl":4265,"line":"U2","towards":"Seestadt"}
```

**B. CSV join.** If the line you want isn't currently running (e.g. a night-only route), join `linien.csv` (LineID for your line name) with `fahrwegverlaeufe.csv` (filter by LineID and Direction) and look up the matching StopID in `haltepunkte.csv`.

Quick examples (verify before relying — IDs do change):

| Display | RBL |
|---|---|
| U2 → Karlsplatz @ Krieau | `4258` |
| U2 → Seestadt @ Krieau | `4265` |
| U4 → Heiligenstadt @ Schottenring | `4429` |
| D → Nußdorf @ Spittelau | `121` |

## Developing & testing

This repo is the source of truth for the templates, polling URL, and field definitions. To publish or update the plugin, paste these files into the TRMNL plugin builder.

### Set up the plugin in TRMNL

1. Go to <https://usetrmnl.com/plugins> → **Create new plugin** (Private is fine while iterating).
2. **Strategy:** `Polling`.
3. **Form fields:** paste [`form-fields.yml`](form-fields.yml) into the **Form fields** textarea.
4. **Polling URL:** paste the single line from [`polling-url.txt`](polling-url.txt). Method `GET`, no extra headers.
5. **Markup:** paste each `templates/*.liquid` file into its matching layout slot (`full`, `half_horizontal`, `half_vertical`, `quadrant`).
6. Install the plugin on your own account, fill in 1–4 RBL numbers, hit **Preview**.

### Verify the Wiener Linien response shape

Curl the polling URL with your RBLs substituted:

```bash
curl -s "https://www.wienerlinien.at/ogd_realtime/monitor?rbl=4429&rbl=121&rbl=247&rbl=4900" | jq
```

See [`sample-response.json`](sample-response.json) for a trimmed four-monitor example you can use as fixture data.

## Files

| Path | Purpose |
|---|---|
| [`form-fields.yml`](form-fields.yml) | YAML to paste into the TRMNL plugin builder's **Form fields** textarea |
| [`polling-url.txt`](polling-url.txt) | Single-line polling URL with `{{ }}` substitutions |
| [`templates/full.liquid`](templates/full.liquid) | 800×480 full-screen layout |
| [`templates/half_horizontal.liquid`](templates/half_horizontal.liquid) | 800×240 horizontal half |
| [`templates/half_vertical.liquid`](templates/half_vertical.liquid) | 400×480 vertical half |
| [`templates/quadrant.liquid`](templates/quadrant.liquid) | 400×240 quadrant |
| [`sample-response.json`](sample-response.json) | Trimmed Wiener Linien realtime response — useful as fixture data |

## Notes & caveats

- **Form field access in templates** — TRMNL exposes form values via `trmnl.plugin_settings.custom_fields_values.<keyname>`, not the `{{ keyname }}` shortcut. This plugin doesn't need them in the templates because all configuration is consumed by the polling URL.
- **Response root** — the Wiener Linien endpoint returns `{ data: { monitors: [...] }, message: {…} }`. TRMNL merges top-level JSON keys directly into the Liquid scope, so the templates access the monitors as `data.monitors` (the API's `data` key) and the server time as `message.serverTime`.
- **Empty RBL slots are fine** — the API accepts empty `rbl=` parameters and just ignores them, so unused slots in the form don't break anything.
- **Realtime vs planned** — when `realtimeSupported` is true, `timeReal` is used and reflects live tracking. Otherwise the schedule (`timePlanned`) is shown. The full layout flags planned-only lines with a `· planned` suffix.
- **One RBL = one direction** — to show both directions of the same line at the same stop, you need two RBLs (one per direction).
- **HTTPS only** — TRMNL's polling worker won't talk to plain `http://` endpoints. The Wiener Linien endpoint is HTTPS.
- **Rate limits** — Wiener Linien's OGD endpoint is public and unauthenticated; please don't poll faster than necessary. TRMNL's default refresh cadence is well within their guidelines.

## Contributing

PRs welcome. The cycle is: edit a file here → paste it into the TRMNL plugin builder → preview → commit.

## License

[MIT](LICENSE)
