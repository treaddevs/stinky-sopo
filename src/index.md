---
toc: false
---

# Residents are noticing petroleum odors beyond the monitoring fence line

South Portland's smell reports show where residents are experiencing
petroleum-related odors across the community. These reports provide a
community-level signal that fixed air monitors cannot capture on their own.

## More than half of the reports contain petroleum-related language

```js
const [rangeStart, rangeEnd] = startEnd;

const reportsInRange = dataWithPetroleum.filter(
  (d) => rangeStart <= d.datetime && d.datetime < rangeEnd,
);

const petroleumCount = reportsInRange.filter((d) => d.petroleumRelated).length;

const otherCount = reportsInRange.length - petroleumCount;
const totalReports = reportsInRange.length;

const reportCategoryCounts = [
  {
    category: "Petroleum-related",
    count: petroleumCount,
    percentage: totalReports ? petroleumCount / totalReports : 0,
    label: `${petroleumCount} (${d3.format(".0%")(
      totalReports ? petroleumCount / totalReports : 0,
    )})`,
  },
  {
    category: "Other",
    count: otherCount,
    percentage: totalReports ? otherCount / totalReports : 0,
    label: `${otherCount} (${d3.format(".0%")(
      totalReports ? otherCount / totalReports : 0,
    )})`,
  },
];

display(html`
  <div style="margin-bottom: 8px; font-size: 14px;">
    <strong>Selected range:</strong>
    ${formatMapDateRange(startEnd)}
    <span style="color: #888;"> · ${d3.format(",")(totalReports)} reports</span>
  </div>
  ${Plot.plot({
    height: 350,
    y: {
      grid: true,
      label: "Number of reports",
      inset: 10,
      domain: [
        0,
        Math.max(1, d3.max(reportCategoryCounts, (d) => d.count) * 1.12),
      ],
    },
    x: {
      label: null,
      domain: ["Petroleum-related", "Other"],
    },
    color: {
      domain: ["Petroleum-related", "Other"],
      range: ["#d97706", "#888888"],
    },
    marks: [
      Plot.barY(reportCategoryCounts, {
        x: "category",
        y: "count",
        fill: "category",
      }),
      Plot.text(reportCategoryCounts, {
        x: "category",
        y: "count",
        text: "label",
        dy: -10,
        fontWeight: "bold",
      }),
    ],
  })}
`);
```

```js
display(
  Plot.plot({
    height: 180,
    y: {
      grid: true,
      label: "Monthly reports",
      zero: true,
    },
    x: {
      type: "utc",
      label: "Drag the selection to choose a date range",
    },
    marks: [
      Plot.areaY(mapTimeline, {
        x: "month",
        y: "count",
        fill: "#d97706",
        fillOpacity: 0.18,
      }),
      Plot.lineY(mapTimeline, {
        x: "month",
        y: "count",
        stroke: "#d97706",
      }),
      (index, scales, channels, dimensions) => {
        const x1 = dimensions.marginLeft;
        const x2 = dimensions.width - dimensions.marginRight;
        const y1 = dimensions.marginTop;
        const y2 = dimensions.height - dimensions.marginBottom;

        const brush = d3
          .brushX()
          .extent([
            [x1, y1],
            [x2, y2],
          ])
          .on("brush end", (event) => {
            if (!event.sourceEvent || !event.selection) return;

            const range = event.selection.map(scales.x.invert);
            setStartEnd(range);
          });

        const g = d3.create("svg:g").call(brush);

        // Start at the earliest month in the dataset.
        g.call(brush.move, defaultStartEnd.map(scales.x));

        return g.node();
      },
    ],
  }),
);
```

As of August 2026, the majority of Smell MyCity reports in the Portland, Maine area contain at least one petroleum-related keyword which includes: **_oil, petrol, petro, tar, petroleum, gasoline, fuel, diesel, asphalt, oily, petrochemical, crude, asphalty_**

Petroleum-related language appears in 56% of the submitted reports.

## Exploring odor reports trends from Smell MyCity data

