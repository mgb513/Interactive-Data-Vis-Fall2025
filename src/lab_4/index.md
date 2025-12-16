---
title: "Lab 4: Clearwater Crisis"
toc: false
---
```js
const waterQuality = await FileAttachment("data/water_quality.csv")
  .csv({ typed: true })
  .then(data => data.map(d => ({
    ...d,
    date: new Date(d.date)
  })));
```
```js
waterQuality
```



 ```js
 const pollutants = [
  "heavy_metals_ppb",
  "nitrogen_mg_per_L",
  "phosphorus_mg_per_L",
  "turbidity_ntu",
  "dissolved_oxygen_mg_per_L"
];

const waterLong = waterQuality.flatMap(d =>
  pollutants.map(p => ({
    date: d.date,
    station_id: d.station_id,
    pollutant: p,
    value: d[p]
  }))
);
```
```js
waterLong
```
```js
Plot.plot({
  title: "Water Quality by Station",
  width: 900,
  height: 600,
  marginRight: 60,
  y: { grid: true },
  fy: { label: null },
  color: { legend: true },
  marks: [
    Plot.lineY(
      waterLong,
      {
        x: "date",
        y: "value",
        fy: "pollutant",
        stroke: "station_id"
      }
    )
  ]
})
```
## heay metals only


```js
Plot.plot({
  title: "Heavy Metal Contamination by Station",
  width: 900,
  height: 400,
  y: { grid: true, label: "heavy metals (ppb)" },
  x: { label: "date" },
  color: { legend: true },
  marks: [
    Plot.ruleY([30], { stroke: "#ef4444", strokeDasharray: "4,4" }),
    Plot.text([{ label: "EPA Limit (30 ppb)" }], {
      x: new Date("2023-02-01"),
      y: 30,
      text: "label",
      dy: -8,
      fill: "#ef4444",
      fontSize: 10
    }),
    Plot.lineY(
      waterLong.filter(d => d.pollutant === "heavy_metals_ppb"),
      {
        x: "date",
        y: "value",
        stroke: "station_id",
        tip: true
      }
    ),
    Plot.dot(
      waterLong.filter(d => d.pollutant === "heavy_metals_ppb" && d.station_id === "West" && d.value > 30),
      { x: "date", y: "value", fill: "#ef4444", r: 4 }
    ),
    Plot.text(
      waterLong.filter(d => d.pollutant === "heavy_metals_ppb" && d.station_id === "West" && d.value > 30),
      { x: "date", y: "value", text: d => d.date.toLocaleDateString(), dy: -10, fontSize: 9 }
    )
  ]
})
```

Evidence shows systematic heavy metal contamination correlated with quarterly "maintenance shutdowns" — with pollution levels spiking up to 4× normal at the nearest monitoring station.

## Fish Data

```js
const fishSurveys = await FileAttachment("data/fish_surveys.csv")
  .csv({ typed: true })
  .then(data => data.map(d => ({
    ...d,
    date: new Date(d.date)
  })));
  ```
```js
  const fishByQuarter = fishSurveys.map(d => ({
  ...d,
  quarter: `${d.date.getFullYear()}-Q${Math.floor(d.date.getMonth() / 3) + 1}`
}));
```
```js
fishByQuarter
```
```js
Plot.plot({
  title: "Fish Population by Species",
  width: 900,
  height: 400,
  x: { label: null, tickRotate: -45 },
  y: { grid: true, label: "Count" },
  fx: { label: null },
  color: { legend: true },
  marks: [
    Plot.barY(
      fishByQuarter,
      Plot.groupX(
        { y: "sum" },
        { x: "species", y: "count", fill: "species", fx: "quarter" }
      )
    )
  ]
})
```

```js
Plot.plot({
  title: "Fish Population by Species and Station",
  width: 750,
  height: 600,
  x: { label: null, tickRotate: -45 },
  y: { grid: true, label: "Count" },
  fx: { label: null },
  fy: { label: null },
  color: {
    legend: true,
    domain: ["Trout", "Bass", "Carp"],
    range: ["#ef8cd5ff", "#be7febff", "#a9a9b4ff"]
  },
  marks: [
    Plot.barY(
      fishByQuarter,
      Plot.groupX(
        { y: "sum" },
        {
          x: "species",
          y: "count",
          fill: "species",
          fx: "quarter",
          fy: "station_id",
          inset: 2
        }
      )
    )
  ]
})
```

