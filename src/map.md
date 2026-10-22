---
toc: false
---

# Portland Harbor VOC monitoring

Explore volatile organic compound (VOC) measurements from monitoring stations around Portland Harbor. Select a compound and station to inspect recorded concentrations, available wind observations, and changes over time.

This dashboard presents monitoring data for exploration. Screening references are contextual tools, not determinations that an individual reading is safe or unsafe.

```js
import * as d3 from "npm:d3";
import * as Plot from "npm:@observablehq/plot";
import * as Inputs from "npm:@observablehq/inputs";
```

```js
const maplibregl = await new Promise((resolve, reject) => {
  if (window.maplibregl) return resolve(window.maplibregl);

  const css = document.createElement("link");
  css.rel = "stylesheet";
  css.href = "https://unpkg.com/maplibre-gl@4.7.1/dist/maplibre-gl.css";
  document.head.appendChild(css);

  const script = document.createElement("script");
  script.src = "https://unpkg.com/maplibre-gl@4.7.1/dist/maplibre-gl.js";
  script.onload = () => resolve(window.maplibregl);
  script.onerror = reject;
  document.head.appendChild(script);
});
```

```js
const vocRows = await FileAttachment("data/voc-data.csv").csv({
  typed: true,
});
```

## Data preparation

The code below detects common column-name variants so it can work with CSV headers such as `Station`, `Station ID`, `Date`, `Sample Date`, `Benzene`, and `Benzene (ppbv)`. If your file uses different headers, adjust the aliases in this section.

```js
function normalizeKey(value) {
  return String(value ?? "")
    .toLowerCase()
    .replace(/[^a-z0-9]/g, "");
}

function findColumn(columns, aliases) {
  const normalized = columns.map((name) => ({
    name,
    key: normalizeKey(name),
  }));

  for (const alias of aliases) {
    const match = normalized.find(
      (column) => column.key === normalizeKey(alias),
    );
    if (match) return match.name;
  }

  return null;
}

function firstValue(row, aliases) {
  const columns = Object.keys(row);
  const column = findColumn(columns, aliases);
  return column ? row[column] : null;
}

function numericValue(value) {
  if (value == null || String(value).trim() === "") return null;

  const cleaned = String(value).replace(/,/g, "").replace(/[<>]/g, "").trim();

  const number = Number(cleaned);
  return Number.isFinite(number) ? number : null;
}

function parseDate(value) {
  if (!value) return null;

  const date = new Date(value);
  return Number.isNaN(date.getTime()) ? null : date;
}

function escapeHTML(value) {
  return String(value ?? "").replace(
    /[&<>"']/g,
    (character) =>
      ({
        "&": "&amp;",
        "<": "&lt;",
        ">": "&gt;",
        '"': "&quot;",
        "'": "&#39;",
      })[character],
  );
}

function formatNumber(value, digits = 3) {
  return value == null || !Number.isFinite(value)
    ? "—"
    : value.toLocaleString(undefined, { maximumFractionDigits: digits });
}

function formatDate(value) {
  return value ? d3.timeFormat("%b %d, %Y")(value) : "Date unavailable";
}
```

```js
const columns = Object.keys(vocRows[0] ?? {});

const columnMap = {
  station: findColumn(columns, [
    "station",
    "station id",
    "station code",
    "site",
    "site id",
    "location",
  ]),
  date: findColumn(columns, [
    "sample date",
    "sampling date",
    "date",
    "datetime",
    "date time",
    "sample datetime",
    "timestamp",
  ]),
  latitude: findColumn(columns, ["latitude", "lat", "station latitude"]),
  longitude: findColumn(columns, [
    "longitude",
    "long",
    "lon",
    "lng",
    "station longitude",
  ]),
  windDirection: findColumn(columns, [
    "wind direction",
    "wind dir",
    "wind compass",
    "compass",
  ]),
  windBearing: findColumn(columns, [
    "wind bearing",
    "bearing",
    "wind direction degrees",
    "wind degrees",
  ]),
  windGust: findColumn(columns, [
    "wind gust",
    "gust",
    "wind gust mph",
    "gust speed",
  ]),
};

const stationMetadata = {
  POG: {
    name: "Ocean Gateway",
    latitude: 43.65900514,
    longitude: -70.24548956,
  },
  PWC: { name: "PWC" },
  SPFS: { name: "SPFS" },
  SPCC: { name: "SPCC" },
  SPMS: { name: "SPMS" },
  SPPS: { name: "SPPS" },
  SPRB: { name: "SPRB" },
};

function stationCode(row) {
  const value = firstValue(row, [
    "station",
    "station id",
    "station code",
    "site",
    "site id",
    "location",
  ]);

  const text = String(value ?? "")
    .trim()
    .toUpperCase();

  for (const code of Object.keys(stationMetadata)) {
    if (
      text === code ||
      text.startsWith(`${code} `) ||
      text.startsWith(`${code}-`) ||
      text.startsWith(`${code}_`)
    ) {
      return code;
    }
  }

  return text || "Unknown";
}

function getChemicalColumns(rows) {
  const excluded = new Set(
    [
      columnMap.station,
      columnMap.date,
      columnMap.latitude,
      columnMap.longitude,
      columnMap.windDirection,
      columnMap.windBearing,
      columnMap.windGust,
    ].filter(Boolean),
  );

  const metadataNames = new Set([
    "station",
    "stationid",
    "stationcode",
    "site",
    "siteid",
    "location",
    "date",
    "sampledate",
    "samplingdate",
    "datetime",
    "sampledatetime",
    "timestamp",
    "latitude",
    "lat",
    "longitude",
    "long",
    "lon",
    "lng",
    "winddirection",
    "winddir",
    "windcompass",
    "compass",
    "windbearing",
    "bearing",
    "windgust",
    "gust",
  ]);

  return columns.filter((column) => {
    if (excluded.has(column) || metadataNames.has(normalizeKey(column)))
      return false;

    const values = rows
      .slice(0, 100)
      .map((row) => row[column])
      .filter((value) => value != null && String(value).trim() !== "");

    return (
      values.length > 0 && values.some((value) => numericValue(value) != null)
    );
  });
}

const chemicalColumns = getChemicalColumns(vocRows);

function chemicalLabel(column) {
  return String(column)
    .replace(/\s*\(ppbv\)\s*/i, "")
    .replace(/\s*\(ppb\)\s*/i, "")
    .replace(/\s*\(ppm\)\s*/i, "")
    .trim();
}

function concentrationUnit(column) {
  const match = String(column).match(/\(([^)]+)\)/);
  return match ? match[1] : "ppbv";
}

const normalizedRows = vocRows.map((row) => {
  const station = stationCode(row);
  const metadata = stationMetadata[station] ?? {};

  const latitude = numericValue(
    firstValue(row, ["latitude", "lat", "station latitude"]),
  );
  const longitude = numericValue(
    firstValue(row, ["longitude", "long", "lon", "lng", "station longitude"]),
  );

  return {
    raw: row,
    station,
    stationName: metadata.name ?? station,
    date: parseDate(
      firstValue(row, [
        "sample date",
        "sampling date",
        "date",
        "datetime",
        "date time",
        "sample datetime",
        "timestamp",
      ]),
    ),
    latitude: latitude ?? metadata.latitude ?? null,
    longitude: longitude ?? metadata.longitude ?? null,
    windDirection: firstValue(row, [
      "wind direction",
      "wind dir",
      "wind compass",
      "compass",
    ]),
    windBearing: numericValue(
      firstValue(row, [
        "wind bearing",
        "bearing",
        "wind direction degrees",
        "wind degrees",
      ]),
    ),
    windGust: numericValue(
      firstValue(row, ["wind gust", "gust", "wind gust mph", "gust speed"]),
    ),
    ...Object.fromEntries(
      chemicalColumns.map((column) => [column, numericValue(row[column])]),
    ),
  };
});

const availableStations = Array.from(
  new Set(
    normalizedRows
      .map((row) => row.station)
      .filter((station) => station && station !== "Unknown"),
  ),
).sort();
```

