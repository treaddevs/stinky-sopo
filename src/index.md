---
toc: false
---

# Residents are noticing petroleum odors beyond the monitoring fence line

South Portland's smell reports show where residents are experiencing
petroleum-related odors across the community. These reports provide a
community-level signal that fixed air monitors cannot capture on their own.

## More than half of the reports contain petroleum-related language

```js
const petroleumCount = dataWithPetroleum.filter(
  (d) => d.petroleumRelated,
).length;

const otherCount = dataWithPetroleum.filter(
  (d) => !d.petroleumRelated,
).length;

const totalReports = dataWithPetroleum.length;

const reportCategoryCounts = [
  {
    category: "Petroleum-related",
    count: petroleumCount,
    percentage: petroleumCount / totalReports,
    label: `${petroleumCount} (${d3.format(".0%")(
      petroleumCount / totalReports,
    )})`,
  },
  {
    category: "Other",
    count: otherCount,
    percentage: otherCount / totalReports,
    label: `${otherCount} (${d3.format(".0%")(
      otherCount / totalReports,
    )})`,
  },
];

display(
  Plot.plot({
    height: 450,
    y: {
      grid: true,
      label: "Number of reports",
      inset: 10,
      domain: [0, d3.max(reportCategoryCounts, d => d.count) * 1.12],
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
  }),
);
```

56% of the reports contain at least one petroleum-related keyword.

This does not mean that 56% of the air contains petroleum. It means that
petroleum-related language appears in 56% of the submitted reports.

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
const filteredReports =
  reportType === "All reports"
    ? dataWithPetroleum
    : dataWithPetroleum.filter((d) =>
        reportType === "Petroleum-related"
          ? d.petroleumRelated
          : !d.petroleumRelated,
      );
```

```js
function basemapUrl(z, x, y) {
  return `https://services.arcgisonline.com/ArcGIS/rest/services/Canvas/World_Light_Gray_Base/MapServer/tile/${z}/${y}/${x}`;
}

function basemap(projection, width, height) {
  return () => {
    const tiles = tile()
      .extent([
        [0, 0],
        [width, height],
      ])
      .scale(projection.scale() * 2 * Math.PI)
      .translate(projection([0, 0]))();

    const [tx, ty] = tiles.translate;
    const k = tiles.scale;

    const clipId = `basemap-${Math.random().toString(36).slice(2)}`;

    const g = d3.create("svg:g").attr("clip-path", `url(#${clipId})`);

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
  };
}
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

        Plot.dot(filteredReports, {
          x: "longitude",
          y: "latitude",

          r: 4,

          fill: (d) =>
            d.petroleumRelated
              ? "#d97706"
              : "#888888",

          stroke: "white",
          strokeWidth: 0.75,

          // Used to identify the dots for click events
          className: "report-dot",
        }),
      ],
    });

    // -------------------------
    // Zoom and pan
    // -------------------------

    const svg = d3.select(plot);

    const mapGroups = svg.selectAll(":scope > g");

    const zoom = d3
      .zoom()
      .scaleExtent([1, 8])
      .extent([
        [0, 0],
        [width, height],
      ])
      .translateExtent([
        [0, 0],
        [width, height],
      ])
      .on("zoom", (event) => {
        mapGroups.attr("transform", event.transform);
      });

    svg.call(zoom);

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
      svg
        .transition()
        .duration(250)
        .call(zoom.scaleBy, 1.5);
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
      <strong>Report Type</strong>

      <div style="margin-top:6px;">
        <span style="
          display:inline-block;
          width:10px;
          height:10px;
          border-radius:50%;
          background:#d97706;
          margin-right:6px;
        "></span>
        Petroleum-related
      </div>

      <div>
        <span style="
          display:inline-block;
          width:10px;
          height:10px;
          border-radius:50%;
          background:#888888;
          margin-right:6px;
        "></span>
        Other
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

    // -------------------------
    // Click report dots
    // -------------------------

    svg.selectAll("circle").each(function (d, i) {
      const report = filteredReports[i];

      if (!report) return;

      const circle = d3.select(this);

      circle
        .style("cursor", "pointer")
        .on("click", function (event) {
          event.stopPropagation();

          const type = report.petroleumRelated
            ? "Petroleum-related odor report"
            : "Other odor report";

          const date = formatDate(
            report.day ?? report.datetime
          );

          const wind = report.compass || "Not available";

          const description =
            report.description ||
            "No description provided.";

          const comments =
            report.comments || "";

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
          popup
            .querySelector("button")
            .addEventListener("click", () => {
              popup.style.display = "none";
            });

          // Position popup near clicked point
          const [x, y] = d3.pointer(event, plot);

          popup.style.left = `${Math.min(
            x + 12,
            width - 300
          )}px`;

          popup.style.top = `${Math.max(
            y - 20,
            10
          )}px`;

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

    const container = document.createElement("div");

    container.style.position = "relative";
    container.style.width = "100%";

    container.appendChild(plot);
    container.appendChild(controls);
    container.appendChild(legend);
    container.appendChild(popup);

    return container;
  }),
);
```
*(Click a point to see what was reported at that location)*

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