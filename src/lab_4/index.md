---
title: "Lab 4: Clearwater Crisis"
toc: false
---

# Water Quality 

To determine what caused the ecological strain on Clearwater Lake and to explain the percipitous decline of the lake's trout and bass populations, we begin by taking a closer look at the water quality measurements collected in 2023 nd 2024 at the four measuring stations. This will give us insight into which pollutants exceed regulatory limits and which measuring stations log hazardous levels of pollutants. 

```js
const waterQuality = await FileAttachment("data/water_quality.csv")
  .csv({ typed: true })
  .then(data => data.map(d => ({
    ...d,
    date: new Date(d.date)
  })));
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
const limits = [
  {pollutant: "heavy_metals_ppb", concern: 20, regulatory: 30},
  {pollutant: "nitrogen_mg_per_L", concern: 1.5, regulatory: 2.0},
  {pollutant: "phosphorus_mg_per_L", concern: 0.05, regulatory: 0.1},
  {pollutant: "dissolved_oxygen_mg_per_L", concern: 7, regulatory: 5},
  {pollutant: "turbidity_ntu", concern: 10, regulatory: 25}
];
```

```js
const waterByLimit = waterLong.map(d => {
  const limit = limits.find(l => l.pollutant === d.pollutant);
  if (!limit) return {...d, pct_of_limit: null};
  
  let pct;
  if (d.pollutant === "dissolved_oxygen_mg_per_L") {
    // For oxygen: lower is worse, so invert the calculation
    // 100% means AT the dangerous low level
    pct = (limit.regulatory / d.value) * 100;
  } else {
    // For others: higher is worse
    pct = (d.value / limit.regulatory) * 100;
  }
  
  return {...d, pct_of_limit: pct};
}).filter(d => d.pct_of_limit !== null);
```

```js
const pollutantLabels = {
  "heavy_metals_ppb": "Heavy Metals (ppb)",
  "nitrogen_mg_per_L": "Nitrogen (mg/L)",
  "phosphorus_mg_per_L": "Phosphorus (mg/L)",
  "dissolved_oxygen_mg_per_L": "Dissolved Oxygen (mg/L)",
  "turbidity_ntu": "Turbidity (NTU)"
};

const waterByLimitLabeled = waterByLimit.map(d => ({
  ...d,
  pollutantLabel: pollutantLabels[d.pollutant]
}));
```

```js
Plot.plot({
  title: "Water Quality Measures as % of Regulatory Limit",
  subtitle: "Red dashed line = 100% of EPA regulatory limit (danger zone)",
  width: 900,
  height: 800,
  marginRight: 140,
  marginLeft: 125,
  y: {
    grid: true, 
    label: null,
    axis: "right"
  },
  fy: {label: null},
  color: {
    legend: true,
    domain: ["North Station", "South Station", "East Station", "West Station"],
    range: ["#e76f51", "#e9c46a", "#2a9d8f", "#264653"]
  },
  marks: [
    Plot.frame(),
    Plot.ruleY([100], {stroke: "#ef4444", strokeWidth: 2, strokeDasharray: "4,4"}),
    Plot.lineY(
      waterByLimitLabeled,
      {
        x: "date",
        y: "pct_of_limit",
        fy: "pollutantLabel",
        stroke: d => d.station_id + " Station"
      }
    )
  ]
})
```
The visualizations show that the only measurement that repeatedly exceeds the regulatory limit are the Heavey Metals measured at the West Station. 

Anna Boutsikaris, a Watershed Specialist at the NYS Department of Environmental Conservation (and also part of my family), confirmed that heavy metal contents are most likely to affect aquatic life. 

With this in mind, we can consider other aspects of the data in more detail. We can look at how the suspects' various activities influence heavy metal levels, how ChemTech Manufacturing's activities (located near the West Station) align with spikes in heavy metal readings, and finally, how fish population developed since 2023 overall and specifically near sites with high heavy metal readings. 


# Suspects' Activities & Heavy Metal Contamination


