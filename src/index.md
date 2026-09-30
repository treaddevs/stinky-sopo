---
toc: false
---

# Stinky South Portland

## Exploring petroleum-related odor reports

South Portland's Stinky Reports dataset documents odor complaints submitted by residents. This project examines reports containing language associated with petroleum—including oil, petroleum, gasoline, diesel, fuel, asphalt, and tar—to explore how frequently these reports occur, how residents describe them, and where they are concentrated geographically.

The analysis is based on keyword matching in the `description` and `comments` fields. A petroleum-related classification indicates that a report contains at least one targeted keyword; it does not establish the chemical composition or source of an odor.

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

## Petroleum-related reports make up part of the dataset

```js
const reportCategoryCounts = [
  {
    category: "Petroleum-related",
    count: dataWithPetroleum.filter((d) => d.petroleumRelated).length,
  },
  {
    category: "Other",
    count: dataWithPetroleum.filter((d) => !d.petroleumRelated).length,
  },
];

display(
  Plot.plot({
    height: 400,
    y: {
      grid: true,
      label: "Number of reports",
    },
    x: {
      label: null,
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
    ],
  }),
);
```

## "Oil" is the most common petroleum-related term

```js
const petroleumKeywordCounts = petroleumKeywords
  .map((keyword) => ({
    term: keyword,
    count: dataWithPetroleum.reduce((total, d) => {
      const text = `${d.description ?? ""} ${d.comments ?? ""}`.toLowerCase();

      return total + (text.includes(keyword) ? 1 : 0);
    }, 0),
  }))
  .sort((a, b) => b.count - a.count);

display(
  Plot.plot({
    height: 500,
    marginLeft: 110,
    x: {
      grid: true,
      label: "Number of reports mentioning keyword",
    },
    y: {
      label: null,
      domain: petroleumKeywordCounts.map((d) => d.term),
    },
    marks: [
      Plot.barX(petroleumKeywordCounts, {
        x: "count",
        y: "term",
        sort: { y: "-x" },
        fill: "#d97706",
        tip: true,
      }),
      Plot.ruleX([0]),
    ],
  }),
);
```

## Explore the reports

```js
const reportType = view(
  Inputs.select(["All reports", "Petroleum-related", "Other"], {
    label: "Show:",
    value: "Petroleum-related",
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

    return Plot.plot({
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

          fill: (d) => (d.petroleumRelated ? "#d97706" : "#888888"),

          stroke: "white",
          strokeWidth: 0.75,

          title: (d) => {
            const date = d.day ? d3.timeFormat("%Y-%m-%d")(d.day) : "";

            return [
              d.petroleumRelated ? "Petroleum-related" : "Other",
              date,
              d.compass ? `Wind: ${d.compass}` : "",
              d.description ?? "",
              d.comments ?? "",
            ]
              .filter(Boolean)
              .join("\n");
          },
        }),
      ],
    });
  }),
);
```

### Interpreting the map

The geographic distribution shows where reports were recorded, but it does not establish that petroleum infrastructure caused an odor report. The petroleum-related classification is based only on words appearing in the report text, and the dataset does not provide chemical measurements or verified odor sources.