```js
import { tile } from "npm:d3-tile";

const data = await FileAttachment("data/stinky-reports.csv").csv({
  typed: true,
});

const petroleumKeywords = [
  "oil",
  "oily",
  "petroleum",
  "petrol",
  "petrochemical",
  "petro",
  "gasoline",
  "diesel",
  "fuel",
  "crude",
  "asphalt",
  "asphalty",
  "tar",
];

const dataWithPetroleum = data.map((d) => {
  const text = `${d.description ?? ""} ${d.comments ?? ""}`.toLowerCase();

  return {
    ...d,
    datetime: new Date(d.datetime),
    petroleumRelated: petroleumKeywords.some((keyword) =>
      text.includes(keyword),
    ),
  };
});
```

```js
const reportType = view(
  Inputs.select(["All reports", "Petroleum-related", "Other"], {
    label: "Show:",
    value: "All reports",
  }),
);
```

```js
const filteredReports = dataWithPetroleum.filter((d) => {
  const inDateRange = startEnd[0] <= d.datetime && d.datetime < startEnd[1];

  const matchesType =
    reportType === "All reports" ||
    (reportType === "Petroleum-related"
      ? d.petroleumRelated
      : !d.petroleumRelated);

  return inDateRange && matchesType;
});
```

```js
function basemapUrl(z, x, y) {
  return `https://services.arcgisonline.com/ArcGIS/rest/services/Canvas/World_Light_Gray_Base/MapServer/tile/${z}/${y}/${x}`;
}

// function basemapUrl(z, x, y) {
//   return `https://tile.openstreetmap.org/${z}/${x}/${y}.png`;
// }

function renderBasemap(
  projection,
  width,
  height,
  zoomTransform = d3.zoomIdentity,
) {
  const effectiveScale = projection.scale() * 2 * Math.PI * zoomTransform.k;

  const effectiveTranslate = zoomTransform.apply(projection([0, 0]));

  const tiles = tile()
    .extent([
      [0, 0],
      [width, height],
    ])
    .scale(effectiveScale)
    .translate(effectiveTranslate)();

  const [tx, ty] = tiles.translate;
  const k = tiles.scale;

  const clipId = `basemap-${Math.random().toString(36).slice(2)}`;

  const g = d3
    .create("svg:g")
    .attr("class", "basemap-layer")
    .attr("clip-path", `url(#${clipId})`);

  g.append("clipPath")
    .attr("id", clipId)
    .append("rect")
    .attr("width", width)
    .attr("height", height);

  g.selectAll("image")
    .data(tiles)
    .join("image")
    .attr("href", ([x, y, z]) => basemapUrl(z, x, y))
    .attr("x", ([x]) => Math.round((x + tx) * k))
    .attr("y", ([, y]) => Math.round((y + ty) * k))
    .attr("width", k)
    .attr("height", k);

  return g.node();
}

function basemap(projection, width, height) {
  return () => renderBasemap(projection, width, height);
}
```

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
```