Trout and Bass at West Station.
```js
Plot.plot({
  title: "Fish Population by Station",
  width: 900,
  height: 500,
  x: { label: null, tickRotate: -45 },
  y: { grid: true, label: "Count" },
  fx: { label: null },
  fy: { label: null },
  color: { legend: true },
  marks: [
    Plot.barY(
      fishByQuarter,
      Plot.groupX(
        { y: "sum" },
        { x: "station_id", y: "count", fill: "station_id", fx: "quarter", fy: "species" }
      )
    )
  ]
})
```
```js
Plot.plot({
  title: "Trout and Bass at West Station",
  width: 900,
  height: 400,
  x: { label: null },
  y: { grid: true, label: "Count" },
  fx: { label: null },
  color: { legend: true },
  marks: [
    Plot.barY(
      fishByQuarter.filter(d => d.station_id === "West" && (d.species === "Trout" || d.species === "Bass")),
      Plot.groupX(
        { y: "sum" },
        { x: "species", y: "count", fill: "species", fx: "quarter" }
      )
    )
  ]
})
```
You should see Trout dropping from ~43 down to ~13 while Bass falls from ~60 to ~37. Both declining, but Trout's collapse is more dramatic — exactly what you'd expect with heavy metal pollution hitting sensitive species harder.

```js
const activities = await FileAttachment("data/suspect_activities.csv")
  .csv({ typed: true })
  .then(data => data.map(d => ({
    ...d,
    date: new Date(d.date)
  })));
```

```js
const activitiesDaily = activities.flatMap(d => {
  const days = [];
  for (let i = 0; i < d.duration_days; i++) {
    const date = new Date(d.date);
    date.setDate(date.getDate() + i);
    days.push({
      ...d,
      date: date
    });
  }
  return days;
});
```
```js
activitiesDaily
```
```js
Plot.plot({
  title: "Suspect Activity Timeline",
  width: 750,
  height: 300,
  y: {label: null},
  x: {
    grid: true,
    domain: [new Date("2023-01-01"), new Date("2025-03-01")]
  },
  color: {
    legend: true,
    domain: ["Low", "Medium", "High"],
    range: ["#fecaca", "#f87171", "#b91c1c"]
  },
  marks: [
    Plot.barX(activities, {
      x1: "date",
      x2: (d) => d3.timeDay.offset(d.date, d.duration_days),
      y: "suspect",
      fill: "intensity",
      tip: true
    })
  ]
})
```

```js
Plot.plot({
  title: "Heavy Metals at West Station vs. ChemTech Shutdowns",
  width: 900,
  height: 400,
  marginRight: 150,
  x: {
    grid: true,
    tickFormat: d3.timeFormat("%b %Y"),
    ticks: activities.filter(d => d.suspect === "ChemTech Manufacturing").map(d => d.date)
  },
  y: {grid: true, label: "heavy metals (ppb)", domain: [-3, 55]},
  marks: [
    Plot.rectX(
      activities.filter(d => d.suspect === "ChemTech Manufacturing"),
      {
        x1: "date",
        x2: (d) => d3.timeDay.offset(d.date, d.duration_days),
        y1: 0,
        y2: 53,
        fill: "#fee2e2",
        stroke: "#f87171",
        strokeWidth: 1
      }
    ),
    Plot.text(
      activities.filter(d => d.suspect === "ChemTech Manufacturing"),
      {
        x: "date",
        y: 52,
        text: "Shutdown",
        fontSize: 8,
        fill: "#b91c1c"
      }
    ),
    Plot.ruleY([20], {stroke: "#facc15", strokeDasharray: "4,4"}),
    Plot.ruleY([30], {stroke: "#ef4444", strokeDasharray: "4,4"}),
    Plot.text([{y: 20, label: "EPA Concern (20 ppb)"}], {
      x: new Date("2024-12-15"),
      y: "y",
      text: "label",
      fill: "#facc15",
      dx: 60,
      fontSize: 10
    }),
    Plot.text([{y: 30, label: "EPA Limit (30 ppb)"}], {
      x: new Date("2024-12-15"),
      y: "y",
      text: "label",
      fill: "#ef4444",
      dx: 60,
      fontSize: 10
    }),
    Plot.lineY(
      waterLong.filter(d => d.pollutant === "heavy_metals_ppb" && d.station_id === "West"),
      {x: "date", y: "value", stroke: "#0891b2", strokeWidth: 2, tip: true}
    ),
    Plot.dot(
      waterLong.filter(d => d.pollutant === "heavy_metals_ppb" && d.station_id === "West" && d.value > 20),
      {x: "date", y: "value", fill: "#ef4444", r: 5}
    )
  ]
})
```

<div style="display: flex; gap: 20px; font-size: 12px; margin-top: 8px;">
  <div style="display: flex; align-items: center; gap: 6px;">
    <div style="width: 16px; height: 12px; background: #fee2e2; border: 1px solid #f87171;"></div>
    <span>ChemTech Maintenance Shutdown</span>
  </div>
  <div style="display: flex; align-items: center; gap: 6px;">
    <div style="width: 16px; height: 3px; background: #0891b2;"></div>
    <span>Heavy Metals (West Station)</span>
  </div>
</div>


-----
## New Metric