## Select a compound and station

```js
const selectedChemical = view(
  Inputs.select(chemicalColumns, {
    label: "VOC / compound",
    value:
      chemicalColumns.find((column) =>
        normalizeKey(column).startsWith("benzene"),
      ) ?? chemicalColumns[0],
    format: chemicalLabel,
  }),
);
```

```js
const selectedStation = view(
  Inputs.select(availableStations, {
    label: "Monitoring station",
    value: availableStations.includes("POG") ? "POG" : availableStations[0],
    format: (code) => stationMetadata[code]?.name ?? code,
  }),
);
```

## Screening reference

```js
display(html`
  <div
    style="border-left:3px solid var(--theme-foreground-faint);padding:10px 14px;margin:12px 0;"
  >
    ${selectedReference
      ? html`
          <strong>
            ${escapeHTML(chemicalLabel(selectedChemical))}:
            ${escapeHTML(selectedReference.title)}
          </strong>

          <div style="display:flex;flex-wrap:wrap;gap:16px;margin:12px 0;">
            <div>
              <div style="color:#e53935;font-weight:700;">Acute</div>
              <div style="color:#e53935;font-size:20px;font-weight:650;">
                ${selectedReference.acute ?? "—"}
                ${escapeHTML(selectedReference.unit)}
              </div>
            </div>

            <div>
              <div style="color:#f28c28;font-weight:700;">Intermediate</div>
              <div style="color:#f28c28;font-size:20px;font-weight:650;">
                ${selectedReference.intermediate ?? "—"}
                ${escapeHTML(selectedReference.unit)}
              </div>
            </div>

            <div>
              <div style="color:#d4a900;font-weight:700;">Chronic</div>
              <div style="color:#d4a900;font-size:20px;font-weight:650;">
                ${selectedReference.chronic ?? "—"}
                ${escapeHTML(selectedReference.unit)}
              </div>
            </div>
          </div>

          <p style="margin:8px 0;">
            <strong style="color:#e53935;">Acute (1–14 days):</strong>
            This acute inhalation MRL is intended for screening exposures
            lasting 1–14 days. A comparison with an individual measurement does
            not establish whether a person experienced harmful effects.
          </p>

          <p style="margin:8px 0;">
            <strong style="color:#f28c28;">Intermediate (15–364 days):</strong>
            This intermediate inhalation MRL is intended for screening exposures
            lasting 15–364 days. It is not a regulatory limit, and a comparison
            alone does not establish whether harmful effects occurred.
          </p>

          <p style="margin:8px 0;">
            <strong style="color:#d4a900;"
              >Chronic (365 days or longer):</strong
            >
            This chronic inhalation MRL is intended for screening exposures
            lasting a year or longer. It is not a regulatory limit, and a
            comparison alone does not establish whether harmful effects
            occurred.
          </p>

          <p style="margin:6px 0;font-size:13px;">
            MRLs are health-based screening guidelines for non-cancer effects. A
            missing value means no reference is configured for that duration; it
            does not mean zero risk.
          </p>

          <a
            href="${selectedReference.source ?? mrlSource}"
            target="_blank"
            rel="noopener noreferrer"
          >
            View the official ATSDR source
          </a>
        `
      : html`
          <strong>
            No screening reference configured for
            ${escapeHTML(chemicalLabel(selectedChemical))}
          </strong>
          <p style="margin:6px 0;">
            No verified acute, intermediate, or chronic inhalation MRL is
            configured for this compound. No comparison thresholds are
            displayed.
          </p>
        `}
  </div>
