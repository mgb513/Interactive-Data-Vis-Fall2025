---
title: "Lab 1: Passing Pollinators"
toc: true
---

This page is where you can iterate. Follow the lab instructions in the [readme.md](./README.md).

# Pollinator Activity Report

## June 2024 Pollinator Data Analysis

<p>NOTE: <i>I have left several versions of graphs in this dashboard as opposed to cleaning the whole thing up. I still have some trouble with annotations and am not sure how to keep thing in/vsible. So I erred on the side of visibility. I am aware that I am probably moving slower than others. I hope to catch up. </i></p>

<p>In respones to your bee-related inquiries we have found the following:</p>

<h2> Question 1 </h2>
<p>What is the body mass and wing span distrbution of each pollinator species observed?</p>

```js
const pollinator = await FileAttachment ("./data/pollinator_activity_data.csv").
csv({ typed: true })
display(pollinator)
```

```js
Plot.plot({
  grid: true,
  x: {label: "avg_wing_span_mm"},
  y: {label: "avg_body_mass_g"},
  symbol: {legend: true},
  marks: [
    Plot.dot(pollinator, {x: "avg_wing_span_mm", y: "avg_body_mass_g", stroke: "pollinator_species", symbol: "pollinator_species"})
  ]
})
```

```js
Plot.dot(pollinator, {x: "avg_wing_span_mm", y: "avg_body_mass_g", stroke: "pollinator_species", tip: true }).plot()
```
<p> Honeybee = red; Bumblebee = blue; Carpenter Bee = yellow </p>

```js
Plot.plot({
  grid: true,
  x: {label: "avg_body_mass_g"},
  y: {label: "avg_wing_span_mm"},
  symbol: {legend: true},
  marks: [
    Plot.dot(pollinator, {x: "avg_body_mass_g", y: "avg_wing_span_mm", stroke: "pollinator_species", symbol: "pollinator_species"})
   ]
})
```
<p> The three species measured appear in very distinct clusters or groupings. Wing span and body mass are correlated within each group."

<h2> Question 3 </h2>
<p>Which flower has the most nectar production?</p>

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

<p> This bar graph shows that during June 2024 sunflowers produced the most nectar on the farm. The y-axis shows the accumulated microlitres for the month.
All flowers produced over 60 microlitres. Sunflowers produces over 80 microlitres. </p>

```js
Plot.plot({
  y: { domain: [0, 100] }, // extend y-axis to 100
  marks: [
    // baseline
    Plot.ruleY([0]),

    // horizontal rulers at 60, 70, 80 (styled)
    Plot.ruleY([60, 70, 80], {
      stroke: "#bbb",
      strokeWidth: 1,
      strokeDasharray: "4 4"
    }),

    // bars (data unchanged)
    Plot.barY(pollinator, {
      x: "flower_species",
      y: "nectar_production",
      sort: { x: "y", reverse: true },
      fill: "steelblue"
    })
  ]
})
```

```js
const colors = ["#9e2d1b", "#d99102", "#b0b370"];
Plot.plot({
  marks: [
    Plot.ruleY([0]),
    Plot.barY(pollinator, {
      x: "flower_species",
      y: "nectar_production",
      sort: {x: "y", reverse: true},
      fill: d => +d.nectar_production > 1.0 ? colors[0]
                 : +d.nectar_production > 0.7 ? colors[1]
                 : colors[2]
    }),
    // show a nicer floating tooltip on hover
    Plot.tip(
      pollinator,
      Plot.pointer({
        x: "flower_species",
        y: "nectar_production",
        title: d => d.flower_species,
        channels: { "Nectar (µL)": d => (+d.nectar_production).toFixed(2) }
      })
    )
  ]
})
```

```js
Plot.plot({
  marks: [
    Plot.ruleY([0]),
    Plot.barY(pollinator, {
      x: "flower_species",
      y: "nectar_production",
      sort: {x: "y", reverse: true},
      fill: (d, i) => ["#241ae4ff", "#3780b8ff", "#4aafacff"][i % 3], // alternating 3 colors
    tip: true})
  ]
})
```
<p><i> This looks cool. And I had a plan. Namely to make distisnct colcors for nectar produced per flower in three increments. 
But I couldn't get there. And I am not sure why the stuff below isn't invisible.</i><p>

// ```js
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
```
```js 
Plot.plot({
    //width: 300,
    //height: 300,
	marks:[
        //Plot.frame(), 
	    Plot.dot(pollinator, {
            x: "pollinator_species",
	        y: "avg_body_mass_g”,
	    //Stroke: “species”,
	    //Fill: “island”,
	         tip: true
	        })
	]
})
```

<h2> Question 2</h2>

```js
Plot.plot({
  marks: [
    Plot.ruleY([0,65]),
    Plot.barY(pollinator, {x: "date", y: "visit_count"})
  ]
})
```
<p> Based on the visit count per day in June of 2024, we can find the days on which the most visits occurred. One way would be to look at weather conditions on 
these high-visit days. Ultimately, I have chosen another way.
<i>BUT: wait a minute. I am now no longer sure what visit_count means. I am assuming it means bees visiting flowers.</i></p>

```js
Plot.plot({
  marks: [
    Plot.ruleY([0,450,550,650]),
    Plot.barY(pollinator, {x: "weather_condition", y: "visit_count"})
  ]
})
```
<p> Looking at the data by weather condition alone, we notice that cloudy conditions lead to more visits. However, when seen in conjunction with other atmospheric conditions, cloud-cover becomes less salient. </p>

```js
Plot.dot(pollinator, {x: "temperature", y: "visit_count", stroke: "weather_condition", tip: true }).plot()
```

```js
Plot.dot(pollinator, {x: "wind_speed", y: "visit_count", stroke: "weather_condition", tip: true }).plot()
```

<p>We see that wind speed and temperature were influenced the visit count. Higher temperatures and lower wind-speeds constituted more ideal conditions for pollinating.</p>

```js
Plot.dot(pollinator, {x: "humidity", y: "visit_count", stroke: "weather_condition", tip: true }).plot()
```

<p>Finally, humidity seems to have the least influence on the visit count, as density and varios humidity levels seems consistenly disributed. </p>