instead of using the self-reported "intensity" from the activities data, we calculate actual impact based on heavy metal readings during each activity.
Let's create a metric. For each activity, we could calculate:

Peak heavy metals at West station during the activity period
Average heavy metals during the activity
Spike above baseline (reading minus ~12 ppb baseline)

```js
const activitiesWithImpact = activities.map(d => {
  const startDate = d.date;
  const endDate = d3.timeDay.offset(d.date, d.duration_days);
  
  const readings = waterLong.filter(r => 
    r.pollutant === "heavy_metals_ppb" &&
    r.station_id === "West" &&
    r.date >= startDate &&
    r.date <= endDate
  );
  
  const peakReading = readings.length > 0 ? d3.max(readings, r => r.value) : null;
  const avgReading = readings.length > 0 ? d3.mean(readings, r => r.value) : null;
  
  return {
    ...d,
    peak_metals: peakReading,
    avg_metals: avgReading,
    spike: peakReading ? peakReading - 12 : null
  };
});
```


## weekly impact

```js
const weeklyImpact = activities.flatMap(d => {
  const weeks = [];
  let currentDate = d.date;
  const endDate = d3.timeDay.offset(d.date, d.duration_days);
  
  while (currentDate < endDate) {
    const weekEnd = d3.timeDay.offset(currentDate, 7);
    
    const readings = waterLong.filter(r =>
      r.pollutant === "heavy_metals_ppb" &&
      r.station_id === "West" &&
      r.date >= currentDate &&
      r.date < weekEnd
    );
    
    const avgReading = readings.length > 0 ? d3.mean(readings, r => r.value) : null;
    
    weeks.push({
      suspect: d.suspect,
      activity_type: d.activity_type,
      week_start: new Date(currentDate),
      avg_metals: avgReading
    });
    
    currentDate = weekEnd;
  }
  
  return weeks;
});
```

```js
const suspectStation = {
  "ChemTech Manufacturing": "West",
  "Riverside Farm": "North",
  "Lakeview Resort": "East",
  "Clearwater Fishing Lodge": "South"
};

const weeklyImpactByStation = activities.flatMap(d => {
  const weeks = [];
  let currentDate = d.date;
  const endDate = d3.timeDay.offset(d.date, d.duration_days);
  const station = suspectStation[d.suspect];
  
  while (currentDate < endDate) {
    const weekEnd = d3.timeDay.offset(currentDate, 7);
    
    const readings = waterLong.filter(r =>
      r.pollutant === "heavy_metals_ppb" &&
      r.station_id === station &&
      r.date >= currentDate &&
      r.date < weekEnd
    );
    
    const avgReading = readings.length > 0 ? d3.mean(readings, r => r.value) : null;
    
    weeks.push({
      suspect: d.suspect,
      activity_type: d.activity_type,
      station: station,
      week_start: new Date(currentDate),
      avg_metals: avgReading
    });
    
    currentDate = weekEnd;
  }
  
  return weeks;
});
```

```js
Plot.plot({
  title: "Suspect Activity Timeline — Weekly Heavy Metal Impact",
  width: 900,
  height: 300,
  marginLeft: 180,
  marginBottom: 40,
  y: {label: null},
  x: {grid: true, label: null},
  color: {
    legend: true,
    label: "Avg Heavy Metals (ppb)",
    scheme: "reds",
    domain: [8, 50]
  },
  marks: [
    Plot.barX(weeklyImpactByStation.map(d => ({
      ...d,
      suspectLabel: `${d.suspect}\n(${d.station} Station)`
    })), {
      x1: "week_start",
      x2: (d) => d3.timeDay.offset(d.week_start, 7),
      y: "suspectLabel",
      fill: "avg_metals",
      stroke: "#666",
      strokeWidth: 0.5,
      tip: true,
      channels: {
        Activity: "activity_type",
        Date: (d) => d.week_start.toLocaleDateString("en-US", {month: "short", day: "numeric", year: "numeric"}),
        "Heavy Metals (ppb)": "avg_metals"
      }
    })
  ]
})
```



# Evidence Summary

Geographic correlation: The West monitoring station, located just 800m from ChemTech Manufacturing, shows consistently elevated heavy metal levels (17% above lake average).

Temporal correlation: Heavy metal concentrations spike dramatically during ChemTech's quarterly "maintenance shutdowns" — rising from ~12 ppb baseline to peaks of 23–49 ppb.

Biological indicator: Pollution-sensitive Trout populations at West station have crashed 70% while pollution-tolerant Carp have increased 35% — a classic signature of heavy metal contamination.

Ruling out alternatives: Riverside Farm's fertilizer applications would cause nitrogen/phosphorus spikes (not observed significantly). Lakeview Resort construction would cause turbidity increases (actually decreased at East station). Clearwater Lodge's fishing activity cannot explain heavy metal contamination.
