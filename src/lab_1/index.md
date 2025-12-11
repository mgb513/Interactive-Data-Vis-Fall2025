---
title: "Lab 1: Passing Pollinators"
toc: true
---

This page is where you can iterate. Follow the lab instructions in the [readme.md](./README.md).

```js
const pollinator = await FileAttachment ("./data/pollinator_activity_data.csv").
csv({ typed: true })
display(pollinator)
```
-------------------------------

# Pollinator Activity Report

## June 2024 Pollinator Data Analysis


In respone to your bee-related inquiries we have found the following:

<h2> Question 1 </h2>
<p>What is the body mass and wing span distrbution of each pollinator species observed?</p>


```js
Plot.plot({
  title: "Bee Body Size: Wing Span vs Body Mass",
  grid: true,
  x: { label: "Average Wing Span (mm)" },
  y: { label: "Average Body Mass (g)" },
  color: { 
    domain: ["Honeybee", "Bumblebee", "Carpenter Bee"],
    range: ["gold", "orange", "saddlebrown"],
    legend: true, 
    label: "Pollinator Species"
  },
  marks: [
    Plot.dot(pollinator, {
      x: "avg_wing_span_mm",
      y: "avg_body_mass_g",
      stroke: "pollinator_species",
      fill: "pollinator_species",
      r: 5,
      opacity: 0.7,
      tip: true
    })
  ]
})
```

The three bee species measured appear in very distinct clusters or groupings. Wing span and body mass are correlated within each group, with carpenter bees having the highest mass and widest wingspan, and honeybees showing the lowest.


<!-- ```js
    const colors = ["#e41a1c", "#377eb8", "#4daf4a"];
    Plot.plot({
  marks: [
    Plot.ruleY([0]),
    Plot.barY(pollinator, {
      x: "flower_species",
      y: "nectar_production",
      sort: {x: "y", reverse: true},
      fill: (d, i) => colors[i % 3]
    }),
// Tip that appears on hover (pointer makes it respond to nearest item)
    Plot.tip(
      pollinator,
      Plot.pointer({
        x: "flower_species",
        y: "nectar_production",
        // channels shown in the tip — you can customize labels/format
        title: d => d.flower_species,
        // show value labeled as "Nectar" (optional)
        channels: { Nectar: d => d.nectar_production }
      })
    )
  ]
})
``` -->


<h2> Question 2</h2>
<p>What is the ideal weather for pollinating?</p>


Based on the visit count per day in June of 2024, we can find the days on which the most visits occurred. One way would be to look at weather conditions on these high-visit days. 

```js
Plot.plot({
  title: "Daily Pollinator Activity by Weather - June 2024",
  x: { label: "Date", tickRotate: -45 },
  y: { label: "Total Visits" },
  color: {
    domain: ["Cloudy", "Partly Cloudy", "Sunny"],
    range: ["grey", "steelblue", "orange"],
    legend: true,
    label: "Weather"
  },
  marks: [
    Plot.ruleY([0]),
    Plot.barY(pollinator,
      Plot.groupX(
        { y: "sum", fill: "mode" }, // most common weather that day
        {
          x: "date",
          y: "visit_count",
          fill: "weather_condition",
          tip: true
        }
      )
    )
  ]
})
```
After looking at the relationship between cloudcover and visit count throughout June, it is notciable that sunny and cloudy days both have high-activity days. Although cloudy conditions lead to more visits, there does not seem to be a strong collreation between cloud coverage and activity. 

We can now look at other aspects contributing the weather condition --temperature, wind, humidity-- separately to ascertain their influence on pollinator activity.

<!--
```js
Plot.plot({
  title: "Pollinator Visits by Weather Condition",
  y: { label: "Total Visit Count" },
  x: { label: "Weather Condition" },
  color: { 
    domain: ["Cloudy", "Partly Cloudy", "Sunny"],
    range: ["grey", "steelblue", "orange"]
  },
  marks: [
    Plot.ruleY([0]),
    Plot.ruleY([450, 550, 650], {
      stroke: "#bbb",
      strokeWidth: 1,
      strokeDasharray: "4 4"
    }),
    Plot.barY(pollinator, 
      Plot.groupX(
        { y: "sum" },
        {
          x: "weather_condition",
          y: "visit_count",
          fill: "weather_condition", // color by weather
          sort: { x: "-y" },
          tip: true
        }
      )
    ),
    Plot.text(pollinator,
      Plot.groupX(
        { y: "sum", text: "sum" },
        {
          x: "weather_condition",
          y: "visit_count",
          text: "visit_count",
          dy: -8,
          fontSize: 12
        }
      )
    )
  ]
})
```

