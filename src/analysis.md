---
toc: false
---

# Stinky SOPO - Smell MyCity Data

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

```js
const petroleumKeywordCounts = petroleumKeywords
  .map((keyword) => ({
    term: keyword,
    count: dataWithPetroleum.reduce((total, d) => {
      const inRange = startEnd[0] <= d.datetime && d.datetime < startEnd[1];

      if (!inRange) return total;

      const text = `${d.description ?? ""} ${d.comments ?? ""}`.toLowerCase();

      return total + (text.includes(keyword) ? 1 : 0);
    }, 0),
  }))
  .sort((a, b) => b.count - a.count);

display(html`
  <h2>Most Common Petroleum-Related Keywords</h2>
  <p>Reports from ${formatDateRange(startEnd)}</p>
  ${Plot.plot({
    height: 500,
    marginLeft: 110,
    x: {
      grid: true,
      label: "Number of reports mentioning keyword",
      zero: true,
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
        tip: {
          format: { x: false, y: false },
          channels: {
            Keyword: "term",
            Reports: "count",
          },
        },
      }),
      Plot.ruleX([0]),
    ],
  })}
`);
```

```js
const monthlyPetroleumCounts = d3.rollup(
  dataWithPetroleum.filter((d) => d.petroleumRelated),
  (rows) => rows.length,
  (d) => +d3.timeMonth.floor(d.datetime),
);

const firstMonth1 = d3.timeMonth.floor(
  d3.min(dataWithPetroleum, (d) => d.datetime),
);

const lastMonth1 = d3.timeMonth.floor(
  d3.max(dataWithPetroleum, (d) => d.datetime),
);

const petroleumTimeline = d3.timeMonth
  .range(firstMonth1, d3.timeMonth.offset(lastMonth1, 1))
  .map((month) => ({
    month,
    count: monthlyPetroleumCounts.get(+month) ?? 0,
  }));

display(
  Plot.plot({
    height: 180,
    y: {
      grid: true,
      label: "Petroleum-related reports",
      zero: true,
    },
    x: {
      label: "Select a date range →",
    },
    marks: [
      Plot.lineY(petroleumTimeline, {
        x: "month",
        y: "count",
        stroke: "#d97706",
      }),

      (index, scales, channels, dimensions, context) => {
        const x1 = dimensions.marginLeft;
        const y1 = dimensions.marginTop;
        const x2 = dimensions.width - dimensions.marginRight;
        const y2 = dimensions.height - dimensions.marginBottom;

        const brushed = (event) => {
          if (!event.sourceEvent) return;

          let { selection } = event;

          if (!selection) return;

          const dates = selection.map(scales.x.invert);
          setStartEnd(dates);
        };

        const brush = d3
          .brushX()
          .extent([
            [x1, y1],
            [x2, y2],
          ])
          .on("brush end", brushed);

        const g = d3.create("svg:g").call(brush);

        const windowSize = 12; // months
        const start = petroleumTimeline[0].month;
        const end = d3.timeMonth.offset(start, windowSize);

        g.call(brush.move, [start, end].map(scales.x));

        return g.node();
      },
    ],
  }),
);
```

## Petroleum-related Reports Over Time

To examine how odor reports changed over time, I plotted reports by month using the **`datetime`** field counting the number of reports containing petroleum-related keywords for each month. I also counted reports that did not contain petroleum language which I plotted alongside the petroleum-related reports.

Reports peaked between 2020 and 2021 and in **April 2021**, there were approximately **139% more petroleum-related reports** than other reports.

```js
function formatDateRange([start, end]) {
  const format = d3.utcFormat("%b %Y");

  // The brush's end date is exclusive, so display the
  // last month included in the selected range.
  const lastIncluded = d3.utcMonth.offset(end, -1);

  return `${format(start)} – ${format(lastIncluded)}`;
}
```

```js
const categories = ["Petroleum-related", "Other reports"];

const classifiedReports = dataWithPetroleum.map((d) => ({
  ...d,
  category: d.petroleumRelated ? "Petroleum-related" : "Other reports",
}));

const monthlyCounts = d3.rollup(
  classifiedReports,
  (rows) => rows.length,
  (d) => d.category,
  (d) => +d3.timeMonth.floor(d.datetime),
);

const firstMonth2 = d3.timeMonth.floor(
  d3.min(classifiedReports, (d) => d.datetime),
);

const lastMonth2 = d3.timeMonth.floor(
  d3.max(classifiedReports, (d) => d.datetime),
);

const monthlyReports = d3.timeMonth
  .range(firstMonth2, d3.timeMonth.offset(lastMonth2, 1))
  .flatMap((month) =>
    categories.map((category) => ({
      month,
      category,
      count: monthlyCounts.get(category)?.get(+month) ?? 0,
    })),
  );
```

```js
const defaultStartEnd = [
  d3.timeMonth.offset(lastMonth2, -11),
  d3.timeMonth.offset(lastMonth2, 1),
];

const startEnd = Mutable(defaultStartEnd);

const setStartEnd = (range) => {
  startEnd.value = range ?? defaultStartEnd;
};

const getStartEnd = () => startEnd.value;
```