```js
display(
  resize((width) => {
    const height = 600;

    const extent = {
      type: "MultiPoint",
      coordinates: [
        [-70.34, 43.564],
        [-70.19, 43.723],
      ],
    };

    const projection = d3.geoMercator().fitExtent(
      [
        [0, 0],
        [width, height],
      ],
      extent,
    );

    // -------------------------
    // Create the map
    // -------------------------

    const plot = Plot.plot({
      width,
      height,
      margin: 0,
      projection: () => projection,

      marks: [
        basemap(projection, width, height),

        // Odor reports
        Plot.dot(filteredReports, {
          x: "longitude",
          y: "latitude",
          r: 4,
          fill: (d) => (d.petroleumRelated ? "#d97706" : "#888888"),
          stroke: "white",
          strokeWidth: 0.75,
          className: "report-dot",
        }),

        // Petroleum terminals
        Plot.dot(terminals, {
          x: "longitude",
          y: "latitude",
          r: 8,
          fill: "#facc15",
          stroke: "white",
          strokeWidth: 2,
          className: "terminal-dot",
        }),
      ],
    });

    // -------------------------
    // Zoom and pan
    // -------------------------

    const svg = d3.select(plot);

    const mapGroups = svg.selectAll(":scope > g").filter(function () {
      return !d3.select(this).classed("basemap-layer");
    });

    const zoom = d3
      .zoom()
      .scaleExtent([1, 8])
      .extent([
        [0, 0],
        [width, height],
      ])
      .on("zoom", (event) => {
        const transform = event.transform;

        const oldBasemap = svg.select(".basemap-layer").node();

        if (oldBasemap) {
          const newBasemap = renderBasemap(
            projection,
            width,
            height,
            transform,
          );

          oldBasemap.replaceWith(newBasemap);
        }

        mapGroups.attr("transform", transform);

        mapGroups
          .selectAll("circle")
          .attr("r", 6 / transform.k)
          .attr("stroke-width", 1 / transform.k);
      });

    svg.call(zoom);
    svg.call(zoom.scaleBy, 2);

    // -------------------------
    // Zoom controls
    // -------------------------

    const controls = document.createElement("div");

    controls.style.position = "absolute";
    controls.style.top = "10px";
    controls.style.left = "10px";
    controls.style.zIndex = "10";
    controls.style.display = "flex";
    controls.style.flexDirection = "column";
    controls.style.gap = "2px";

    const createZoomButton = (label, title) => {
      const button = document.createElement("button");

      button.textContent = label;
      button.title = title;

      button.style.width = "32px";
      button.style.height = "32px";
      button.style.border = "1px solid #aaa";
      button.style.background = "black";
      button.style.fontSize = "20px";
      button.style.fontWeight = "bold";
      button.style.cursor = "pointer";
      button.style.padding = "0";

      button.addEventListener("mouseenter", () => {
        button.style.background = "darkgrey";
      });

      button.addEventListener("mouseleave", () => {
        button.style.background = "black";
      });

      return button;
    };

    const zoomIn = createZoomButton("+", "Zoom in");
    const zoomOut = createZoomButton("−", "Zoom out");

    zoomIn.addEventListener("click", () => {
      svg.transition().duration(250).call(zoom.scaleBy, 1.5);
    });

    zoomOut.addEventListener("click", () => {
      svg
        .transition()
        .duration(250)
        .call(zoom.scaleBy, 1 / 1.5);
    });

    controls.appendChild(zoomIn);
    controls.appendChild(zoomOut);

    // -------------------------
    // Legend
    // -------------------------

    const legend = document.createElement("div");

    legend.style.position = "absolute";
    legend.style.top = "10px";
    legend.style.right = "10px";
    legend.style.background = "white";
    legend.style.color = "black";
    legend.style.padding = "10px 12px";
    legend.style.borderRadius = "4px";
    legend.style.boxShadow = "0 1px 4px rgba(0,0,0,0.2)";
    legend.style.fontFamily = "sans-serif";
    legend.style.fontSize = "13px";
    legend.style.zIndex = "10";

    legend.innerHTML = `
      <strong>Map legend</strong>

      <div style="margin-top:8px;">
        <span style="
          display:inline-block;
          width:10px;
          height:10px;
          border-radius:50%;
          background:#d97706;
          margin-right:6px;
        "></span>
        Petroleum-related report
      </div>

      <div style="margin-top:5px;">
        <span style="
          display:inline-block;
          width:10px;
          height:10px;
          border-radius:50%;
          background:#888888;
          margin-right:6px;
        "></span>
        Other odor report
      </div>

      <div style="margin-top:5px;">
        <span style="
          display:inline-block;
          width:10px;
          height:10px;
          border-radius:50%;
          background:#facc15;
          margin-right:6px;
          vertical-align:middle;
        "></span>
        Petroleum terminal
      </div>
    `;

    // -------------------------
    // Popup
    // -------------------------

    const popup = document.createElement("div");

    popup.style.position = "absolute";
    popup.style.display = "none";
    popup.style.background = "white";
    popup.style.color = "black";
    popup.style.border = "1px solid #ccc";
    popup.style.borderRadius = "6px";
    popup.style.padding = "12px";
    popup.style.width = "280px";
    popup.style.maxWidth = "calc(100% - 24px)";
    popup.style.boxShadow = "0 2px 8px rgba(0,0,0,0.2)";
    popup.style.fontFamily = "sans-serif";
    popup.style.fontSize = "13px";
    popup.style.lineHeight = "1.4";
    popup.style.zIndex = "20";

    // Close button
    const closeButton = document.createElement("button");

    closeButton.textContent = "×";

    closeButton.style.position = "absolute";
    closeButton.style.top = "4px";
    closeButton.style.right = "7px";
    closeButton.style.border = "none";
    closeButton.style.color = "black";
    closeButton.style.background = "none";
    closeButton.style.fontSize = "20px";
    closeButton.style.cursor = "pointer";

    closeButton.addEventListener("click", () => {
      popup.style.display = "none";
    });

    popup.appendChild(closeButton);

    // -------------------------
    // Helper for popup text
    // -------------------------

    const formatDate = (value) => {
      if (!value) return "Unknown";

      const date = new Date(value);

      if (Number.isNaN(date.getTime())) {
        return String(value);
      }

      return d3.timeFormat("%a %b %d")(date);
    };

    // Click terminal markers
    svg.selectAll(".terminal-dot").each(function (d, i) {
      const terminal = terminals[i];

      if (!terminal) return;

      d3.select(this)
        .style("cursor", "pointer")
        .on("click", function (event) {
          event.stopPropagation();

          popup.innerHTML = `
            <button
              style="
                position:absolute;
                top:4px;
                right:7px;
                border:none;
                background:none;
                color:black;
                font-size:20px;
                cursor:pointer;
              "
              aria-label="Close"
            >×</button>

            <div style="
              font-size:15px;
              font-weight:600;
              margin-bottom:10px;
              padding-right:20px;
            ">
              ${terminal.name}
            </div>

            <div>
              <strong>Address:</strong><br>
              ${terminal.address}
            </div>

            <div style="margin-top:8px;">
              <strong>Coordinates:</strong><br>
              ${terminal.latitude.toFixed(4)},
              ${terminal.longitude.toFixed(4)}
            </div>
          `;

          popup.querySelector("button").addEventListener("click", () => {
            popup.style.display = "none";
          });

          const [x, y] = d3.pointer(event, plot);

          popup.style.left = `${Math.max(10, Math.min(x + 12, width - 300))}px`;

          popup.style.top = `${Math.max(10, Math.min(y - 20, height - 160))}px`;

          popup.style.display = "block";
        });
    });

    // -------------------------
    // Click report dots
    // -------------------------

    svg.selectAll("circle").each(function (d, i) {
      const report = filteredReports[i];

      if (!report) return;

      const circle = d3.select(this);

      circle.style("cursor", "pointer").on("click", function (event) {
        event.stopPropagation();

        const type = report.petroleumRelated
          ? "Petroleum-related odor report"
          : "Other odor report";

        const date = formatDate(report.day ?? report.datetime);

        const wind = report.compass || "Not available";

        const description = report.description || "No description provided.";

        const comments = report.comments || "";

        popup.innerHTML = `
            <button
              style="
                position:absolute;
                top:4px;
                right:7px;
                border:none;
                background:none;
                color:black;
                font-size:20px;
                cursor:pointer;
              "
              aria-label="Close"
            >
              ×
            </button>

            <div style="
              font-size:15px;
              font-weight:600;
              margin-bottom:10px;
              padding-right:20px;
            ">
              ${type}
            </div>

            <div style="margin-bottom:6px;">
              <strong>Date:</strong>
              ${date}
            </div>

            <div style="margin-bottom:6px;">
              <strong>Wind direction:</strong>
              ${wind}
            </div>

            <div>
              <strong>Description:</strong><br>
              ${description}
            </div>

            ${
              comments
                ? `
                  <div style="margin-top:8px;">
                    <strong>Comments:</strong><br>
                    ${comments}
                  </div>
                `
                : ""
            }
          `;

        // Reconnect close button after rebuilding popup HTML
        popup.querySelector("button").addEventListener("click", () => {
          popup.style.display = "none";
        });

        // Position popup near clicked point
        const [x, y] = d3.pointer(event, plot);

        popup.style.left = `${Math.min(x + 12, width - 300)}px`;

        popup.style.top = `${Math.max(y - 20, 10)}px`;

        popup.style.display = "block";
      });
    });

    // Clicking the map background closes popup
    svg.on("click", () => {
      popup.style.display = "none";
    });

    // -------------------------
    // Map container
    // -------------------------

    const attribution = document.createElement("div");

    attribution.style.position = "absolute";
    attribution.style.bottom = "0";
    attribution.style.right = "0";
    attribution.style.padding = "4px 7px";
    attribution.style.background = "rgba(255,255,255,0.9)";
    attribution.style.font = "11px sans-serif";
    attribution.style.zIndex = "10";

    attribution.innerHTML =
      '<a href="https://www.openstreetmap.org/copyright" ' +
      'target="_blank" rel="noopener noreferrer">' +
      "© OpenStreetMap contributors</a>";

    const container = document.createElement("div");

    container.style.position = "relative";
    container.style.width = "100%";

    container.appendChild(plot);
    container.appendChild(controls);
    container.appendChild(legend);
    container.appendChild(popup);
    container.appendChild(attribution);

    return container;
  }),
);
```