`);
```

```js
const mrlSource = "https://wwwn.cdc.gov/TSP/MRLS/mrlslisting.aspx";

// MRL values in ppbv unless otherwise specified.
// null = no value configured for that exposure duration.
const screeningReferences = {
  Acrolein: {
    acute: 3,
    intermediate: 0.4,
    chronic: 0.4,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — acrolein",
    description:
      "ATSDR health-based screening references for non-cancer effects.",
    source: mrlSource,
  },

  Benzene: {
    acute: 9,
    intermediate: 7,
    chronic: 2,
    unit: "ppbv",
    status: "Draft",
    title: "ATSDR inhalation Minimal Risk Levels — benzene",
    description:
      "Draft ATSDR inhalation screening references for non-cancer effects.",
    source: mrlSource,
  },

  Ethylbenzene: {
    acute: 5000,
    intermediate: 2000,
    chronic: 60,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — ethylbenzene",
    description:
      "ATSDR health-based screening references for non-cancer effects.",
    source: mrlSource,
  },

  Toluene: {
    acute: 2000,
    intermediate: null,
    chronic: 1000,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — toluene",
    description:
      "No intermediate-duration value is configured in this dataset.",
    source: mrlSource,
  },

  "Methylene Chloride": {
    acute: 600,
    intermediate: 300,
    chronic: 300,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — methylene chloride",
    description:
      "ATSDR health-based screening references for non-cancer effects.",
    source: mrlSource,
  },

  Tetrachloroethylene: {
    acute: 6,
    intermediate: 6,
    chronic: 6,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — tetrachloroethylene",
    description: "Also known as perchloroethylene or PCE.",
    source: mrlSource,
  },

  "1,4-Dichlorobenzene": {
    acute: 2000,
    intermediate: 200,
    chronic: 10,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — 1,4-dichlorobenzene",
    description:
      "ATSDR health-based screening references for non-cancer effects.",
    source: mrlSource,
  },

  "Carbon Tetrachloride": {
    acute: null,
    intermediate: 30,
    chronic: 30,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — carbon tetrachloride",
    description: "No acute-duration value is configured in this dataset.",
    source: mrlSource,
  },

  Chloroform: {
    acute: 1,
    intermediate: 0.8,
    chronic: 0.4,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — chloroform",
    description:
      "ATSDR health-based screening references for non-cancer effects.",
    source: mrlSource,
  },

  "1,1,1-Trichloroethane": {
    acute: 1000,
    intermediate: 700,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — 1,1,1-trichloroethane",
    description: "No chronic-duration value is configured in this dataset.",
    source: mrlSource,
  },

  "1,1,2-Trichloroethane": {
    acute: 30,
    intermediate: 2,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — 1,1,2-trichloroethane",
    description: "No chronic-duration value is configured in this dataset.",
    source: mrlSource,
  },

  "1,1-Dichloroethene": {
    acute: null,
    intermediate: 1,
    chronic: 1,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — 1,1-dichloroethene",
    description: "No acute-duration value is configured in this dataset.",
    source: mrlSource,
  },

  "Ethylene Dichloride": {
    acute: 100,
    intermediate: 100,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — ethylene dichloride",
    description:
      "Also known as 1,2-dichloroethane. No chronic-duration value is configured in this dataset.",
    source: mrlSource,
  },

  "Trans-1,2-Dichloroethene": {
    acute: 3000,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — trans-1,2-dichloroethene",
    description:
      "No intermediate- or chronic-duration values are configured in this dataset.",
    source: mrlSource,
  },

  Trichloroethylene: {
    acute: null,
    intermediate: 0.4,
    chronic: 0.4,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — trichloroethylene",
    description:
      "Also known as TCE. No acute-duration value is configured in this dataset.",
    source: mrlSource,
  },

  "Vinyl Chloride": {
    acute: 500,
    intermediate: 20,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — vinyl chloride",
    description: "No chronic-duration value is configured in this dataset.",
    source: mrlSource,
  },

  "Methyl Chloride": {
    acute: 500,
    intermediate: 300,
    chronic: 30,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — methyl chloride",
    description: "Also known as chloromethane.",
    source: mrlSource,
  },

  "Ethyl Chloride": {
    acute: 13000,
    intermediate: 13000,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — ethyl chloride",
    description:
      "Also known as chloroethane. No chronic-duration value is configured in this dataset.",
    source: mrlSource,
  },

  "Methyl Bromide": {
    acute: null,
    intermediate: 20,
    chronic: 1,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — methyl bromide",
    description:
      "Also known as bromomethane. No acute-duration value is configured in this dataset.",
    source: mrlSource,
  },

  Naphthalene: {
    acute: 0.06,
    intermediate: null,
    chronic: null,
    unit: "ppb",
    status: "Final",
    title: "ATSDR inhalation Minimal Risk Levels — naphthalene",
    description:
      "The configured acute value uses ppb as supplied. Confirm the unit matches the measurement data before comparing.",
    source: mrlSource,
  },

  "m,p-Xylenes": {
    acute: 2000,
    intermediate: 400,
    chronic: 10,
    unit: "ppbv",
    status: "Draft",
    title: "ATSDR inhalation Minimal Risk Levels — mixed xylenes",
    description:
      "The ATSDR reference is for mixed xylenes, not an isomer-specific MRL. Confirm applicability to this measurement column before interpreting comparisons.",
    source: mrlSource,
    note: "ATSDR reference is for mixed xylenes, not an isomer-specific MRL.",
  },
};

const selectedReference = screeningReferences[selectedChemical] ?? null;

function getScreeningReference(column) {
  const key = normalizeKey(chemicalLabel(column));
  const reference = screeningReferences[key];

  if (!reference) return null;

  return reference.unit === concentrationUnit(column) ? reference : null;
}
```