We can create a metric to better understand the heavy metal pollution's impact by tracking weekly heavy metal readings at nearby stations in relation to ther various activities at all four enterprises. Activites range from daily operations at recreatonal facilities to fertilizing activities at a farm to ChemTech's quarterly Maintainance Shutdown (which includes equipment cleaning). 

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
  marginLeft: 125,
  marginBottom: 40,
  y: {label: null},
  x: {grid: true, label: null},
  color: {
    legend: true,
    label: "Avg Heavy Metals (ppb)",
    type: "threshold",
    domain: [10, 20, 30],
    range: ["#deebf7", "#9ecae1", "#4292c6", "#08519c"]
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
      channels: {
        Activity: "activity_type",
        Date: (d) => d.week_start.toLocaleDateString("en-US", {month: "short", day: "numeric", year: "numeric"}),
        "Heavy Metals (ppb)": "avg_metals"
      },
      tip: {
        format: {
          x1: false,
          x2: false,
          y: false,
          fill: false,
          Activity: true,
          Date: true,
          "Heavy Metals (ppb)": true
        }
      }
    })
  ]
})
```

While activities at each of the suspects' show heavy metal readings, the long-lasting/ongoing activities don't show signs of compounding heavy metals measures, and levels stay mostly below 20 ppb (the EPA's concern level). The spikes in heavy metal pollution above the regulatory limit of 30 ppb remain confined to the activities of ChemTech. 


<!--  
```js
Plot.plot({ //heavy metal line graph
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
-->

# A Pattern of Heavy Metal Contamination 

Before investigating the impact on the various fish species, we can zoom in and clarfify the relationship between ChemTech's maintainance shutdowns and spike in heavy metal measurements. 

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
Plot.plot({
  title: "Heavy Metals at West Station vs. ChemTech Shutdowns",
  subtitle: html`<div style="display: flex; flex-wrap: wrap; gap: 16px; font-size: 11px; margin-top: 4px; margin-bottom: 16px;">
    <div style="display: flex; align-items: center; gap: 6px;">
      <div style="width: 16px; height: 12px; background: #deebf7; border: 1px solid #4292c6;"></div>
      <span>ChemTech Maintenance Shutdown</span>
    </div>
    <div style="display: flex; align-items: center; gap: 6px;">
      <div style="width: 16px; height: 3px; background: #264653;"></div>
      <span>Heavy Metals (West Station)</span>
    </div>
    <div style="display: flex; align-items: center; gap: 6px;">
      <div style="width: 16px; height: 0; border-top: 2px dashed #e9c46a;"></div>
      <span>EPA Concern (20 ppb)</span>
    </div>
    <div style="display: flex; align-items: center; gap: 6px;">
      <div style="width: 16px; height: 0; border-top: 2px dashed #ef4444;"></div>
      <span>EPA Limit (30 ppb)</span>
    </div>
  </div>`,
  width: 900,
  height: 400,
  marginRight: 60,
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
        fill: "#deebf7",
        stroke: "#4292c6",
        strokeWidth: 2
      }
    ),
    Plot.text(
      activities.filter(d => d.suspect === "ChemTech Manufacturing"),
      {
        x: "date",
        y: 52,
        text: "Shutdown",
        fontSize: 8,
        fill: "#08519c"
      }
    ),
    Plot.ruleY([20], {stroke: "#e9c46a", strokeDasharray: "4,4", strokeWidth: 2}),
    Plot.ruleY([30], {stroke: "#ef4444", strokeDasharray: "4,4", strokeWidth: 2}),
    Plot.lineY(
      waterLong.filter(d => d.pollutant === "heavy_metals_ppb" && d.station_id === "West"),
      {x: "date", y: "value", stroke: "#264653", strokeWidth: 2, tip: true}
    ),
    Plot.dot(
      waterLong.filter(d => d.pollutant === "heavy_metals_ppb" && d.station_id === "West" && d.value > 20),
      {x: "date", y: "value", fill: "#f10c0cff", r: 6}
    )
  ]
})
```
Nearly every ChemTech shutdown period aligns with a spike in heavy metal readings. This isn't coincidental; it suggests they're dumping or releasing contaminated water during these "maintenance" windows.
The spikes regularly cross the EPA Concern threshold (20 ppb) and sometimes breach the EPA Limit (30 ppb), indicating environmental risks. Between shutdowns, heavy metal levels hover at a lower baseline. This constitutes further evidence that the spikes are event-driven, not due to gradual accumulation.

Now we can look at whether or how these contamination patterns affected the lake's fish populations.


# Tracking Fish Populations


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
<!--
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
-->


<h3 style="margin: 0 0 4px 0; font-size: 18px;">Fish Population Decline at Clearwater Lake</h3>
<p style="margin: 0 0 16px 0; font-size: 13px; color: #666;">Overall trend (2023–2024) and breakdown by monitoring station (2024)</p>

```js
const fishTotals = Array.from(
  d3.rollup(
    fishSurveys,
    v => d3.sum(v, d => d.count),
    d => d.date,
    d => d.species
  ),
  ([date, speciesMap]) => Array.from(speciesMap, ([species, count]) => ({
    date,
    species,
    count
  }))
).flat()
```


```js
Plot.plot({
  subtitle: "Lake-wide total across all stations",
  width: 900,
  height: 280,
  x: {label: null},
  y: {grid: true, label: "Total Count"},
  color: {
    legend: true,
    domain: ["Trout", "Bass", "Carp"],
    range: ["#ec4899", "#7c3aed", "#6b7280"]
  },
  marks: [
    Plot.lineY(fishTotals, {x: "date", y: "count", stroke: "species", strokeWidth: 2}),
    Plot.dot(fishTotals, {
      x: "date",
      y: "count",
      fill: "species",
      r: 4,
      channels: {
        Species: "species",
        Date: d => d.date.toLocaleDateString("en-US", {month: "short", day: "numeric", year: "numeric"}),
        Count: "count"
      },
      tip: {
        format: {
          x: false,
          y: false,
          fill: false,
          Species: true,
          Date: true,
          Count: true
        }
      }
    })
  ]
})
```

```js
Plot.plot({
  subtitle: "By station (2024) — note the steeper decline at West Station",
  width: 900,
  height: 250,
  x: {label: null},
  y: {grid: true, label: "Count", domain: [0, 70]},
  fx: {label: null},
  color: {
    legend: false,
    domain: ["Trout", "Bass", "Carp"],
    range: ["#ec4899", "#7c3aed", "#6b7280"]
  },
  marks: [
    Plot.frame(),
    Plot.rect([{station_id: "West"}], {
      fx: "station_id",
      x1: d3.min(fishSurveys, d => d.date),
      x2: d3.max(fishSurveys, d => d.date),
      y1: 0,
      y2: 70,
      stroke: "#000000ff",
      strokeWidth: 3,
      fill: "none"
    }),
    Plot.lineY(fishSurveys, {x: "date", y: "count", stroke: "species", strokeWidth: 2, fx: "station_id"}),
    Plot.dot(fishSurveys, {
      x: "date",
      y: "count",
      fill: "species",
      r: 2,
      fx: "station_id",
      channels: {
        Species: "species",
        Date: d => d.date.toLocaleDateString("en-US", {month: "short", day: "numeric", year: "numeric"}),
        Count: "count"
      },
      tip: {
        format: {
          x: false,
          y: false,
          fill: false,
          fx: false,
          Species: true,
          Date: true,
          Count: true
        }
      }
    })
  ]
})
```
These paired charts reveal a lake-wide decline in fish populations over 2023–2024, with all two of the three species (Trout, Bass) showing downward trends — but the station breakdown exposes where the damage is worst. While East, North, and South stations show relatively stable or gradual declines, West Station stands out with the steepest drop, particularly for Trout (the most pollution-sensitive species). This geographic pattern aligns directly with the heavy metal contamination data: West Station sits nearest to ChemTech Manufacturing, and the fish populations there are collapsing fastest. 


<!--
```js
Plot.plot({
  title: "Fish Population Over Time — West Station",
  width: 900,
  height: 400,
  x: {label: null},
  y: {grid: true, label: "Count"},
  color: {
    legend: true,
    domain: ["Trout", "Bass", "Carp"],
    range: ["#7c3aed", "#ec4899", "#6b7280"]
  },
  marks: [
    Plot.lineY(
      fishSurveys.filter(d => d.station_id === "West"),
      {x: "date", y: "count", stroke: "species", strokeWidth: 2}
    ),
    Plot.dot(
      fishSurveys.filter(d => d.station_id === "West"),
      {x: "date", y: "count", fill: "species", r: 4, tip: true}
    )
  ]
})
```
-->

<!--
```js
Plot.plot({ //bar plot - first draft
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
-->


<h3 style="margin: 0 0 4px 0; font-size: 18px;">Trout, Bass, and Carp at West Station</h3>

```js
const selectedSpecies = view(Inputs.select(["All", "Trout", "Bass", "Carp"], {label: "Highlight species"}));
```

```js
Plot.plot({
  width: 900,
  height: 400,
  marginLeft: 50,
  color: {
    domain: ["Trout", "Bass", "Carp"],
    range: ["#ec4899", "#7c3aed", "#6b7280"],
    legend: true
  },
  fx: {label: null},
  y: {grid: true, label: "Count"},
  x: {axis: null, domain: ["Trout", "Bass", "Carp"]},
  marks: [
    Plot.ruleX(
      fishByQuarter.filter(d => d.station_id === "West"),
      Plot.groupX(
        {y: "mean"},
        {
          x: "species",
          y: "count",
          stroke: "species",
          strokeWidth: 3,
          fx: "quarter",
          opacity: selectedSpecies === "All" ? 1 : 0.2
        }
      )
    ),
    Plot.ruleX(
      fishByQuarter.filter(d => d.station_id === "West" && d.species === selectedSpecies),
      Plot.groupX(
        {y: "mean"},
        {
          x: "species",
          y: "count",
          stroke: "species",
          strokeWidth: 4,
          fx: "quarter"
        }
      )
    ),
    Plot.dot(
      fishByQuarter.filter(d => d.station_id === "West"),
      Plot.groupX(
        {y: "mean"},
        {
          x: "species",
          y: "count",
          fill: "species",
          r: 5,
          fx: "quarter",
          opacity: selectedSpecies === "All" ? 1 : 0.2
        }
      )
    ),
    Plot.dot(
      fishByQuarter.filter(d => d.station_id === "West" && d.species === selectedSpecies),
      Plot.groupX(
        {y: "mean"},
        {
          x: "species",
          y: "count",
          fill: "species",
          r: 7,
          fx: "quarter"
        }
      )
    ),
    Plot.tip(
      fishByQuarter.filter(d => d.station_id === "West"),
      Plot.pointer(
        Plot.groupX(
          {y: "mean"},
          {
            x: "species",
            y: "count",
            fx: "quarter",
            format: {
              x: false,
              fx: false,
              y: d => `${Math.round(d)}`
            }
          }
        )
      )
    ),
    Plot.ruleY([0])
  ]
})
```


<!--

```js
Plot.plot({
  title: "Trout, Bass, and Carp at West Station",
  width: 900,
  height: 400,
  x: {label: null},
  y: {grid: true, label: "Count"},
  fx: {label: null},
  color: {
    legend: true,
    domain: ["Trout", "Bass", "Carp"],
    range: ["#ef8cd5ff", "#be7febff", "#a9a9b4ff"]
  },
  marks: [
    Plot.barY(
      fishByQuarter.filter(d => d.station_id === "West"),
      Plot.groupX(
        {y: "sum"},
        {x: "species", y: "count", fill: "species", fx: "quarter"}
      )
    )
  ]
})
```
```js
Plot.plot({
  title: "Trout, Bass, and Carp at West Station",
  width: 900,
  height: 400,
  marginLeft: 50,
  color: {
    domain: ["Trout", "Bass", "Carp"],
    range: ["#ef8cd5ff", "#be7febff", "#a9a9b4ff"],
    legend: true
  },
  fx: {label: null},
  y: {grid: true, label: "Count"},
  x: {axis: null, domain: ["Trout", "Bass", "Carp"]},
  marks: [
    Plot.ruleX(
      fishByQuarter.filter(d => d.station_id === "West"),
      Plot.groupX(
        {y: "mean"},
        {x: "species", y: "count", stroke: "species", strokeWidth: 3, fx: "quarter"}
      )
    ),
    Plot.dot(
      fishByQuarter.filter(d => d.station_id === "West"),
      Plot.groupX(
        {y: "mean"},
        {x: "species", y: "count", fill: "species", r: 5, fx: "quarter"}
      )
    ),
    Plot.ruleY([0])
  ]
})
```
--> 

We see trout count at the West Station dropping from 43 down to 13 while Bass fall from 60 to 37. Both are declining, but the Trout's collapse is more dramatic — exactly what we would expect with heavy metal pollution affecting sensitive species more severely.

While Trout and Bass show clear downward trends, Carp — a species known for higer heavy-metal tolerane — actually increases slightly from 23 to 31. This divergence is another ecological signature of pollution: sensitive species decline while tolerant species fill the gap.

-----

# Evidence Summary Pointing to ChemTech

Geographic correlation: The West monitoring station, located just 800m from ChemTech Manufacturing, shows consistently elevated heavy metal levels (17% above lake average).

Temporal correlation: Heavy metal concentrations spike dramatically during ChemTech's quarterly "maintenance shutdowns" — rising from 12 ppb baseline to peaks of 23–49 ppb.

Biological indicator: Pollution-sensitive Trout populations at West station have crashed 70% while pollution-tolerant Carp have increased 35% — a classic signature of heavy metal contamination.

<!--
## ADDENDUM: South Station Cross-check

Because we observed repeated drops in trout population at the South Station, we wanted take a closer look.



```js
Plot.plot({
  title: "West Heavy Metals vs South Trout & Bass (Normalized)",
  width: 900,
  height: 400,
  x: {label: null},
  y: {grid: true, label: "Normalized Value (0-1)"},
  color: {legend: true},
  marks: [
    Plot.lineY(
      waterLong.filter(d => d.pollutant === "heavy_metals_ppb" && d.station_id === "West"),
      {x: "date", y: d => (d.value - 8) / (50 - 8), stroke: "#264653", strokeWidth: 2}
    ),
    Plot.lineY(
      fishSurveys.filter(d => d.station_id === "South" && d.species === "Trout"),
      {x: "date", y: d => (d.count - 10) / (70 - 10), stroke: "#7c3aed", strokeWidth: 2}
    ),
    Plot.lineY(
      fishSurveys.filter(d => d.station_id === "South" && d.species === "Bass"),
      {x: "date", y: d => (d.count - 10) / (70 - 10), stroke: "#ec4899", strokeWidth: 2}
    ),
    Plot.text([{x: new Date("2024-12-01"), y: 0.95, label: "West Heavy Metals"}], {x: "x", y: "y", text: "label", fill: "#264653", fontSize: 11}),
    Plot.text([{x: new Date("2024-12-01"), y: 0.85, label: "South Trout"}], {x: "x", y: "y", text: "label", fill: "#7c3aed", fontSize: 11}),
    Plot.text([{x: new Date("2024-12-01"), y: 0.75, label: "South Bass"}], {x: "x", y: "y", text: "label", fill: "#ec4899", fontSize: 11})
  ]
})
```
This shows how the heavy metals spike at West correlates with fish populations at South (the control station). If South fish are relatively stable while West fish decline, that strengthens the case that the pollution is localized.

-->


