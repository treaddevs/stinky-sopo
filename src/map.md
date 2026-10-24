---
toc: false
---

# Portland Harbor VOC Monitoring

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

// MRL values in ppbv.
// Acute: 1–14 days; intermediate: 15–364 days;
// chronic: 1 year or longer.
// null = no applicable value configured.
// Aggregate categories are not individual chemical MRLs.
const screeningReferences = {
  // Aliphatics
  "Aliphatics+Aromatics": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Not applicable",
    title: "Combined aliphatics and aromatics",
    description: "Aggregate category; no individual-compound MRL.",
    source: mrlSource,
  },
  "Aliphatics sum total": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Not applicable",
    title: "Aliphatics sum total",
    description: "Aggregate category; no individual-compound MRL.",
    source: mrlSource,
  },
  "1-Pentene": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — 1-pentene",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "2,3-Dimethylbutane": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — 2,3-dimethylbutane",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "2,3-Dimethylpentane": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — 2,3-dimethylpentane",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  Isopentane: {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — isopentane",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "n-Butane": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — n-butane",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "n-Pentane": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — n-pentane",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  Propane: {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — propane",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  Propylene: {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — propylene",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },

  // Aromatics
  "Aromatics sum total": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Not applicable",
    title: "Aromatics sum total",
    description: "Aggregate category; no individual-compound MRL.",
    source: mrlSource,
  },
  "1,2,4-Trimethylbenzene": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — 1,2,4-trimethylbenzene",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "1,3,5-Trimethylbenzene": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — 1,3,5-trimethylbenzene",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  Benzene: {
    acute: 9,
    intermediate: 7,
    chronic: 2,
    unit: "ppbv",
    status: "Draft",
    title: "ATSDR inhalation MRL — benzene",
    description:
      "Draft inhalation MRLs; non-cancer health-based screening references.",
    source: mrlSource,
  },
  Ethylbenzene: {
    acute: 5000,
    intermediate: 2000,
    chronic: 60,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — ethylbenzene",
    description: "Health-based inhalation screening references.",
    source: mrlSource,
  },
  "m,p-Xylenes": {
    acute: 2000,
    intermediate: 400,
    chronic: 10,
    unit: "ppbv",
    status: "Draft",
    title: "ATSDR inhalation MRL — mixed xylenes",
    description:
      "Mixed-xylene references; not isomer-specific. Confirm applicability.",
    source: mrlSource,
    note: "ATSDR values are for mixed xylenes.",
  },
  Naphthalene: {
    acute: 0.06,
    intermediate: null,
    chronic: null,
    unit: "ppb",
    status: "Final",
    title: "ATSDR inhalation MRL — naphthalene",
    description:
      "Retained from the existing configuration; verify units and duration against the current source before comparison.",
    source: mrlSource,
  },
  "o-Xylene": {
    acute: 2000,
    intermediate: 400,
    chronic: 10,
    unit: "ppbv",
    status: "Draft",
    title: "ATSDR inhalation MRL — mixed xylenes",
    description:
      "Mixed-xylene references, not an o-xylene-specific MRL. Confirm applicability.",
    source: mrlSource,
    note: "ATSDR values are for mixed xylenes.",
  },
  Toluene: {
    acute: 2000,
    intermediate: null,
    chronic: 1000,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — toluene",
    description: "Intermediate-duration value is not configured.",
    source: mrlSource,
  },

  // Chlorinated solvents
  "Chlorinated Solvents sum total": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Not applicable",
    title: "Chlorinated solvents sum total",
    description: "Aggregate category; no individual-compound MRL.",
    source: mrlSource,
  },
  "1,1,1-Trichloroethane": {
    acute: 1000,
    intermediate: 700,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — 1,1,1-trichloroethane",
    description: "Chronic-duration value is not configured.",
    source: mrlSource,
  },
  "1,1,2,2-Tetrachloroethane": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — 1,1,2,2-tetrachloroethane",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "1,1,2-Trichloroethane": {
    acute: 30,
    intermediate: 2,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — 1,1,2-trichloroethane",
    description: "No chronic-duration value configured.",
    source: mrlSource,
  },
  "1,1-Dichloroethene": {
    acute: null,
    intermediate: 1,
    chronic: 1,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — 1,1-dichloroethene",
    description: "No acute-duration value configured.",
    source: mrlSource,
  },
  "1,2-Dichlorobenzene": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — 1,2-dichlorobenzene",
    description:
      "Existing oral MRLs should not be used as inhalation thresholds.",
    source: mrlSource,
  },
  "1,3-Dichlorobenzene": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — 1,3-dichlorobenzene",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "1,4-Dichlorobenzene": {
    acute: 2000,
    intermediate: 200,
    chronic: 10,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — 1,4-dichlorobenzene",
    description:
      "Verify route-specific values against the current ATSDR listing.",
    source: mrlSource,
  },
  "Carbon Tetrachloride": {
    acute: null,
    intermediate: 30,
    chronic: 30,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — carbon tetrachloride",
    description: "No acute-duration inhalation MRL configured.",
    source: mrlSource,
  },
  Chlorobenzene: {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — chlorobenzene",
    description: "Do not substitute oral MRLs for inhalation references.",
    source: mrlSource,
  },
  Chloroform: {
    acute: 1,
    intermediate: 0.8,
    chronic: 0.4,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — chloroform",
    description: "Health-based inhalation screening references.",
    source: mrlSource,
  },
  "cis-1,2-Dichloroethene": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — cis-1,2-dichloroethene",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "cis-1,3-Dichloropropene": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — cis-1,3-dichloropropene",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "Ethylene Dichloride": {
    acute: 100,
    intermediate: 100,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — 1,2-dichloroethane",
    description:
      "Also known as ethylene dichloride. No chronic value configured.",
    source: mrlSource,
  },
  "Ethylidene Dichloride": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — 1,1-dichloroethane",
    description:
      "Also known as ethylidene dichloride. Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "Methylene Chloride": {
    acute: 600,
    intermediate: 300,
    chronic: 300,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — methylene chloride",
    description: "Also known as dichloromethane.",
    source: mrlSource,
  },
  Tetrachloroethylene: {
    acute: 6,
    intermediate: 6,
    chronic: 6,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — tetrachloroethylene",
    description: "Also known as perchloroethylene or PCE.",
    source: mrlSource,
  },
  "trans-1,2-Dichloroethene": {
    acute: 3000,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — trans-1,2-dichloroethene",
    description: "No intermediate or chronic value configured.",
    source: mrlSource,
  },
  Trichloroethylene: {
    acute: null,
    intermediate: 0.4,
    chronic: 0.4,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — trichloroethylene",
    description: "Also known as TCE. No acute-duration value configured.",
    source: mrlSource,
  },
  "Vinyl Chloride": {
    acute: 500,
    intermediate: 20,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — vinyl chloride",
    description: "No chronic inhalation value configured.",
    source: mrlSource,
  },

  // Combustion
  "Combustion sum total": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Not applicable",
    title: "Combustion sum total",
    description: "Aggregate category; no individual-compound MRL.",
    source: mrlSource,
  },
  "1,3-Butadiene": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — 1,3-butadiene",
    description: "Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  Acrolein: {
    acute: 3,
    intermediate: 0.4,
    chronic: 0.4,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — acrolein",
    description: "Health-based inhalation screening references.",
    source: mrlSource,
  },

  // Coolants and refrigerants
  "Coolants sum total": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Not applicable",
    title: "Coolants sum total",
    description: "Aggregate category; no individual-compound MRL.",
    source: mrlSource,
  },
  Dichlorodifluoromethane: {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — dichlorodifluoromethane",
    description:
      "Also known as CFC-12 or Freon 12. Applicable MRLs not verified.",
    source: mrlSource,
  },
  "Ethyl Chloride": {
    acute: 13000,
    intermediate: 13000,
    chronic: null,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — ethyl chloride",
    description: "Also known as chloroethane. No chronic value configured.",
    source: mrlSource,
  },
  "Freon 113": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — Freon 113",
    description:
      "Also known as 1,1,2-trichloro-1,2,2-trifluoroethane. Applicable MRLs not verified.",
    source: mrlSource,
  },
  "Freon 114": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — Freon 114",
    description:
      "Also known as dichlorotetrafluoroethane. Applicable MRLs not verified.",
    source: mrlSource,
  },
  "Methyl Chloride": {
    acute: 500,
    intermediate: 300,
    chronic: 30,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — methyl chloride",
    description: "Also known as chloromethane.",
    source: mrlSource,
  },
  Trichlorofluoromethane: {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — trichlorofluoromethane",
    description:
      "Also known as CFC-11 or Freon 11. Applicable MRLs not verified.",
    source: mrlSource,
  },

  // Fumigants
  "Fumigants sum total": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Not applicable",
    title: "Fumigants sum total",
    description: "Aggregate category; no individual-compound MRL.",
    source: mrlSource,
  },
  "Ethylene Dibromide": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — ethylene dibromide",
    description:
      "Also known as 1,2-dibromoethane. Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "Methyl Bromide": {
    acute: null,
    intermediate: 20,
    chronic: 1,
    unit: "ppbv",
    status: "Final",
    title: "ATSDR inhalation MRL — methyl bromide",
    description: "Also known as bromomethane. No acute value configured.",
    source: mrlSource,
  },

  // Solvents
  "Solvents sum total": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Not applicable",
    title: "Solvents sum total",
    description: "Aggregate category; no individual-compound MRL.",
    source: mrlSource,
  },
  "Methyl Ethyl Ketone": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — methyl ethyl ketone",
    description:
      "Also known as 2-butanone. Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
  "Methyl Isobutyl Ketone": {
    acute: null,
    intermediate: null,
    chronic: null,
    unit: "ppbv",
    status: "Unverified",
    title: "ATSDR inhalation MRL — methyl isobutyl ketone",
    description: "Also known as MIBK. Applicable inhalation MRLs not verified.",
    source: mrlSource,
  },
};