```js
display(html`
  <h2>Monthly odor reports: ${formatDateRange(startEnd)}</h2>
  ${Plot.plot({
    y: {
      grid: true,
      label: "Monthly reports",
      zero: true,
    },
    x: {
      label: "Month",
    },
    color: {
      domain: categories,
      range: ["#d97706", "#888888"],
      legend: true,
    },
    marks: [
      Plot.lineY(
        monthlyReports.filter(
          (d) => startEnd[0] <= d.month && d.month < startEnd[1],
        ),
        {
          x: "month",
          y: "count",
          stroke: "category",
          tip: true,
        },
      ),
    ],
  })}
`);
```

```js
display(html`
  <h2>Monthly odor reports: May 2019 - Aug 2026</h2>
  ${Plot.plot({
    y: {
      grid: true,
      label: "Monthly reports",
      zero: true,
    },
    x: {
      label: "Month",
    },
    color: {
      domain: categories,
      range: ["#d97706", "#888888"],
      legend: true,
    },
    marks: [
      Plot.lineY(monthlyReports, {
        x: "month",
        y: "count",
        stroke: "category",
        tip: true,
      }),

      (index, scales, channels, dimensions, context) => {
        const x1 = dimensions.marginLeft;
        const y1 = 0;
        const x2 = dimensions.width - dimensions.marginRight;
        const y2 = dimensions.height - dimensions.marginBottom;

        const brushed = (event) => {
          if (!event.sourceEvent) return;

          let { selection } = event;

          if (!selection) {
            const r = 10;
            let [px] = d3.pointer(event, context.ownerSVGElement);

            px = Math.max(x1 + r, Math.min(x2 - r, px));

            selection = [px - r, px + r];
            g.call(brush.move, selection);
          }

          setStartEnd(selection.map(scales.x.invert));
        };

        const brush = d3
          .brushX()
          .extent([
            [x1, y1],
            [x2, y2],
          ])
          .on("brush end", brushed);

        const g = d3.create("svg:g").call(brush);

        g.call(brush.move, getStartEnd().map(scales.x));

        return g.node();
      },
    ],
  })}
`);
```

## Reports by Wind Direction

To investigate how odor reports were distributed across prevailing wind directions, I calculated the percentage of **petroleum-related** and **other reports** associated with each wind direction.

The wind direction comes from the **`compass`** field in the dataset and represents the prevailing wind direction associated with the report. For each report type, the percentage represents the number of reports associated with a given wind direction divided by the total number of reports of that type with a recorded wind direction.

**Percentage = reports of each type for a given wind direction ÷ total reports of that type with a recorded wind direction × 100**

Comparing these distributions can reveal patterns worth investigating further, but it does not establish that wind direction or a particular source caused the reported odors.

```js
const windDirections = ["N", "NE", "E", "SE", "S", "SW", "W", "NW"];

const reportsWithWind = dataWithPetroleum.filter((d) =>
  windDirections.includes(String(d.compass ?? "").trim().toUpperCase()),
);

const totalByType = {
  "Petroleum-related": reportsWithWind.filter((d) => d.petroleumRelated).length,
  "Other reports": reportsWithWind.filter((d) => !d.petroleumRelated).length,
};

const reportsByWindDirection = windDirections.flatMap((direction) => {
  const directionReports = reportsWithWind.filter(
    (d) => String(d.compass).trim().toUpperCase() === direction,
  );

  return ["Petroleum-related", "Other reports"].map((type) => {
    const isPetroleum = type === "Petroleum-related";
    const count = directionReports.filter(
      (d) => Boolean(d.petroleumRelated) === isPetroleum,
    ).length;
    const denominator = totalByType[type];

    return {
      direction,
      type,
      count,
      percentage: denominator ? (count / denominator) * 100 : 0,
      label: `${denominator ? ((count / denominator) * 100).toFixed(2) : "0.00"}%`,
    };
  });
});
```

```js
display(
  Plot.plot({
    height: 360,
    marginBottom: 50,
    x: {
      label: "Wind direction",
      domain: windDirections,
    },
    y: {
      label: "Reports by type (%)",
      domain: [0, 50],
      grid: true,
      tickFormat: (d) => `${d}%`,
    },
    color: {
      domain: ["Petroleum-related", "Other reports"],
      range: ["#d97706", "#888888"],
      legend: true,
    },
    marks: [
      Plot.barY(reportsByWindDirection, {
        x: "direction",
        y: "percentage",
        fill: "type",
        fx: undefined,
        sort: null,
        dx: (d) => (d.type === "Petroleum-related" ? -8 : 8),
        tip: {
          format: {
            x: false,
            y: false,
          },
          channels: {
            "Wind direction": "direction",
            "Report type": "type",
            Reports: "count",
            "Percentage": (d) => `${d.percentage.toFixed(2)}%`,
          },
        },
      }),
      Plot.ruleY([0]),
    ],
  }),
);
```