```js
const selectedRows = normalizedRows
  .filter((row) => row.station === selectedStation)
  .filter((row) => row[selectedChemical] != null)
  .sort((a, b) => (a.date?.getTime() ?? 0) - (b.date?.getTime() ?? 0));

const latestReading =
  [...selectedRows]
    .filter((row) => row.date)
    .sort((a, b) => b.date - a.date)[0] ??
  selectedRows.at(-1) ??
  null;

const stationInfo = stationMetadata[selectedStation] ?? {
  name: selectedStation,
};

const stationLatitude = latestReading?.latitude ?? stationInfo.latitude ?? null;

const stationLongitude =
  latestReading?.longitude ?? stationInfo.longitude ?? null;
```

```js
display(html`
  <div
    style="border:1px solid var(--theme-foreground-faint);border-radius:12px;padding:18px;margin:12px 0;"
  >
    <div style="font-size:13px;color:var(--theme-foreground-muted);">
      ${escapeHTML(selectedStation)} ·
      ${escapeHTML(chemicalLabel(selectedChemical))}
    </div>
    <h2 style="margin:8px 0;">
      ${escapeHTML(stationInfo.name ?? selectedStation)}
    </h2>
    <div style="color:var(--theme-foreground-muted);font-size:13px;">
      ${stationLatitude != null && stationLongitude != null
        ? `${stationLatitude.toFixed(8)}, ${stationLongitude.toFixed(8)}`
        : "Station coordinates unavailable"}
    </div>
    <div style="margin-top:20px;font-size:16px;">
      Latest recorded concentration
    </div>
    <div style="font-size:32px;font-weight:650;margin:4px 0;">
      ${latestReading
        ? `${formatNumber(latestReading[selectedChemical])} ${escapeHTML(concentrationUnit(selectedChemical))}`
        : "No reading available"}
    </div>
    <div style="font-size:13px;color:var(--theme-foreground-muted);">
      ${latestReading
        ? `Sample date: ${formatDate(latestReading.date)}`
        : "No measurement is fabricated for this station."}
    </div>
    <hr
      style="margin:18px 0;border:0;border-top:1px solid var(--theme-foreground-faint);"
    />
    <div
      style="display:grid;grid-template-columns:repeat(auto-fit,minmax(140px,1fr));gap:16px;"
    >
      <div>
        <div style="font-size:12px;color:var(--theme-foreground-muted);">
          Wind direction
        </div>
        <div style="font-size:18px;font-weight:600;">
          ${latestReading?.windDirection ||
          (latestReading?.windBearing != null
            ? `${latestReading.windBearing}°`
            : "—")}
        </div>
      </div>
      <div>
        <div style="font-size:12px;color:var(--theme-foreground-muted);">
          Wind bearing
        </div>
        <div style="font-size:18px;font-weight:600;">
          ${latestReading?.windBearing != null
            ? `${latestReading.windBearing}°`
            : "—"}
        </div>
      </div>
      <div>
        <div style="font-size:12px;color:var(--theme-foreground-muted);">
          Wind gust
        </div>
        <div style="font-size:18px;font-weight:600;">
          ${latestReading?.windGust != null
            ? formatNumber(latestReading.windGust)
            : "—"}
        </div>
      </div>
    </div>
  </div>
`);
```

## Map of monitoring stations & terminals

Petroleum terminals are optional contextual markers. Their locations below are not supplied because verified terminal coordinates should be used rather than guessed.

```js
const terminals = [
  {
    name: "CITGO South Portland Terminal",
    address: "102 Mechanic St, South Portland, ME 04106",
    longitude: -70.2393,
    latitude: 43.6425,
  },
  {
    name: "South Portland Terminal LLC",
    address: "170 Lincoln St, South Portland, ME 04106",
    longitude: -70.2856,
    latitude: 43.6356,
  },
  {
    name: "Portland Pipe Line Corporation",
    address: "30 Hill St, South Portland, ME 04106",
    longitude: -70.2641,
    latitude: 43.6277,
  },
  {
    name: "Global Partners Terminal",
    address: "1 Clarks Rd, South Portland, ME 04106",
    longitude: -70.2801,
    latitude: 43.6331,
  },
  {
    name: "Sprague Energy Corp.",
    address: "59 Main St, South Portland, ME 04106",
    longitude: -70.2442,
    latitude: 43.6443,
  },
  {
    name: "Sunoco / Gulf Oil Terminal",
    address: "175 Front St, South Portland, ME 04106",
    longitude: -70.2319,
    latitude: 43.6508,
  },
];

const terminalGeoJSON = {
  type: "FeatureCollection",
  features: terminals.map((terminal) => ({
    type: "Feature",
    properties: {
      name: terminal.name,
      address: terminal.address,
    },
    geometry: {
      type: "Point",
      coordinates: [terminal.longitude, terminal.latitude],
    },
  })),
};
```

```js
const showTerminals = view(
  Inputs.checkbox(["Show petroleum terminals"], {
    label: "Map layers",
    value: ["Show petroleum terminals"],
  }),
);
```

```js
const terminalLocations = [
  // Add verified terminal coordinates here:
  // { name: "Terminal name", latitude: 43.x, longitude: -70.x }
];

const mappedStations = availableStations
  .map((code) => {
    const metadata = stationMetadata[code] ?? {};
    const row = [...normalizedRows]
      .filter((item) => item.station === code)
      .find((item) => item.latitude != null && item.longitude != null);

    const latitude = metadata.latitude ?? row?.latitude ?? null;
    const longitude = metadata.longitude ?? row?.longitude ?? null;

    if (latitude == null || longitude == null) return null;

    const latest = [...normalizedRows]
      .filter((item) => item.station === code && item[selectedChemical] != null)
      .sort((a, b) => (b.date?.getTime() ?? 0) - (a.date?.getTime() ?? 0))[0];

    return {
      type: "Feature",
      geometry: { type: "Point", coordinates: [longitude, latitude] },
      properties: {
        station: code,
        name: metadata.name ?? code,
        concentration: latest?.[selectedChemical] ?? null,
        date: latest?.date ? formatDate(latest.date) : "No dated reading",
      },
    };
  })
  .filter(Boolean);

const stationGeoJSON = {
  type: "FeatureCollection",
  features: mappedStations,
};
```