const selectedReference = screeningReferences[selectedChemical] ?? null;

function getScreeningReference(column) {
  const key = normalizeKey(chemicalLabel(column));
  const reference = screeningReferences[key] ?? screeningReferences[column];

  if (!reference) return null;

  const referenceUnit = normalizeConcentrationUnit(reference.unit);
  const selectedUnit = normalizeConcentrationUnit(concentrationUnit(column));

  return referenceUnit === selectedUnit ? reference : null;
}

const selectedScreeningReference = getScreeningReference(selectedChemical);

const mrlExceedanceCounts = Object.fromEntries(
  ["acute", "intermediate", "chronic"].map((duration) => {
    const threshold = selectedScreeningReference?.[duration];

    if (threshold == null || !Number.isFinite(threshold)) {
      return [duration, null];
    }

    return [
      duration,
      rangeRows.filter((row) => {
        const rawValue = row[selectedChemical];
        const value = Number(rawValue);

        return (
          rawValue != null &&
          rawValue !== "" &&
          Number.isFinite(value) &&
          value >= threshold
        );
      }).length,
    ];
  }),
);
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
    <div
      class="map-legend"
      style="
    position: absolute;
    bottom: 16px;
    left: 16px;
    z-index: 2;
    padding: 12px 14px;
    background: rgba(255, 255, 255, 0.95);
    border: 1px solid #d1d5db;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);
    font-family: system-ui, sans-serif;
    font-size: 12px;
    color: #222;
  "
    >
      <div
        style="
      font-weight: 700;
      font-size: 13px;
      margin-bottom: 9px;
    "
      >
        Map key
      </div>

      <div
        style="
      display: flex;
      align-items: center;
      gap: 9px;
      margin-bottom: 7px;
    "
      >
        <span
          style="
        display: inline-block;
        width: 11px;
        height: 11px;
        flex-shrink: 0;
        border-radius: 50%;
        background: #315d78;
        border: 2px solid #fff;
        outline: 1px solid #315d78;
      "
        ></span>
        Monitoring station
      </div>

      <div
        style="
      display: flex;
      align-items: center;
      gap: 9px;
    "
      >
        <span
          style="
        display: inline-block;
        width: 11px;
        height: 11px;
        flex-shrink: 0;
        border-radius: 50%;
        background: #f2c94c;
        border: 2px solid #fff;
        outline: 1px solid #d1aa24;
      "
        ></span>
        Petroleum terminal
      </div>
    </div>
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

