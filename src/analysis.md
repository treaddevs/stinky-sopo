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

## Most Common Petroleum-Related Keywords

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
        tip: {
            format: {
                x: false,
                y: false,
            },
            channels: {
                Keyword: "term",
                Reports: "count",
            },
        },
      }),
      Plot.ruleX([0]),
    ],
  }),
);
```

## Petroleum-related Reports Over Time

To examine how petroleum-related odor reports changed over time, I grouped the reports by month using the **`datetime`** field and counted the number of reports classified as petroleum-related in each month. The resulting time series shows periods when petroleum-related reports were more or less frequent. 

Total reports, and those charted here capturing petroleum-related keywords peaked between 2020-2021. We can infer that more people were using the app to report unpleasant odors during the COVID-19 pandemic.

```js
Plot.plot({
  y: {
    grid: true,
    label: "Petroleum-related reports"
  },
  x: {
    label: "Date"
  },
  marks: [
    Plot.lineY(
      dataWithPetroleum.filter(d => d.petroleumRelated),
      Plot.binX(
        {y: "count"},
        {
          x: "datetime",
          interval: "month"
        }
      )
    )
  ]
})
```

## Petroleum-related Reports by Wind Direction

To investigate how petroleum-related reports were distributed across prevailing wind directions, I calculated the percentage of petroleum-related reports associated with each wind direction.

The wind direction comes from the **`compass`** field in the dataset and represents the prevailing wind direction associated with the report. For each direction, the percentage represents the number of petroleum-related reports associated with that direction divided by the total number of petroleum-related reports with a recorded wind direction.

**Percentage = reports for each wind direction ÷ total petroleum-related reports × 100**

This relationship can reveal patterns worth investigating further, but it does not establish that wind direction or a particular source caused the reported odors.

```js
Plot.plot({
  marginLeft: 50,
  x: {
    label: "Petroleum-related reports (%)",
    domain: [0, 50],
    grid: true,
    tickFormat: d => `${d}%`
  },
  y: {
    label: "Wind direction",
    domain: ["N", "NE", "E", "SE", "S", "SW", "W", "NW"]
  },
  marks: [
    Plot.barX(
      petroleumByWindDirection,
      {
        x: "percentage",
        y: "direction",
        sort: {y: "-x"},
        fill: "#d97706",
        tip: {
            format: {
                x: false,
                y: false,
            },
            channels: {
                "Wind direction": "direction",
                "Reports": "count",
                "Share": "percentage",
            },
        },
      }
    ),
    Plot.ruleX([0])
  ]
})
```

```js
const petroleumByWindDirection = ["N", "NE", "E", "SE", "S", "SW", "W", "NW"]
  .map((direction) => {
    const petroleumReports = dataWithPetroleum.filter(
      (d) => d.petroleumRelated,
    );

    const directionReports = petroleumReports.filter(
      (d) => d.compass === direction,
    );

    return {
      direction,
      count: directionReports.length,
      percentage:
        (directionReports.length / petroleumReports.length) * 100,
    };
  });
```