```js
const mapContainer = html`
  <div
    style="
    position: relative;
    width: 100%;
    height: 650px;
    min-width: 0;
    border-radius: 10px;
    overflow: hidden;
    background: #e8edf0;
  "
  >
    <div
      class="map-inner"
      style="
      position: absolute;
      inset: 0;
      width: 100%;
      height: 100%;
    "
    ></div>
  </div>
`;

display(mapContainer);
```

```js
{
  const map = new maplibregl.Map({
    container: mapContainer.querySelector(".map-inner"),
    style: {
      version: 8,
      glyphs: "https://demotiles.maplibre.org/font/{fontstack}/{range}.pbf",
      sources: {
        basemap: {
          type: "raster",
          tiles: [
            "https://services.arcgisonline.com/ArcGIS/rest/services/Canvas/World_Light_Gray_Base/MapServer/tile/{z}/{y}/{x}",
          ],
          tileSize: 256,
          attribution: "Tiles © Esri",
        },
      },
      layers: [
        {
          id: "basemap",
          type: "raster",
          source: "basemap",
        },
      ],
    },
    center: [-70.24548956, 43.65900514],
    zoom: 11,
  });

  // Ensure the map canvas fills the container.
  const canvas = map.getCanvas();

  map.addControl(new maplibregl.NavigationControl(), "top-right");

  const mapElement = mapContainer.querySelector(".map-inner");

  const resizeObserver = new ResizeObserver(() => {
    map.resize();
  });

  resizeObserver.observe(mapElement);

  map.on("load", () => {
    map.addSource("stations", {
      type: "geojson",
      data: stationGeoJSON,
    });

    map.addLayer({
      id: "station-circles",
      type: "circle",
      source: "stations",
      paint: {
        "circle-radius": [
          "case",
          ["==", ["get", "station"], selectedStation],
          9,
          6,
        ],
        "circle-color": [
          "case",
          ["==", ["get", "station"], selectedStation],
          "#00a6d6",
          "#315d78",
        ],
        "circle-stroke-color": "#ffffff",
        "circle-stroke-width": 2,
      },
    });

    map.addLayer({
      id: "station-labels",
      type: "symbol",
      source: "stations",
      layout: {
        "text-field": ["get", "station"],
        "text-size": 12,
        "text-offset": [0, 1.3],
        "text-allow-overlap": true,
      },
      paint: {
        "text-color": "#000000",
        "text-halo-color": "#ffffff",
        "text-halo-width": 1.5,
      },
    });

    map.on("click", "station-circles", (event) => {
    const feature = event.features?.[0];
    if (!feature) return;

    const properties = feature.properties;
    const coordinates = feature.geometry.coordinates.slice();

    new maplibregl.Popup()
        .setLngLat(coordinates)
        .setHTML(
        `
            <div style="font-family:system-ui;min-width:180px;color:#000;">
            <strong style="color:#000;">
                ${escapeHTML(properties.station)}
            </strong>
            <div style="margin-top:8px;color:#000;">
                ${escapeHTML(chemicalLabel(selectedChemical))}
            </div>
            <div style="font-size:18px;font-weight:650;margin:3px 0;color:#000;">
                ${
                properties.concentration == null
                    ? "No reading available"
                    : `${formatNumber(Number(properties.concentration))} ${escapeHTML(concentrationUnit(selectedChemical))}`
                }
            </div>
            <div style="font-size:12px;color:#000;">
                ${escapeHTML(properties.date)}
            </div>
            </div>
        `,
        )
        .addTo(map);
    });

    map.on("mouseenter", "station-circles", () => {
      map.getCanvas().style.cursor = "pointer";
    });

    map.on("mouseleave", "station-circles", () => {
      map.getCanvas().style.cursor = "";
    });

    map.addSource("terminals", {
      type: "geojson",
      data: terminalGeoJSON,
    });

    const terminalVisibility = showTerminals.includes(
      "Show petroleum terminals",
    )
      ? "visible"
      : "none";

    map.addLayer({
      id: "terminal-circles",
      type: "circle",
      source: "terminals",
      layout: {
        visibility: terminalVisibility,
      },
      paint: {
        "circle-radius": 7,
        "circle-color": "#f2c94c",
        "circle-stroke-color": "#ffffff",
        "circle-stroke-width": 2,
      },
    });

    map.addLayer({
      id: "terminal-labels",
      type: "symbol",
      source: "terminals",
      layout: {
        "text-field": ["get", "name"],
        "text-size": 11,
        "text-offset": [0, 1.3],
        "text-allow-overlap": true,
        visibility: terminalVisibility,
      },
      paint: {
        "text-color": "#000000",
        "text-halo-color": "#ffffff",
        "text-halo-width": 1.5,
      },
    });

    map.on("click", "terminal-circles", (event) => {
      const feature = event.features?.[0];
      if (!feature) return;

      const { name, address } = feature.properties;
      const coordinates = feature.geometry.coordinates.slice();

      new maplibregl.Popup()
        .setLngLat(coordinates)
        .setHTML(
          `
            <div style="font-family:system-ui;min-width:200px;color:#000;">
                <strong>${escapeHTML(name)}</strong>
                <div style="margin-top:8px;font-size:13px;line-height:1.5;color:#000;">
                ${escapeHTML(address)}
                </div>
                <div style="margin-top:8px;font-size:12px;color:#000;">
                Petroleum terminal
                </div>
            </div>
            `,
        )
        .addTo(map);
    });

    map.on("mouseenter", "terminal-circles", () => {
      map.getCanvas().style.cursor = "pointer";
    });

    map.on("mouseleave", "terminal-circles", () => {
      map.getCanvas().style.cursor = "";
    });

    const validFeatures = stationGeoJSON.features;

    if (validFeatures.length > 1) {
      const bounds = new maplibregl.LngLatBounds();

      validFeatures.forEach((feature) => {
        bounds.extend(feature.geometry.coordinates);
      });

      map.fitBounds(bounds, {
        padding: 50,
        maxZoom: 12,
      });
    }

    map.resize();
  });

  invalidation.then(() => {
    resizeObserver.disconnect();
    map.remove();
  });
}
```