_(Click a point to see what was reported at that location)_

```js
display(html`
  <div style="margin: 12px 0; font-size: 14px;">
    <strong>Selected range:</strong>
    ${formatMapDateRange(startEnd)}
  </div>
`);
```

```js
function formatMapDateRange([start, end]) {
  const format = d3.timeFormat("%b %Y");

  // The end date is exclusive, so show the previous month.
  const lastIncludedMonth = d3.timeMonth.offset(d3.timeMonth.floor(end), -1);

  return `${format(start)} – ${format(lastIncludedMonth)}`;
}
```

```js
const mapTimelineCounts = d3.rollup(
  dataWithPetroleum,
  (rows) => rows.length,
  (d) => +d3.timeMonth.floor(d.datetime),
);

const mapTimelineFirstMonth = d3.timeMonth.floor(
  d3.min(dataWithPetroleum, (d) => d.datetime),
);

const mapTimelineLastMonth = d3.timeMonth.floor(
  d3.max(dataWithPetroleum, (d) => d.datetime),
);

const mapTimeline = d3.timeMonth
  .range(mapTimelineFirstMonth, d3.timeMonth.offset(mapTimelineLastMonth, 1))
  .map((month) => ({
    month,
    count: mapTimelineCounts.get(+month) ?? 0,
  }));

// -------------------------
// Shared date-range state
// -------------------------

const defaultStartEnd = [
  mapTimelineFirstMonth,
  d3.timeMonth.offset(mapTimelineFirstMonth, 12),
];

const startEnd = Mutable(defaultStartEnd);

const setStartEnd = (range) => {
  startEnd.value = range ?? defaultStartEnd;
};

const getStartEnd = () => startEnd.value;

// -------------------------
// Interactive timeline brush
// -------------------------

display(
  Plot.plot({
    height: 180,

    y: {
      grid: true,
      label: "Number of reports",
      zero: true,
    },

    x: {
      label: "Drag the selection to choose a date range",
    },

    marks: [
      Plot.areaY(mapTimeline, {
        x: "month",
        y: "count",
        fill: "#d97706",
        fillOpacity: 0.18,
      }),

      Plot.lineY(mapTimeline, {
        x: "month",
        y: "count",
        stroke: "#d97706",
      }),

      (index, scales, channels, dimensions, context) => {
        const x1 = dimensions.marginLeft;
        const x2 = dimensions.width - dimensions.marginRight;
        const y1 = dimensions.marginTop;
        const y2 = dimensions.height - dimensions.marginBottom;

        const brushed = (event) => {
          if (!event.sourceEvent || !event.selection) return;

          setStartEnd(event.selection.map(scales.x.invert));
        };

        const brush = d3
          .brushX()
          .extent([
            [x1, y1],
            [x2, y2],
          ])
          .on("brush end", brushed);

        const g = d3.create("svg:g").call(brush);

        // Initialize the selection at the beginning of the timeline
        g.call(brush.move, defaultStartEnd.map(scales.x));

        return g.node();
      },
    ],
  }),
);
```

### Petroleum-related reports extend beyond the waterfront

Orange points are reports whose descriptions or comments contain
petroleum-related terms such as oil, fuel, diesel, gasoline, or asphalt.

The pattern does not prove that petroleum infrastructure caused any
individual odor report. Report locations are also approximate, and smell
reports are not measurements of pollutant concentrations.

### What this adds to air monitoring

DEP and CDC's fixed monitors measure air quality at specific locations.
Smell reports capture something different: **where people notice odors
throughout the community.**

Together, these sources could provide a more complete picture of when
and where petroleum-related odors reach neighborhoods.

## Our recommendation

**Use community smell reports alongside fence-line monitoring to identify
where and when petroleum-related odors are reaching neighborhoods.**