const rangeRows = chartRows.filter((row) => {
  if (!brushRange) return true;

  const [start, end] = brushRange;

  return row.date >= start && row.date <= end;
});
```

```js
display(html`
  <h2>
    ${brushRange
      ? `Reports: ${formatDateRange(brushRange)}`
      : "Reports: Full timeline"}
  </h2>

  ${(() => {
    const mrlReference = screeningReferences[selectedChemical];

    const normalizeUnit = (unit) => {
      const normalized = unit?.trim().toLowerCase();
      return normalized === "ppb" || normalized === "ppbv"
        ? "ppbv"
        : normalized;
    };

    // Get thresholds that have valid values and compatible units.
    const availableThresholds = [
      {
        duration: "Acute",
        value: mrlReference?.acute,
        color: "#e53935",
      },
      {
        duration: "Intermediate",
        value: mrlReference?.intermediate,
        color: "#f28c28",
      },
      {
        duration: "Chronic",
        value: mrlReference?.chronic,
        color: "#d4a900",
      },
    ].filter(
      (d) =>
        d.value != null &&
        Number.isFinite(d.value) &&
        normalizeUnit(mrlReference?.unit) ===
          normalizeUnit(concentrationUnit(selectedChemical)),
    );

    // Find the highest valid measurement in the displayed date range.
    const dataValues = rangeRows
      .map((d) => Number(d[selectedChemical]))
      .filter(Number.isFinite);

    const maxDataValue = dataValues.length ? Math.max(...dataValues) : null;

    // If any reading exceeds a threshold, show all available thresholds.
    const hasExceededThreshold =
      maxDataValue != null &&
      availableThresholds.some((threshold) => maxDataValue > threshold.value);

    // Otherwise, show only the threshold closest to the maximum reading.
    const mrlThresholds = hasExceededThreshold
      ? availableThresholds
      : maxDataValue != null && availableThresholds.length
        ? [
            availableThresholds.reduce((closest, current) =>
              Math.abs(current.value - maxDataValue) <
              Math.abs(closest.value - maxDataValue)
                ? current
                : closest,
            ),
          ]
        : [];

    return Plot.plot({
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
        // Dotted MRL reference lines.
        ...mrlThresholds.map((threshold) =>
          Plot.ruleY([threshold.value], {
            stroke: threshold.color,
            strokeWidth: 1.5,
            strokeDasharray: "4,4",
            strokeOpacity: 0.9,
          }),
        ),

        // Concentration trend.
        Plot.line(rangeRows, {
          x: "date",
          y: (d) => d[selectedChemical],
          stroke: "#008fbd",
          strokeOpacity: 0.6,
        }),

        // Measurements colored by their MRL assessment.
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

        // Zero baseline.
        Plot.ruleY([0], {
          stroke: "#888",
          strokeOpacity: 0.4,
        }),
      ],
    });
  })()}
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
                color: ${getMrlAssessment(selectedReport[selectedChemical])
              .color};
              "
          >
            ${formatNumber(selectedReport[selectedChemical])}
            ${concentrationUnit(selectedChemical)}
          </div>

          <div
            style="
                margin-top: 5px;
                font-size: 13px;
                color: ${getMrlAssessment(selectedReport[selectedChemical])
              .color};
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

          <div style="margin-top: 8px;">
            <table
              style="
                width: 100%;
                border-collapse: collapse;
                font-size: 13px;
                text-align: left;
                "
            >
              <thead>
                <tr>
                  <th style="padding: 6px 8px; border-bottom: 1px solid #888;">
                    Compass
                  </th>
                  <th style="padding: 6px 8px; border-bottom: 1px solid #888;">
                    Bearing
                  </th>
                  <th style="padding: 6px 8px; border-bottom: 1px solid #888;">
                    Gust
                  </th>
                </tr>
              </thead>
              <tbody>
                <tr>
                  <td style="padding: 6px 8px;">
                    ${selectedReport.windDirection ??
                    selectedReport.raw?.compass ??
                    "—"}
                  </td>
                  <td style="padding: 6px 8px;">
                    ${(selectedReport.windBearing ??
                      selectedReport.raw?.bearing) != null
                      ? `${selectedReport.windBearing ?? selectedReport.raw?.bearing}°`
                      : "—"}
                  </td>
                  <td style="padding: 6px 8px;">
                    ${selectedReport.windGust ??
                    selectedReport.raw?.gust ??
                    "—"}
                  </td>
                </tr>
              </tbody>
            </table>
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
  {
    duration: "Intermediate",
    value: mrlReference?.intermediate,
    color: "#f28c28",
  },
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
    margin-bottom: 6px;
  "
        >
          MRL exceedances
        </div>

        <div style="display: flex; gap: 14px; flex-wrap: wrap;">
          ${[
            { label: "Acute", key: "acute", color: "#e53935" },
            { label: "Intermediate", key: "intermediate", color: "#f28c28" },
            { label: "Chronic", key: "chronic", color: "#b08b00" },
          ].map(
            (item) => html`
              <div>
                <div
                  style="
          font-size: 11px;
          color: var(--theme-foreground-muted);
        "
                >
                  ${item.label}
                </div>
                <strong
                  style="
          color: ${item.color};
          font-variant-numeric: tabular-nums;
        "
                >
                  ${mrlExceedanceCounts[item.key] == null
                    ? "N/A"
                    : mrlExceedanceCounts[item.key].toLocaleString()}
                </strong>
              </div>
            `,
          )}
        </div>
      </div>

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

  ${mrlThresholds.length > 0
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
    : html``}
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

```js
const vocExceedanceCounts = (() => {
  const durations = [
    { key: "acute", label: "Acute", period: "1–14 days", color: "#e53935" },
    {
      key: "intermediate",
      label: "Intermediate",
      period: "15–364 days",
      color: "#f28c28",
    },
    { key: "chronic", label: "Chronic", period: "≥1 year", color: "#b08b00" },
  ];

  const compounds = Object.entries(screeningReferences);

  return durations.map((duration) => {
    let count = 0;
    const breakdown = [];

    for (const [chemical, reference] of compounds) {
      const threshold = reference[duration.key];

      if (!Number.isFinite(threshold)) continue;

      let compoundCount = 0;

      for (const row of normalizedRows) {
        const value = Number(row[chemical]);

        if (
          row[chemical] != null &&
          row[chemical] !== "" &&
          Number.isFinite(value) &&
          value > threshold
        ) {
          compoundCount++;
        }
      }

      if (compoundCount > 0) {
        count += compoundCount;
        breakdown.push({
          chemical,
          count: compoundCount,
        });
      }
    }

    breakdown.sort((a, b) => b.count - a.count);

    return {
      ...duration,
      count,
      compoundsExceeded: breakdown.length,
      breakdown,
    };
  });
})();

const totalExceedances = vocExceedanceCounts.reduce(
  (sum, item) => sum + item.count,
  0,
);

display(html`
  <section
    style="
    margin: 20px 0;
    border: 1px solid var(--theme-foreground-faint);
    border-radius: 12px;
    overflow: hidden;
  "
  >
    <header
      style="
      padding: 18px 20px;
      border-bottom: 1px solid var(--theme-foreground-faint);
      display: flex;
      justify-content: space-between;
      align-items: flex-start;
      gap: 16px;
      flex-wrap: wrap;
    "
    >
      <div>
        <div
          style="
          font-size: 11px;
          font-weight: 700;
          letter-spacing: .1em;
          text-transform: uppercase;
          color: var(--theme-foreground-muted);
        "
        >
          Screening summary
        </div>
        <h2 style="font-size: 19px; margin: 6px 0 4px;">
          VOC threshold exceedances
        </h2>
        <div style="font-size: 13px; color: var(--theme-foreground-muted);">
          Breakdown by compound and screening duration
        </div>
      </div>

      <div style="text-align: right; min-width: 100px;">
        <div
          style="
          font-size: 30px;
          line-height: 1.1;
          font-weight: 700;
          font-variant-numeric: tabular-nums;
        "
        >
          ${totalExceedances.toLocaleString()}
        </div>
        <div
          style="
          margin-top: 5px;
          font-size: 11px;
          color: var(--theme-foreground-muted);
        "
        >
          Total exceedances*
        </div>
      </div>
    </header>

    <div style="padding: 4px 20px;">
      ${vocExceedanceCounts.map(
        (item, index) => html`
          <section
            style="
          padding: 16px 0;
          ${index < vocExceedanceCounts.length - 1
              ? "border-bottom: 1px solid var(--theme-foreground-faint);"
              : ""}
        "
          >
            <div
              style="
            display: flex;
            justify-content: space-between;
            align-items: center;
            gap: 12px;
            margin-bottom: 12px;
          "
            >
              <div style="display: flex; align-items: center; gap: 10px;">
                <div
                  style="
                width: 4px;
                height: 38px;
                border-radius: 4px;
                background: ${item.color};
              "
                ></div>
                <div>
                  <div style="font-size: 15px; font-weight: 650;">
                    ${item.label}
                  </div>
                  <div
                    style="
                  margin-top: 3px;
                  font-size: 12px;
                  color: var(--theme-foreground-muted);
                "
                  >
                    ${item.period} · ${item.compoundsExceeded} VOCs with
                    exceedances
                  </div>
                </div>
              </div>
              <div style="text-align: right;">
                <div
                  style="
                font-size: 23px;
                font-weight: 650;
                color: ${item.color};
                font-variant-numeric: tabular-nums;
              "
                >
                  ${item.count.toLocaleString()}
                </div>
                <div
                  style="
                font-size: 11px;
                color: var(--theme-foreground-muted);
              "
                >
                  measurements
                </div>
              </div>
            </div>

            ${item.breakdown.length
              ? html`
                  <div
                    style="
                border: 1px solid var(--theme-foreground-faint);
                border-radius: 8px;
                overflow: hidden;
              "
                  >
                    <div
                      style="
                  display: grid;
                  grid-template-columns: minmax(0, 1fr) auto;
                  gap: 12px;
                  padding: 9px 12px;
                  font-size: 11px;
                  font-weight: 700;
                  color: var(--theme-foreground-muted);
                  background: var(--theme-foreground-faintest, transparent);
                "
                    >
                      <span>VOC compound</span>
                      <span style="text-align: right;">Exceedances</span>
                    </div>
                    ${item.breakdown.map(
                      (compound, i) => html`
                        <div
                          style="
                    display: grid;
                    grid-template-columns: minmax(0, 1fr) auto;
                    gap: 12px;
                    align-items: center;
                    padding: 10px 12px;
                    font-size: 13px;
                    ${i < item.breakdown.length - 1
                            ? "border-bottom: 1px solid var(--theme-foreground-faint);"
                            : ""}
                  "
                        >
                          <span style="overflow-wrap: anywhere;">
                            ${compound.chemical}
                          </span>
                          <strong
                            style="
                      font-variant-numeric: tabular-nums;
                      color: ${item.color};
                    "
                            >${compound.count.toLocaleString()}</strong
                          >
                        </div>
                      `,
                    )}
                  </div>
                `
              : html`
                  <div
                    style="
                padding: 12px;
                border-radius: 8px;
                background: var(--theme-foreground-faintest, transparent);
                color: var(--theme-foreground-muted);
                font-size: 13px;
              "
                  >
                    No measurements exceeded configured references in this
                    category.
                  </div>
                `}
          </section>
        `,
      )}
    </div>

    <footer
      style="
      padding: 14px 20px;
      background: var(--theme-foreground-faintest, transparent);
      border-top: 1px solid var(--theme-foreground-faint);
      font-size: 12px;
      line-height: 1.6;
      color: var(--theme-foreground-muted);
    "
    >
      <strong style="color: var(--theme-foreground);">How to read this</strong>
      <p style="margin: 5px 0 0;">
        Each compound count represents the number of recorded measurements above
        that compound's configured screening reference. Compounds are ranked
        from most to fewest exceedances within each category. These counts do
        not represent unique samples, people affected, health cases, or
        confirmed harmful exposures.
      </p>
      <p style="margin: 6px 0 0;">
        *The total sums compound-level exceedances across all three duration
        categories. A measurement can contribute to multiple categories if it
        exceeds multiple references. Comparisons require compatible units.
      </p>
    </footer>
  </section>
`);
```

## Data sources and interpretation

- **VOC measurements:** Maine DEP monitoring data. <https://www.maine.gov/dep/air/monitoring/spo-voc-monitor.html>
- **Minimal Risk Levels (MRLs) for Hazardous Substances:** ATSDR Agency for Toxic Sustances and Disease Registry. <https://wwwn.cdc.gov/TSP/MRLS/mrlslisting.aspx>
- **Map basemap:** © OpenStreetMap contributors.

The chart reflects observations recorded in the supplied CSV and does not imply continuous monitoring between sample dates. Minimal Risk Levels (MRLs) are intended to support exposure assessment, not to serve as thresholds for individual 24-hour samples. Missing concentration readings and wind observations are shown as unavailable rather than estimated.