```js
display(html`
  <style>
    .maplibregl-popup-close-button {
      color: #000000 !important;
    }

    .maplibregl-popup-close-button:hover {
      color: #000000 !important;
      background-color: #f0f0f0;
    }
  </style>
`);
```

## Concentration over time

```js
function getMrlAssessment(value) {
  const concentration = Number(value);

  if (
    !selectedReference ||
    value == null ||
    value === "" ||
    !Number.isFinite(concentration)
  ) {
    return {
      color: "#008fbd",
      level: null,
      label: "Below configured MRLs",
    };
  }

  if (
    selectedReference.acute != null &&
    concentration >= selectedReference.acute
  ) {
    return {
      color: "#e53935",
      level: "acute",
      label: "At or above acute MRL",
      threshold: selectedReference.acute,
    };
  }

  if (
    selectedReference.intermediate != null &&
    concentration >= selectedReference.intermediate
  ) {
    return {
      color: "#f28c28",
      level: "intermediate",
      label: "At or above intermediate MRL",
      threshold: selectedReference.intermediate,
    };
  }

  if (
    selectedReference.chronic != null &&
    concentration >= selectedReference.chronic
  ) {
    return {
      color: "#d4a900",
      level: "chronic",
      label: "At or above chronic MRL",
      threshold: selectedReference.chronic,
    };
  }

  return {
    color: "#008fbd",
    level: null,
    label: "Below configured MRLs",
  };
}
```

```js
const brushRangeInput = (() => {
  const node = html`<div hidden></div>`;
  node.value = null;
  return node;
})();

const brushRange = Generators.input(brushRangeInput);
```

```js
const selectedReportInput = (() => {
  const node = html`<div hidden></div>`;
  node.value = null;
  return node;
})();

const selectedReport = Generators.input(selectedReportInput);
```

```js
const chartRows = selectedRows
  .filter(
    (row) =>
      row.date instanceof Date &&
      !isNaN(row.date.getTime()) &&
      row[selectedChemical] != null &&
      Number.isFinite(Number(row[selectedChemical])),
  )
  .sort((a, b) => a.date - b.date);

const fullDateExtent = d3.extent(chartRows, (row) => row.date);

const rangeRows = chartRows.filter(
  (row) =>
    !brushRange || (row.date >= brushRange[0] && row.date <= brushRange[1]),
);
```

```js
display(html`
  <h2>
    ${brushRange
      ? `Reports: ${formatDateRange(brushRange)}`
      : "Reports: Full timeline"}
  </h2>

  ${Plot.plot({
    width: 900,
    height: 330,
    marginLeft: 70,
    marginBottom: 55,
    x: {
      type: "time",
      label: "Sample date",
      grid: true,
    },
    y: {
      label: `${chemicalLabel(selectedChemical)} (${concentrationUnit(selectedChemical)})`,
      zero: true,
      grid: true,
    },
    marks: [
      Plot.line(rangeRows, {
        x: "date",
        y: (d) => d[selectedChemical],
        stroke: "#008fbd",
        strokeOpacity: 0.6,
      }),

      Plot.dot(rangeRows, {
        x: "date",
        y: (d) => d[selectedChemical],
        r: 4,
        fill: (d) => getMrlAssessment(d[selectedChemical]).color,
        stroke: "white",
        strokeWidth: 0.7,
        tip: true,
        title: (d) => {
          const value = Number(d[selectedChemical]);
          const assessment = getMrlAssessment(value);

          return `${formatNumber(value)} ${concentrationUnit(selectedChemical)}
        Sample date: ${formatDate(d.date)}
        ${assessment.label}`;
        },
      }),

      // Larger invisible targets make the points easier to click.
      (index, scales) => {
        const g = d3.create("svg:g");

        g.selectAll("circle")
          .data(rangeRows)
          .join("circle")
          .attr("cx", (d) => scales.x(d.date))
          .attr("cy", (d) => scales.y(d[selectedChemical]))
          .attr("r", 10)
          .attr("fill", "transparent")
          .style("cursor", "pointer")
          .append("title")
          .text(
            (d) =>
              `${formatNumber(d[selectedChemical])} ${concentrationUnit(selectedChemical)} — Sample date: ${formatDate(d.date)}`,
          );

        g.selectAll("circle").on("click", (event, d) => {
          selectedReportInput.value = d;
          selectedReportInput.dispatchEvent(new Event("input"));
        });

        return g.node();
      },

      Plot.ruleY([0], {
        stroke: "#888",
        strokeOpacity: 0.4,
      }),
    ],
  })}
  ${selectedReport
    ? html`
        <div
          style="
        margin: 12px 0 20px;
        padding: 16px;
        border: 1px solid #008fbd;
        border-radius: 8px;
        background: var(--theme-background);
        max-width: 420px;
      "
        >
          <div
            style="
          font-size: 20px;
          font-weight: 700;
          color: ${getMrlAssessment(selectedReport[selectedChemical]).color};
        "
          >
            ${formatNumber(selectedReport[selectedChemical])}
            ${concentrationUnit(selectedChemical)}
          </div>

          <div
            style="
          margin-top: 5px;
          font-size: 13px;
          color: ${getMrlAssessment(selectedReport[selectedChemical]).color};
        "
          >
            ${getMrlAssessment(selectedReport[selectedChemical]).label}
          </div>

          <div style="margin-top: 6px;">
            <strong>Sample date:</strong>
            ${formatDate(selectedReport.date)}
          </div>

          <div style="margin-top: 4px;">
            <strong>Station:</strong>
            ${selectedStation}
          </div>

          <button
            style="margin-top: 12px; cursor: pointer;"
            onclick=${() => {
              selectedReportInput.value = null;
              selectedReportInput.dispatchEvent(new Event("input"));
            }}
          >
            Close
          </button>
        </div>
      `
    : ""}