```js
Plot.plot({
  title: "Optimal Weather Conditions for Pollinator Activity",
  x: { label: "Temperature (°C)" },
  y: { label: "Humidity (%)" },
  color: { scheme: "YlGnBu", legend: true, label: "Avg Visits" },
  marks: [
    Plot.rect(pollinator,
      Plot.bin(
        { fill: "mean" },
        {
          x: "temperature",
          y: "humidity",
          fill: "visit_count",
          thresholds: 5, // creates 5x5 grid
          tip: true
        }
      )
    )
  ]
})
```
-->

```js
Plot.plot({
  title: "Humidity vs Pollinator Activity",
  grid: true,
  x: { label: "Humidity (%)" },
  y: { label: "Visit Count" },
  color: { 
    domain: ["Cloudy", "Partly Cloudy", "Sunny"],
    range: ["grey", "steelblue", "orange"],
    legend: true, 
    label: "Weather" 
  },
  marks: [
    Plot.dot(pollinator, {
      x: "humidity",
      y: "visit_count",
      stroke: "weather_condition",
      fill: "weather_condition",
      opacity: 0.6,
      tip: true
    })
  ]
})
```

```js
Plot.plot({
  title: "Temperature vs Pollinator Activity",
  grid: true,
  x: { label: "Temperature (°C)" },
  y: { label: "Visit Count" },
  color: { 
    domain: ["Cloudy", "Partly Cloudy", "Sunny"],
    range: ["grey", "steelblue", "orange"],
    legend: true, 
    label: "Weather" 
  },
  marks: [
    Plot.dot(pollinator, {
      x: "temperature",
      y: "visit_count",
      stroke: "weather_condition",
      fill: "weather_condition",
      opacity: 0.6,
      tip: true
    })
  ]
})
```

```js
Plot.plot({
  title: "Wind Speed vs Pollinator Activity",
  grid: true,
  x: { label: "Wind Speed (km/h)" },
  y: { label: "Visit Count" },
  color: { 
    domain: ["Cloudy", "Partly Cloudy", "Sunny"],
    range: ["grey", "steelblue", "orange"],
    legend: true, 
    label: "Weather" 
  },
  marks: [
    Plot.dot(pollinator, {
      x: "wind_speed",
      y: "visit_count",
      stroke: "weather_condition",
      fill: "weather_condition",
      opacity: 0.6,
      tip: true
    })
  ]
})
```

While humidity seems to have the least influence on the visit count, we see that wind speed and temperature most clearly influenced the visit count. <b>Higher temperatures and lower wind-speeds constituted more ideal conditions for pollinating.</b>



<h2> Question 3 </h2>
<p>Which flower has the most nectar production?</p>

```js
Plot.plot({
  title: "Nectar Production by Flower Species", //addressing the pseudo bar
  y: { domain: [0, 100], label: "Nectar Production (mg)" },
  x: { label: "Flower Species" },
  marks: [
    Plot.ruleY([0]),
    Plot.ruleY([60, 70, 80], {
      stroke: "#bbb",
      strokeWidth: 1,
      strokeDasharray: "4 4"
    }),
    Plot.barY(pollinator, 
      Plot.groupX(
        { y: "sum" },
        {
          x: "flower_species",
          y: "nectar_production",
          fill: "purple",
          sort: { x: "-y" },
      
        }
      )
    ),
    Plot.text(pollinator,
      Plot.groupX(
        { y: "sum", text: "sum" },
        {
          x: "flower_species",
          y: "nectar_production",
          text: "nectar_production",
          dy: -8,
          fontSize: 12,
          fontWeight: "bold"
        }
      )
    )
  ]
})
```

<!--
```js
Plot.plot({
  marks: [
    Plot.ruleY([0]),
    Plot.barY(pollinator, {x: "flower_species", y: "nectar_production", sort: {x: "y", reverse: true}})
  ]
})
```


```js
Plot.plot({
     y: { domain: [0, 100] }, // here I know what I am doing. The version below is a CHatGPT verison
  marks: [
    Plot.ruleY([0, 60, 70, 80]),
    Plot.barY(pollinator, {
      x: "flower_species",
      y: "nectar_production",
      sort: {x: "y", reverse: true},
      fill: "steelblue"         
    })
  ]
})
```
-->

<p> This bar graph shows that during June 2024 sunflowers produced the most nectar on the farm. The y-axis shows the accumulated microlitres for the month.
All flowers produced over 60 microlitres. Sunflowers produces over 80 microlitres. </p>