`);
```

```js
function normalizeConcentrationUnit(unit) {
  const normalized = unit?.trim().toLowerCase();

  // For gas-phase concentrations, ppb and ppbv both represent
  // parts per billion by volume when the source uses them equivalently.
  if (normalized === "ppb" || normalized === "ppbv") {
    return "ppbv";
  }

  return normalized;
}
```

```js
{
  const button = document.querySelector(".clear-report");

  if (button) {
    button.addEventListener("click", () => {
      selectedReportInput.value = null;
      selectedReportInput.dispatchEvent(new Event("input"));
    });
  }
}
```

```js
const minRow = chartRows.reduce(
(min, row) =>
    min === null ||
    Number(row[selectedChemical]) < Number(min[selectedChemical])
    ? row
    : min,
null,
);

const maxRow = chartRows.reduce(
(max, row) =>
    max === null ||
    Number(row[selectedChemical]) > Number(max[selectedChemical])
    ? row
    : max,
null,
);

const minValue = minRow ? Number(minRow[selectedChemical]) : null;
const maxValue = maxRow ? Number(maxRow[selectedChemical]) : null;

const mrlReference = screeningReferences[selectedChemical];

const mrlThresholds = [
  { duration: "Acute", value: mrlReference?.acute, color: "#e53935" },
  { duration: "Intermediate", value: mrlReference?.intermediate, color: "#f28c28" },
  { duration: "Chronic", value: mrlReference?.chronic, color: "#d4a900" },
].filter(
  (d) =>
    d.value != null &&
    Number.isFinite(d.value) &&
    normalizeConcentrationUnit(mrlReference?.unit) ===
      normalizeConcentrationUnit(concentrationUnit(selectedChemical)),
);

display(html`
<div
    style="
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 24px;
    flex-wrap: wrap;
    margin-bottom: 12px;
    "
>
    <div>
    <h2 style="margin: 0;">Full timeline</h2>
    <p
        style="
        font-size: 13px;
        color: var(--theme-foreground-muted);
        margin: 6px 0 0;
        "
    >
        Drag across the chart to select a date range. Clear the selection to
        restore the full range.
    </p>
    </div>

    <div
    style="
        display: flex;
        gap: 24px;
        align-items: center;
        flex-wrap: wrap;
    "
    >
    <div>
        <div
        style="
            font-size: 12px;
            color: var(--theme-foreground-muted);
        "
        >
        Minimum
        </div>

        <strong
        style="
            color: ${minValue == null
            ? "var(--theme-foreground)"
            : getMrlAssessment(minValue).color};
        "
        >
        ${minValue == null
            ? "N/A"
            : `${formatNumber(minValue)} ${concentrationUnit(selectedChemical)}`}
        </strong>

        <div
        style="
            font-size: 12px;
            color: var(--theme-foreground-muted);
            margin-top: 3px;
        "
        >
        ${minRow ? formatDate(minRow.date) : "No date"}
        </div>
    </div>

    <div>
        <div
        style="
            font-size: 12px;
            color: var(--theme-foreground-muted);
        "
        >
        Maximum
        </div>

        <strong
        style="
            color: ${maxValue == null
            ? "var(--theme-foreground)"
            : getMrlAssessment(maxValue).color};
        "
        >
        ${maxValue == null
            ? "N/A"
            : `${formatNumber(maxValue)} ${concentrationUnit(selectedChemical)}`}
        </strong>

        <div
        style="
            font-size: 12px;
            color: var(--theme-foreground-muted);
            margin-top: 3px;
        "
        >
        ${maxRow ? formatDate(maxRow.date) : "No date"}
        </div>
    </div>
    </div>
</div>

${
    mrlThresholds.length > 0
    ? html`
        <div
            style="
            display: flex;
            gap: 16px;
            flex-wrap: wrap;
            align-items: center;
            margin: 12px 0;
            font-size: 12px;
            "
        >
            ${mrlThresholds.map(
            (d) => html`
                <div
                style="
                    display: flex;
                    align-items: center;
                    gap: 6px;
                "
                >
                <span
                    style="
                    width: 22px;
                    border-top: 2px dotted ${d.color};
                    display: inline-block;
                    "
                ></span>
                <span>${d.duration} MRL</span>
                </div>
            `,
            )}
        </div>
        `
    : html``
}

${Plot.plot({
    width: 900,
    height: 155,
    marginLeft: 70,
    marginBottom: 35,
    x: {
    type: "time",
    label: "Sample date",
    },
    y: {
    label: null,
    axis: null,
    grid: true,
    },
    marks: [
    ...mrlThresholds.map((threshold) =>
        Plot.ruleY([threshold.value], {
        stroke: threshold.color,
        strokeWidth: 1.5,
        strokeDasharray: "4,4",
        strokeOpacity: 0.9,
        }),
    ),

    Plot.line(chartRows, {
        x: "date",
        y: (d) => d[selectedChemical],
        stroke: "#008fbd",
        strokeOpacity: 0.8,
    }),

    Plot.dot(chartRows, {
        x: "date",
        y: (d) => d[selectedChemical],
        r: 1.5,
        fill: "#008fbd",
    }),

    (index, scales, channels, dimensions) => {
        const x1 = dimensions.marginLeft;
        const x2 = dimensions.width - dimensions.marginRight;
        const y1 = dimensions.marginTop;
        const y2 = dimensions.height - dimensions.marginBottom;

        const g = d3.create("svg:g");

        const brush = d3
        .brushX()
        .extent([
            [x1, y1],
            [x2, y2],
        ])
        .on("end", (event) => {
            if (!event.sourceEvent) return;

            const nextRange = event.selection
            ? event.selection.map(scales.x.invert).sort((a, b) => a - b)
            : null;

            brushRangeInput.value = nextRange;
            brushRangeInput.dispatchEvent(new Event("input"));
        });

        g.call(brush);

        // Restore the current selection whenever the chart redraws.
        const initialRange = brushRange || fullDateExtent;

        if (initialRange?.[0] && initialRange?.[1]) {
        g.call(brush.move, initialRange.map(scales.x));
        }

        return g.node();
    },
    ],
})}
`);
```

```js
function formatDateRange([start, end]) {
    const format = d3.timeFormat("%b %Y");
    return `${format(start)} – ${format(end)}`;
}
```

## Screening-reference details

```js
display(
  selectedReference
    ? html`
        <div
          style="border:1px solid var(--theme-foreground-faint);border-radius:10px;padding:16px;margin:12px 0;"
        >
          <div
            style="font-size:12px;text-transform:uppercase;letter-spacing:.06em;color:var(--theme-foreground-muted);"
          >
            ${escapeHTML(selectedReference.status)} screening references
          </div>

          <h2 style="margin:8px 0;">
            ${escapeHTML(chemicalLabel(selectedChemical))}
          </h2>

          <div
            style="font-size:13px;color:var(--theme-foreground-muted);margin-bottom:16px;"
          >
            ${escapeHTML(selectedReference.title)}
          </div>

          <div
            style="display:grid;grid-template-columns:repeat(auto-fit,minmax(150px,1fr));gap:12px;"
          >
            <div
              style="border:1px solid #e53935;border-radius:8px;padding:12px;"
            >
              <div
                style="font-size:12px;text-transform:uppercase;letter-spacing:.05em;color:#e53935;font-weight:700;"
              >
                Acute · 1–14 days
              </div>
              <div
                style="font-size:28px;font-weight:700;color:#e53935;margin:8px 0;"
              >
                ${selectedReference.acute ?? "—"}
              </div>
              <div style="color:#e53935;">
                ${escapeHTML(selectedReference.unit)}
              </div>
              <p style="font-size:13px;margin:10px 0 0;">
                Screening reference for short-term inhalation exposure lasting
                1–14 days.
              </p>
            </div>

            <div
              style="border:1px solid #f28c28;border-radius:8px;padding:12px;"
            >
              <div
                style="font-size:12px;text-transform:uppercase;letter-spacing:.05em;color:#f28c28;font-weight:700;"
              >
                Intermediate · 15–364 days
              </div>
              <div
                style="font-size:28px;font-weight:700;color:#f28c28;margin:8px 0;"
              >
                ${selectedReference.intermediate ?? "—"}
              </div>
              <div style="color:#f28c28;">
                ${escapeHTML(selectedReference.unit)}
              </div>
              <p style="font-size:13px;margin:10px 0 0;">
                Screening reference for exposure lasting 15–364 days.
              </p>
            </div>

            <div
              style="border:1px solid #d4a900;border-radius:8px;padding:12px;"
            >
              <div
                style="font-size:12px;text-transform:uppercase;letter-spacing:.05em;color:#b08b00;font-weight:700;"
              >
                Chronic · ≥1 year
              </div>
              <div
                style="font-size:28px;font-weight:700;color:#b08b00;margin:8px 0;"
              >
                ${selectedReference.chronic ?? "—"}
              </div>
              <div style="color:#b08b00;">
                ${escapeHTML(selectedReference.unit)}
              </div>
              <p style="font-size:13px;margin:10px 0 0;">
                Screening reference for exposure lasting a year or longer.
              </p>
            </div>
          </div>

          <p style="font-size:13px;margin:16px 0 8px;">
            ${escapeHTML(selectedReference.description)}
          </p>

          <p style="font-size:13px;color:var(--theme-foreground-muted);">
            These MRLs are not regulatory limits or individual safety
            thresholds. Comparing a daily sample with an MRL does not establish
            that a person experienced harmful effects; the sampling period and
            exposure assumptions also matter.
          </p>

          <div style="margin-top:12px;">
            <a
              href="${selectedReference.source ?? mrlSource}"
              target="_blank"
              rel="noopener noreferrer"
            >
              ATSDR MRL table
            </a>
          </div>
        </div>
      `
    : html`
        <p>
          No verified inhalation MRLs are configured for
          ${escapeHTML(chemicalLabel(selectedChemical))}. Missing values should
          remain unassigned until verified.
        </p>
      `,
);
```

## Data sources and interpretation

- **VOC measurements:** Maine DEP monitoring data. <https://www.maine.gov/dep/air/monitoring/spo-voc-monitor.html>
- **Benzene screening reference:** ATSDR Minimal Risk Levels. <https://wwwn.cdc.gov/TSP/MRLS/mrlslisting.aspx>
- **Map basemap:** © OpenStreetMap contributors.

The chart represents the observations in the supplied CSV; it does not imply continuous monitoring between sample dates. The chronic benzene MRL is intended for chronic-exposure assessment and should not be interpreted as a threshold for individual 24-hour samples. Missing readings and missing wind observations are displayed as unavailable rather than filled with estimates.
