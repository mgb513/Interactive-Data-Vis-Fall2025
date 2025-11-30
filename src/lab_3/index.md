---
title: "Lab 3: Mayoral Mystery"
toc: false
---

This page is where you can iterate. Follow the lab instructions in the [readme.md](./README.md).

<!-- Import Data -->
```js
const nyc = await FileAttachment("data/nyc.json").json();
const results = await FileAttachment("data/election_results.csv").csv({ typed: true });
const survey = await FileAttachment("data/survey_responses.csv").csv({ typed: true });
const events = await FileAttachment("data/campaign_events.csv").csv({ typed: true });

// NYC geoJSON data
display(nyc)
// Campaign data (first 10 objects)
display(results.slice(0,10))
display(survey.slice(0,10))
display(events.slice(0,10))
```


```js
// The nyc file is saved in data as a topoJSON instead of a geoJSON. Thats primarily for size reasons -- it saves us 3MB of data. For Plot to render it, we have to convert it back to its geoJSON feature collection. 
const districts = topojson.feature(nyc, nyc.objects.districts)
display(districts)
```

```js
// Simple rendering of the NYC districts topoJSON
Plot.plot({
  // this projection is already zoomed into NYC
  projection: {
    domain: districts,
    type: "mercator",
  },
  marks: [
    Plot.geo(districts),
  ]
})
```

Getting a sense of the candidates overall performance:

```js
const district_performance = (() => {
  const boroughs = {
    "1": "Manhattan",
    "2": "Bronx",
    "3": "Brooklyn",
    "4": "Queens",
    "5": "Staten Island"
  };
  
  return results
    .map(d => {
      const boro_code = String(d.boro_cd).charAt(0);
      const cd_number = String(d.boro_cd).slice(1);
      return {
        boro_cd: d.boro_cd,
        district_label: `${boroughs[boro_code]} CD-${parseInt(cd_number)}`,
        income_category: d.income_category,
        median_household_income: d.median_household_income,
        votes_candidate: d.votes_candidate,
        votes_opponent: d.votes_opponent,
        total_votes: d.votes_candidate + d.votes_opponent,
        vote_share: d.votes_candidate / (d.votes_candidate + d.votes_opponent),
        total_registered_voters: d.total_registered_voters,
        turnout_rate: d.turnout_rate,
        gotv_doors_knocked: d.gotv_doors_knocked,
        candidate_hours_spent: d.candidate_hours_spent
      };
    })
    .sort((a, b) => b.vote_share - a.vote_share);
})();
```

```js
Plot.plot({
  title: "Candidate Vote Share by District",
  subtitle: "Income levels: Low (<$50K) · Middle ($50K–$100K) · High (>$100K)",
  marginLeft: 140,
  marginRight: 50,
  height: district_performance.length * 22,
  x: {
    domain: [0, 1],
    label: "Vote Share",
    tickFormat: ".0%"
  },
  y: {
    label: "Community District"
  },
  color: {
    domain: ["Low", "Middle", "High"],
    range: ["#e15759", "#76b7b2", "#4e79a7"],
    legend: true
  },
  marks: [
    Plot.ruleX([0.5], { stroke: "#888", strokeDasharray: "4,4" }),
    Plot.barX(district_performance, {
      x: "vote_share",
      y: "district_label",
      fill: "income_category",
      sort: { y: "-x" },
      tip: true,
      title: d => `${d.district_label}\nVote Share: ${(d.vote_share * 100).toFixed(1)}%\nCandidate: ${d.votes_candidate.toLocaleString()} votes\nOpponent: ${d.votes_opponent.toLocaleString()} votes\nIncome Level: ${d.income_category}`
    }),
    Plot.text(district_performance, {
      x: "vote_share",
      y: "district_label",
      text: d => `${(d.vote_share * 100).toFixed(1)}%`,
      dx: 4,
      textAnchor: "start"
    })
  ]
})
```
a bar chart grouped by borough

```js
const borough_income_performance = (() => {
  const boroughs = {
    "1": "Manhattan",
    "2": "Bronx",
    "3": "Brooklyn",
    "4": "Queens",
    "5": "Staten Island"
  };
  
  const grouped = d3.rollup(
    results,
    v => ({
      votes_candidate: d3.sum(v, d => d.votes_candidate),
      votes_opponent: d3.sum(v, d => d.votes_opponent)
    }),
    d => String(d.boro_cd).charAt(0),
    d => d.income_category
  );
  
  const data = [];
  for (const [boro_code, incomeMap] of grouped) {
    for (const [income_category, votes] of incomeMap) {
      data.push({
        borough: boroughs[boro_code],
        income_category: income_category,
        votes_candidate: votes.votes_candidate,
        votes_opponent: votes.votes_opponent,
        total_votes: votes.votes_candidate + votes.votes_opponent,
        vote_share: votes.votes_candidate / (votes.votes_candidate + votes.votes_opponent)
      });
    }
  }
  
  return data;
})();
```

```js
Plot.plot({
  title: "Candidate Vote Share by Borough and Income Level",
  subtitle: "Income levels: Low (<$50K) · Middle ($50K–$100K) · High (>$100K)",
  marginLeft: 50,
  marginBottom: 80,
  height: 400,
  x: {
    label: null,
    tickRotate: -45
  },
  y: {
    label: "Total Votes",
    tickFormat: "s"
  },
  color: {
    domain: ["Low", "Middle", "High"],
    range: ["#c7e9c0", "#41ab5d", "#00441b"],
    legend: true
  },
  marks: [
    Plot.barY(borough_income_performance, {
      x: "borough",
      y: "votes_candidate",
      fill: "income_category",
      sort: { x: "-y" },
      tip: true,
      title: d => `${d.borough} (${d.income_category} income)\nCandidate: ${d.votes_candidate.toLocaleString()} votes\nOpponent: ${d.votes_opponent.toLocaleString()} votes\nVote Share: ${(d.vote_share * 100).toFixed(1)}%`
    })
  ]
})
```

## Choropleth attempt:

```js
const districts_with_results = (() => {
  const resultsMap = new Map(district_performance.map(d => [d.boro_cd, d]));
  
  return {
    type: "FeatureCollection",
    features: districts.features.map(f => ({
      ...f,
      properties: {
        ...f.properties,
        ...resultsMap.get(f.properties.BoroCD)
      }
    }))
  };
})();
```

This map shows the voteshare the candidate received in each voting distrisct.

```js
districts_with_results.features.map(f => ({
  BoroCD: f.properties.BoroCD,
  vote_share: f.properties.vote_share
}))
```

```js
Plot.plot({
  title: "Candidate Vote Share by Community District",
  subtitle: "Darker blue = higher vote share for candidate",
  projection: {
    type: "mercator",
    domain: districts_with_results
  },
  width: 800,
  height: 800,
  color: {
    scheme: "blues",
    label: "Vote Share",
    legend: true,
    tickFormat: ".0%"
  },
  marks: [
    Plot.geo(districts_with_results, {
      fill: d => d.properties.vote_share,
      stroke: "white",
      strokeWidth: 0.5,
      tip: true,
      title: d => `${d.properties.district_label}\nVotes for Candidate: ${d.properties.votes_candidate?.toLocaleString()}\nVote Share: ${(d.properties.vote_share * 100).toFixed(1)}%\nIncome Level: ${d.properties.income_category}`
    })
  ]
})
```
```js
Plot.plot({
  title: "Community Districts by Income Level",
  subtitle: "Income levels: Low (<$50K) · Middle ($50K–$100K) · High (>$100K)",
  projection: {
    type: "mercator",
    domain: districts_with_results
  },
  width: 800,
  height: 800,
  color: {
    domain: ["Low", "Middle", "High"],
    range: ["#c7e9c0", "#41ab5d", "#00441b"],
    legend: true
  },
  marks: [
    Plot.geo(districts_with_results, {
      fill: d => d.properties.income_category,
      stroke: "white",
      strokeWidth: 0.5,
      tip: true,
      title: d => `${d.properties.district_label}\nIncome Level: ${d.properties.income_category}\nMedian Household Income: $${d.properties.median_household_income?.toLocaleString()}\nVote Share: ${(d.properties.vote_share * 100).toFixed(1)}%`
    })
  ]
})
```
trying for side by side cards:

```js
(() => {
  const voteShareMap = Plot.plot({
    title: "Vote Share by District",
    subtitle: "Darker blue = higher vote share",
    projection: { type: "mercator", domain: districts_with_results },
    width: 380,
    height: 400,
    color: { 
      scheme: "blues", 
      label: "Vote Share", 
      tickFormat: ".0%", 
      legend: true 
    },
    marks: [
      Plot.geo(districts_with_results, {
        fill: d => d.properties.vote_share,
        stroke: "white",
        strokeWidth: 0.5,
        tip: true,
        title: d => `${d.properties.district_label}\nVote Share: ${(d.properties.vote_share * 100).toFixed(1)}%`
      })
    ]
  });

  const incomeMap = Plot.plot({
    title: "Income Level by District",
    subtitle: "Low (<$50K) · Middle ($50K–$100K) · High (>$100K)",
    projection: { type: "mercator", domain: districts_with_results },
    width: 380,
    height: 400,
    color: { 
      domain: ["Low", "Middle", "High"], 
      range: ["#c7e9c0", "#41ab5d", "#00441b"], 
      legend: true 
    },
    marks: [
      Plot.geo(districts_with_results, {
        fill: d => d.properties.income_category,
        stroke: "white",
        strokeWidth: 0.5,
        tip: true,
        title: d => `${d.properties.district_label}\nIncome Level: ${d.properties.income_category}`
      })
    ]
  });

  const makeCard = (content) => {
    const card = document.createElement("div");
    card.style.backgroundColor = "white";
    card.style.borderRadius = "8px";
    card.style.boxShadow = "0 2px 8px rgba(0,0,0,0.1)";
    card.style.padding = "16px";
    card.appendChild(content);
    return card;
  };

  const container = document.createElement("div");
  container.style.display = "flex";
  container.style.gap = "24px";
  container.style.justifyContent = "center";
  container.appendChild(makeCard(voteShareMap));
  container.appendChild(makeCard(incomeMap));
  return container;
})()
```
```js
(() => {
  // Bivariate color scheme (3x3 grid)
  // Rows = income (Low, Middle, High)
  // Columns = vote share (Low, Mid, High)
  const bivarateColors = {
    "Low": { low: "#e8e8e8", mid: "#b5c0da", high: "#6c83b5" },
    "Middle": { low: "#b8d6be", mid: "#90b2b3", high: "#567994" },
    "High": { low: "#73ae80", mid: "#5a9178", high: "#2a5a5b" }
  };

  const getColor = (voteShare, income) => {
    const voteLevel = voteShare < 0.45 ? "low" : voteShare < 0.55 ? "mid" : "high";
    return bivarateColors[income]?.[voteLevel] || "#ccc";
  };

  const map = Plot.plot({
    title: "Vote Share and Income Level by District",
    projection: { type: "mercator", domain: districts_with_results },
    width: 700,
    height: 700,
    marks: [
      Plot.geo(districts_with_results, {
        fill: d => getColor(d.properties.vote_share, d.properties.income_category),
        stroke: "white",
        strokeWidth: 0.5,
        tip: true,
        title: d => `${d.properties.district_label}\nVote Share: ${(d.properties.vote_share * 100).toFixed(1)}%\nIncome Level: ${d.properties.income_category}`
      })
    ]
  });

  // Create bivariate legend
  const legendSize = 120;
  const cellSize = legendSize / 3;
  
  const legendSvg = document.createElementNS("http://www.w3.org/2000/svg", "svg");
  legendSvg.setAttribute("width", legendSize + 60);
  legendSvg.setAttribute("height", legendSize + 60);

  // Draw the 3x3 grid
  const incomes = ["High", "Middle", "Low"];
  const voteLevels = ["low", "mid", "high"];

  incomes.forEach((income, row) => {
    voteLevels.forEach((vote, col) => {
      const rect = document.createElementNS("http://www.w3.org/2000/svg", "rect");
      rect.setAttribute("x", col * cellSize + 40);
      rect.setAttribute("y", row * cellSize + 10);
      rect.setAttribute("width", cellSize);
      rect.setAttribute("height", cellSize);
      rect.setAttribute("fill", bivarateColors[income][vote]);
      rect.setAttribute("stroke", "white");
      rect.setAttribute("stroke-width", "1");
      legendSvg.appendChild(rect);
    });
  });

  // Add axis labels
  const addText = (text, x, y, rotate = 0) => {
    const t = document.createElementNS("http://www.w3.org/2000/svg", "text");
    t.setAttribute("x", x);
    t.setAttribute("y", y);
    t.setAttribute("font-size", "11");
    t.setAttribute("font-family", "sans-serif");
    t.setAttribute("text-anchor", "middle");
    if (rotate) t.setAttribute("transform", `rotate(${rotate}, ${x}, ${y})`);
    t.textContent = text;
    legendSvg.appendChild(t);
  };

  // X-axis label (Vote Share)
  addText("Vote Share →", 40 + legendSize / 2, legendSize + 35);
  addText("Low", 40 + cellSize / 2, legendSize + 22);
  addText("Mid", 40 + cellSize * 1.5, legendSize + 22);
  addText("High", 40 + cellSize * 2.5, legendSize + 22);

  // Y-axis label (Income)
  addText("Income →", 18, 10 + legendSize / 2, -90);
  addText("High", 30, 10 + cellSize / 2 + 4);
  addText("Mid", 30, 10 + cellSize * 1.5 + 4);
  addText("Low", 30, 10 + cellSize * 2.5 + 4);

  // Legend container
  const legendWrapper = document.createElement("div");
  legendWrapper.style.display = "flex";
  legendWrapper.style.flexDirection = "column";
  legendWrapper.style.alignItems = "center";
  legendWrapper.style.marginTop = "10px";
  
  const legendTitle = document.createElement("div");
  legendTitle.style.fontFamily = "sans-serif";
  legendTitle.style.fontSize = "12px";
  legendTitle.style.fontWeight = "bold";
  legendTitle.style.marginBottom = "8px";
  legendTitle.textContent = "Legend";
  
  legendWrapper.appendChild(legendTitle);
  legendWrapper.appendChild(legendSvg);

  // Combine map and legend
  const container = document.createElement("div");
  container.style.display = "flex";
  container.style.alignItems = "flex-start";
  container.style.gap = "20px";
  container.appendChild(map);
  container.appendChild(legendWrapper);

  return container;
})()
```
## 11-29-25

### OPPORTUNITY MAP

```js
(() => {
  const medianVoteShare = d3.median(district_performance, d => d.vote_share);
  const medianTurnout = d3.median(district_performance, d => d.turnout_rate);

  const opportunityDistricts = district_performance.filter(
    d => d.vote_share >= medianVoteShare && d.turnout_rate <= medianTurnout
  );

  const map = Plot.plot({
    title: "Opportunity Map: High Support, Low Turnout Districts",
    subtitle: "Orange districts had above-median vote share but below-median turnout—prime targets for next campaign",
    projection: { type: "mercator", domain: districts_with_results },
    width: 700,
    height: 700,
    marks: [
      Plot.geo(districts_with_results, {
        fill: d => {
          const voteShare = d.properties.vote_share;
          const turnout = d.properties.turnout_rate;
          if (voteShare >= medianVoteShare && turnout <= medianTurnout) {
            return "#e6550d";
          } else if (voteShare >= medianVoteShare) {
            return "#3182bd";
          } else if (turnout <= medianTurnout) {
            return "#fdae6b";
          } else {
            return "#d9d9d9";
          }
        },
        stroke: "white",
        strokeWidth: 0.5,
        tip: true,
        title: d => `${d.properties.district_label}\nVote Share: ${(d.properties.vote_share * 100).toFixed(1)}%\nTurnout: ${(d.properties.turnout_rate * 100).toFixed(1)}%\nRegistered Voters: ${d.properties.total_registered_voters?.toLocaleString()}`
      })
    ]
  });

  const legendData = [
    { color: "#e6550d", label: "High Opportunity (high support, low turnout)" },
    { color: "#3182bd", label: "Mobilized (high support, high turnout)" },
    { color: "#fdae6b", label: "Persuasion Target (low support, low turnout)" },
    { color: "#d9d9d9", label: "Low Priority (low support, high turnout)" }
  ];

  const legend = document.createElement("div");
  legend.style.marginTop = "16px";
  legend.style.fontFamily = "sans-serif";
  legend.style.fontSize = "12px";

  legendData.forEach(({ color, label }) => {
    const item = document.createElement("div");
    item.style.display = "flex";
    item.style.alignItems = "center";
    item.style.gap = "8px";
    item.style.marginBottom = "6px";

    const swatch = document.createElement("div");
    swatch.style.width = "20px";
    swatch.style.height = "20px";
    swatch.style.backgroundColor = color;
    swatch.style.borderRadius = "3px";

    const text = document.createElement("span");
    text.textContent = label;

    item.appendChild(swatch);
    item.appendChild(text);
    legend.appendChild(item);
  });

  const summary = document.createElement("div");
  summary.style.marginTop = "16px";
  summary.style.padding = "12px";
  summary.style.backgroundColor = "#fff3e0";
  summary.style.borderRadius = "6px";
  summary.style.fontFamily = "sans-serif";
  summary.style.fontSize = "13px";
  summary.innerHTML = `
    <strong>Key Finding:</strong> ${opportunityDistricts.length} districts have high support but low turnout.<br>
    <strong>Median Vote Share:</strong> ${(medianVoteShare * 100).toFixed(1)}%<br>
    <strong>Median Turnout:</strong> ${(medianTurnout * 100).toFixed(1)}%
  `;

  const container = document.createElement("div");
  container.appendChild(map);
  container.appendChild(legend);
  container.appendChild(summary);

  return container;
})()
```
### GOTV Effectiveness

```js
(() => {
  const scatterPlot = Plot.plot({
    title: "GOTV Effectiveness: Doors Knocked vs. Turnout Rate",
    subtitle: "Each circle is a district. Color = income level.",
    width: 700,
    height: 450,
    grid: true,
    x: {
      label: "Doors Knocked (GOTV Effort)",
      tickFormat: "s"
    },
    y: {
      label: "Turnout Rate",
      tickFormat: ".0%"
    },
    color: {
      domain: ["Low", "Middle", "High"],
      range: ["#c7e9c0", "#41ab5d", "#00441b"],
      legend: true
    },
    marks: [
      Plot.dot(district_performance, {
        x: "gotv_doors_knocked",
        y: "turnout_rate",
        r: 6,
        fill: "income_category",
        fillOpacity: 0.8,
        stroke: "#333",
        strokeWidth: 0.5,
        tip: true,
        title: d => `${d.district_label}\nDoors Knocked: ${d.gotv_doors_knocked?.toLocaleString()}\nTurnout: ${(d.turnout_rate * 100).toFixed(1)}%\nVote Share: ${(d.vote_share * 100).toFixed(1)}%`
      }),
      Plot.linearRegressionY(district_performance, {
        x: "gotv_doors_knocked",
        y: "turnout_rate",
        stroke: "#e15759",
        strokeWidth: 2,
        strokeDasharray: "6,4"
      })
    ]
  });

  const candidateHoursPlot = Plot.plot({
    title: "Candidate Hours Spent vs. Vote Share",
    subtitle: "Did personal campaigning translate to votes?",
    width: 700,
    height: 450,
    grid: true,
    x: {
      label: "Candidate Hours Spent in District"
    },
    y: {
      label: "Vote Share",
      tickFormat: ".0%"
    },
    color: {
      domain: ["Low", "Middle", "High"],
      range: ["#c7e9c0", "#41ab5d", "#00441b"],
      legend: true
    },
    marks: [
      Plot.dot(district_performance, {
        x: "candidate_hours_spent",
        y: "vote_share",
        r: 6,
        fill: "income_category",
        fillOpacity: 0.8,
        stroke: "#333",
        strokeWidth: 0.5,
        tip: true,
        title: d => `${d.district_label}\nCandidate Hours: ${d.candidate_hours_spent}\nVote Share: ${(d.vote_share * 100).toFixed(1)}%\nTurnout: ${(d.turnout_rate * 100).toFixed(1)}%`
      }),
      Plot.linearRegressionY(district_performance, {
        x: "candidate_hours_spent",
        y: "vote_share",
        stroke: "#e15759",
        strokeWidth: 2,
        strokeDasharray: "6,4"
      })
    ]
  });

  const correlation = (arr, xKey, yKey) => {
    const xMean = d3.mean(arr, d => d[xKey]);
    const yMean = d3.mean(arr, d => d[yKey]);
    const numerator = d3.sum(arr, d => (d[xKey] - xMean) * (d[yKey] - yMean));
    const denominator = Math.sqrt(
      d3.sum(arr, d => (d[xKey] - xMean) ** 2) *
      d3.sum(arr, d => (d[yKey] - yMean) ** 2)
    );
    return numerator / denominator;
  };

  const gotvCorr = correlation(district_performance, "gotv_doors_knocked", "turnout_rate");
  const hoursCorr = correlation(district_performance, "candidate_hours_spent", "vote_share");

  const insights = document.createElement("div");
  insights.style.marginTop = "20px";
  insights.style.padding = "16px";
  insights.style.backgroundColor = "#f0f7ff";
  insights.style.borderRadius = "8px";
  insights.style.fontFamily = "sans-serif";
  insights.style.fontSize = "14px";
  insights.innerHTML = `
    <h3 style="margin-top: 0;">GOTV Analysis Summary</h3>
    <p><strong>Doors Knocked ↔ Turnout Correlation:</strong> ${gotvCorr.toFixed(3)} 
      ${gotvCorr > 0.3 ? "✅ Positive relationship" : gotvCorr > 0 ? "↗️ Weak positive" : "⚠️ No clear relationship"}</p>
    <p><strong>Candidate Hours ↔ Vote Share Correlation:</strong> ${hoursCorr.toFixed(3)}
      ${hoursCorr > 0.3 ? "✅ Positive relationship" : hoursCorr > 0 ? "↗️ Weak positive" : "⚠️ No clear relationship"}</p>
    <p style="margin-bottom: 0; color: #666; font-size: 12px;">
      Correlation ranges from -1 to 1. Values above 0.3 suggest a meaningful positive relationship.
    </p>
  `;

  const container = document.createElement("div");
  container.appendChild(scatterPlot);
  container.appendChild(candidateHoursPlot);
  container.appendChild(insights);

  return container;
})()
```
```js
(() => {
  const boroughs = {
    "1": "Manhattan",
    "2": "Bronx",
    "3": "Brooklyn",
    "4": "Queens",
    "5": "Staten Island"
  };
  
  const dataWithBorough = district_performance.map(d => ({
    ...d,
    borough: boroughs[String(d.boro_cd).charAt(0)]
  }));

  const scatterPlot = Plot.plot({
    title: "GOTV Effectiveness: Doors Knocked vs. Turnout Rate",
    subtitle: "Faceted by borough",
    width: 950,
    height: 300,
    grid: true,
    marginLeft: 60,
    x: {
      label: "Doors Knocked",
      tickFormat: "s"
    },
    y: {
      label: "Turnout Rate",
      tickFormat: ".0%"
    },
    fx: {
      domain: ["Manhattan", "Bronx", "Brooklyn", "Queens", "Staten Island"],
      label: null
    },
    color: {
      domain: ["Low", "Middle", "High"],
      range: ["#c7e9c0", "#41ab5d", "#00441b"],
      legend: true
    },
    marks: [
      Plot.frame(),
      Plot.dot(dataWithBorough, {
        x: "gotv_doors_knocked",
        y: "turnout_rate",
        fx: "borough",
        r: 5,
        fill: "income_category",
        fillOpacity: 0.8,
        stroke: "#333",
        strokeWidth: 0.5,
        tip: true,
        title: d => `${d.district_label}\nDoors Knocked: ${d.gotv_doors_knocked?.toLocaleString()}\nTurnout: ${(d.turnout_rate * 100).toFixed(1)}%\nVote Share: ${(d.vote_share * 100).toFixed(1)}%`
      }),
      Plot.linearRegressionY(dataWithBorough, {
        x: "gotv_doors_knocked",
        y: "turnout_rate",
        fx: "borough",
        stroke: "#e15759",
        strokeWidth: 2,
        strokeDasharray: "6,4"
      })
    ]
  });

  const candidateHoursPlot = Plot.plot({
    title: "Candidate Hours Spent vs. Vote Share",
    subtitle: "Faceted by borough",
    width: 950,
    height: 300,
    grid: true,
    marginLeft: 60,
    x: {
      label: "Candidate Hours Spent"
    },
    y: {
      label: "Vote Share",
      tickFormat: ".0%"
    },
    fx: {
      domain: ["Manhattan", "Bronx", "Brooklyn", "Queens", "Staten Island"],
      label: null
    },
    color: {
      domain: ["Low", "Middle", "High"],
      range: ["#c7e9c0", "#41ab5d", "#00441b"],
      legend: true
    },
    marks: [
      Plot.frame(),
      Plot.dot(dataWithBorough, {
        x: "candidate_hours_spent",
        y: "vote_share",
        fx: "borough",
        r: 5,
        fill: "income_category",
        fillOpacity: 0.8,
        stroke: "#333",
        strokeWidth: 0.5,
        tip: true,
        title: d => `${d.district_label}\nCandidate Hours: ${d.candidate_hours_spent}\nVote Share: ${(d.vote_share * 100).toFixed(1)}%\nTurnout: ${(d.turnout_rate * 100).toFixed(1)}%`
      }),
      Plot.linearRegressionY(dataWithBorough, {
        x: "candidate_hours_spent",
        y: "vote_share",
        fx: "borough",
        stroke: "#e15759",
        strokeWidth: 2,
        strokeDasharray: "6,4"
      })
    ]
  });

  // Calculate correlation by borough
  const correlation = (arr, xKey, yKey) => {
    if (arr.length < 3) return null;
    const xMean = d3.mean(arr, d => d[xKey]);
    const yMean = d3.mean(arr, d => d[yKey]);
    const numerator = d3.sum(arr, d => (d[xKey] - xMean) * (d[yKey] - yMean));
    const denominator = Math.sqrt(
      d3.sum(arr, d => (d[xKey] - xMean) ** 2) *
      d3.sum(arr, d => (d[yKey] - yMean) ** 2)
    );
    return denominator === 0 ? null : numerator / denominator;
  };

  const boroNames = ["Manhattan", "Bronx", "Brooklyn", "Queens", "Staten Island"];
  
  const boroStats = boroNames.map(boro => {
    const boroData = dataWithBorough.filter(d => d.borough === boro);
    return {
      borough: boro,
      districts: boroData.length,
      gotvCorr: correlation(boroData, "gotv_doors_knocked", "turnout_rate"),
      hoursCorr: correlation(boroData, "candidate_hours_spent", "vote_share")
    };
  });

  const overallGotvCorr = correlation(dataWithBorough, "gotv_doors_knocked", "turnout_rate");
  const overallHoursCorr = correlation(dataWithBorough, "candidate_hours_spent", "vote_share");

  const formatCorr = (val) => {
    if (val === null) return "N/A";
    const emoji = val > 0.3 ? "✅" : val > 0 ? "↗️" : "⚠️";
    return `${val.toFixed(3)} ${emoji}`;
  };

  const insights = document.createElement("div");
  insights.style.marginTop = "20px";
  insights.style.padding = "16px";
  insights.style.backgroundColor = "#f0f7ff";
  insights.style.borderRadius = "8px";
  insights.style.fontFamily = "sans-serif";
  insights.style.fontSize = "13px";
  
  let tableRows = boroStats.map(b => `
    <tr>
      <td style="padding: 4px 12px; border-bottom: 1px solid #ddd;">${b.borough}</td>
      <td style="padding: 4px 12px; border-bottom: 1px solid #ddd; text-align: center;">${b.districts}</td>
      <td style="padding: 4px 12px; border-bottom: 1px solid #ddd; text-align: center;">${formatCorr(b.gotvCorr)}</td>
      <td style="padding: 4px 12px; border-bottom: 1px solid #ddd; text-align: center;">${formatCorr(b.hoursCorr)}</td>
    </tr>
  `).join("");

  insights.innerHTML = `
    <h3 style="margin-top: 0;">GOTV Analysis by Borough</h3>
    <table style="border-collapse: collapse; width: 100%; margin-bottom: 12px;">
      <thead>
        <tr style="background-color: #e0e0e0;">
          <th style="padding: 6px 12px; text-align: left;">Borough</th>
          <th style="padding: 6px 12px; text-align: center;">Districts</th>
          <th style="padding: 6px 12px; text-align: center;">Doors ↔ Turnout</th>
          <th style="padding: 6px 12px; text-align: center;">Hours ↔ Vote Share</th>
        </tr>
      </thead>
      <tbody>
        ${tableRows}
        <tr style="font-weight: bold; background-color: #f5f5f5;">
          <td style="padding: 4px 12px;">Overall</td>
          <td style="padding: 4px 12px; text-align: center;">${dataWithBorough.length}</td>
          <td style="padding: 4px 12px; text-align: center;">${formatCorr(overallGotvCorr)}</td>
          <td style="padding: 4px 12px; text-align: center;">${formatCorr(overallHoursCorr)}</td>
        </tr>
      </tbody>
    </table>
    <p style="margin-bottom: 0; color: #666; font-size: 11px;">
      Correlation ranges from -1 to 1. ✅ = strong positive (>0.3), ↗️ = weak positive, ⚠️ = no/negative relationship
    </p>
  `;

  const container = document.createElement("div");
  container.appendChild(scatterPlot);
  container.appendChild(candidateHoursPlot);
  container.appendChild(insights);

  return container;
})()
```

## SURVEY & EVENT DATA

```js
const policyDiff = [
  {policy: "Childcare Support", difference: 0.91, type: "strength"},
  {policy: "Public Transit", difference: 0.39, type: "strength"},
  {policy: "Small Business Tax", difference: 0.31, type: "strength"},
  {policy: "Affordable Housing", difference: 0.27, type: "strength"},
  {policy: "Police Reform", difference: 0.08, type: "neutral"}
];
```
```js
Plot.plot({
  marginLeft: 150,
  x: {label: "Higher alignment among candidate voters →", domain: [0, 1]},
  marks: [
    Plot.barX(policyDiff, {
      x: "difference",
      y: "policy",
      fill: d => d.difference > 0.5 ? "#16a34a" : d.difference > 0.2 ? "#3b82f6" : "#94a3b8",
      sort: {y: "-x"}
    }),
    Plot.text(policyDiff, {
      x: "difference",
      y: "policy",
      text: d => `+${d.difference.toFixed(2)}`,
      dx: 5,
      textAnchor: "start"
    }),
    Plot.ruleX([0.5], {stroke: "red", strokeDasharray: "4"}),
    Plot.text([{x: 0.5, y: 0}], {
      x: "x",
      y: "y",
      text: ["Strong differentiator →"],
      dy: -10,
      fill: "red"
    })
  ]
})
```

PRIORITY 3: RESOURCE ALLOCATION FAILURES 💰
Finding: Middle-income areas got 25 events with only 77 avg attendance (vs 208 in low-income)
Viz 3A: Resource Distribution by Income Category
Purpose: Show the middle-income gap
Impact: Critical - shows we abandoned a key voter segment

```js
const resourceData = [
  {income: "Low Income", events: 75, attendance: 15620, avg: 208},
  {income: "Middle Income", events: 25, attendance: 1935, avg: 77},
  {income: "High Income", events: 30, attendance: 4837, avg: 161}
];
```
```js
Plot.plot({
  title: "Campaign Events Over Time",
  x: {
    label: "Date (events per month)", 
    type: "time", 
    grid: true,
    tickFormat: (d) => {
      const monthKey = d3.timeMonth(d);
      const count = events.filter(e => 
        d3.timeMonth(new Date(e.event_date)).getTime() === monthKey.getTime()
      ).length;
      return `${d3.timeFormat("%b")(d)}\n(${count})`;
    }
  },
  y: {label: "Attendance"},
  color: {
    legend: true, 
    label: "Income Category",
    domain: ["Low", "Middle", "High"],
    range: ["#c7e9c0", "#41ab5d", "#00441b"]
  },
  marks: [
    Plot.dot(events, {
      x: d => new Date(d.event_date),
      y: "estimated_attendance",
      fill: "income_category",
      r: 6,
      tip: true
    })
  ]
})
```
```js
const monthlyData = d3.flatRollup(
  events,
  v => d3.sum(v, d => d.estimated_attendance),
  d => d3.timeFormat("%B")(new Date(d.event_date)),
  d => d.income_category
).map(([month, income, attendance]) => ({month, income, attendance}))
```

```js
html`<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 20px;">
  <div style="border: 1px solid #ccc; padding: 15px; border-radius: 8px;">
    ${Plot.plot({
      title: "All Events Over Time",
      x: {label: "Date", type: "time", grid: true},
      y: {label: "Attendance", grid: true},
      color: {
        legend: true,
        domain: ["Low", "Middle", "High"],
        range: ["#c7e9c0", "#41ab5d", "#00441b"]
      },
      marks: [
        Plot.dot(events, {
          x: d => new Date(d.event_date),
          y: "estimated_attendance",
          fill: "income_category",
          r: 6,
          tip: true
        })
      ]
    })}
  </div>
  
  <div style="border: 1px solid #ccc; padding: 15px; border-radius: 8px;">
    ${Plot.plot({
      title: "Events by Income Category",
      x: {label: "Date", type: "time", grid: true},
      y: {label: "Attendance", grid: true},
      fy: {label: null},
      color: {
        domain: ["Low", "Middle", "High"],
        range: ["#c7e9c0", "#41ab5d", "#00441b"]
      },
      marks: [
        Plot.dot(events, {
          x: d => new Date(d.event_date),
          y: "estimated_attendance",
          fill: "income_category",
          fy: "income_category",
          r: 6,
          tip: true
        })
      ]
    })}
  </div>
</div>`
```

PRIORITY 4: TIMELINE FAILURES ⏰
Finding: Campaign peaked in May (29 events), collapsed in November (3 events)
Viz 4A: Campaign Activity Timeline
Purpose: Show momentum collapsed before Election Day
Impact: Critical - shows strategic planning failure

```js
const timelineData = [
  {month: "May", events: 29, attendance: 5290},
  {month: "June", events: 17, attendance: 2680},
  {month: "July", events: 25, attendance: 5002},
  {month: "August", events: 20, attendance: 3348},
  {month: "September", events: 22, attendance: 3400},
  {month: "October", events: 14, attendance: 2042},
  {month: "November", events: 3, attendance: 630}
];
```

## Campaing Events between MAy and ELection day by attendance and income. 


```js
Plot.plot({
  title: "All Events Over Time",
  marginBottom: 100,
  fx: {
    padding: 0.1,
    label: "Month",
    domain: ["May", "June", "July", "August", "September", "October", "November"]
  },
  x: {
    axis: null, 
    paddingOuter: 0.2,
    domain: ["Low", "Middle", "High"]
  },
  y: {
    label: "Total Attendance", 
    grid: true,
    nice: true
  },
  color: {
    legend: true,
    domain: ["Low", "Middle", "High"],
    range: ["#c7e9c0", "#41ab5d", "#00441b"]
  },
  marks: [
    Plot.barY(events, Plot.groupX(
      {y: "sum", title: d => `Income: ${d[0].income_category}\nEvents: ${d.length}\nAttendance: ${d3.sum(d, r => r.estimated_attendance).toLocaleString()}`}, 
      {
        x: "income_category",
        fx: d => d.event_date.toLocaleDateString('en-US', {month: 'long', timeZone: 'UTC'}),
        y: "estimated_attendance",
        fill: "income_category",
        tip: true
      }
    )),
    Plot.text(events, Plot.groupX(
      {y: "sum", text: "count"},
      {
        x: "income_category",
        fx: d => d.event_date.toLocaleDateString('en-US', {month: 'long', timeZone: 'UTC'}),
        y: "estimated_attendance",
        dy: -5,
        fontSize: 11,
        fontWeight: "bold",
        fill: "#333"
      }
    )),
    Plot.ruleY([0]),
    Plot.frame({stroke: "#ccc", strokeWidth: 1})
  ]
})
```

```js
// Helper function to get borough name from boro_cd
const getBoroughName = (boro_cd) => {
  const digit = Math.floor(boro_cd / 100);
  const boroughs = {
    1: "Manhattan",
    2: "Bronx", 
    3: "Brooklyn",
    4: "Queens",
    5: "Staten Island"
  };
  return boroughs[digit];
};

// Calculate policy alignment differences by borough
const policyColumns = [
  {field: "affordable_housing_alignment", name: "Affordable Housing"},
  {field: "public_transit_alignment", name: "Public Transit"},
  {field: "childcare_support_alignment", name: "Childcare Support"},
  {field: "small_business_tax_alignment", name: "Small Business Tax"},
  {field: "police_reform_alignment", name: "Police Reform"}
];

const policyDiffByBorough = [];

// For each borough
[1, 2, 3, 4, 5].forEach(boroDigit => {
  const boroughData = survey.filter(d => Math.floor(d.boro_cd / 100) === boroDigit);
  const candidateVoters = boroughData.filter(d => d.voted_for === "Candidate");
  const opponentVoters = boroughData.filter(d => d.voted_for === "Opponent");
  
  // For each policy
  policyColumns.forEach(policy => {
    const candidateAvg = d3.mean(candidateVoters, d => d[policy.field]);
    const opponentAvg = d3.mean(opponentVoters, d => d[policy.field]);
    const difference = candidateAvg - opponentAvg;
    
    policyDiffByBorough.push({
      policy: policy.name,
      difference: Math.abs(difference),
      borough: getBoroughName(boroDigit * 100),
      boro_cd: boroDigit * 100
    });
  });
});
```
```js
Plot.plot({
  width: 1000,
  marginLeft: 150,
  fy: {
    padding: 0.1,  // Add padding between facets
    label: "Borough",
    domain: ["Manhattan", "Bronx", "Brooklyn", "Queens", "Staten Island"]
  },
  x: {label: "Higher alignment among candidate voters →", domain: [0, 2]},
  marks: [
    Plot.barX(policyDiffByBorough, {
      x: "difference",
      y: "policy",
      fy: "borough",
      fill: d => d.difference > 0.5 ? "#16a34a" : d.difference > 0.2 ? "#3b82f6" : "#94a3b8",
      sort: {y: "-x"}
    }),
    Plot.text(policyDiffByBorough, {
      x: "difference",
      y: "policy",
      fy: "borough",
      text: d => `+${d.difference.toFixed(2)}`,
      dx: 5,
      textAnchor: "start",
      fontSize: 10
    }),
    Plot.ruleX([0.5], {stroke: "red", strokeDasharray: "4", strokeWidth: 2}),
    // Label for red line
    Plot.text(
      ["Manhattan", "Bronx", "Brooklyn", "Queens"].map(b => ({borough: b})),
      {
        x: 0.5,
        y: "Affordable Housing",
        fy: "borough",
        text: ["Strong differentiator →"],
        dy: -15,
        dx: 10,
        fill: "red",
        fontSize: 9,
        fontWeight: "bold",
        textAnchor: "start"
      }
    ),
    // Add "No Survey Results" for Staten Island
    Plot.text([{borough: "Staten Island"}], {
      x: 1,
      y: "Childcare Support",
      fy: "borough",
      text: ["No Survey Results"],
      fill: "#999",
      fontSize: 16,
      fontWeight: "bold"
    }),
    // Add frame around each facet to create vertical lines
    Plot.frame({stroke: "#ccc", strokeWidth: 1})
  ]
})
```
This chart shows policy differentiation across boroughs—essentially, which issues most strongly separated candidate voters from opponent voters in each area.
Each horizontal bar represents the absolute difference in average alignment scores between those who voted for the candidate versus those who voted for the opponent on a 1-5 scale. Longer bars indicate stronger differentiation: when a policy has a large difference, it means candidate voters felt much more strongly aligned with that position than opponent voters did.
The red dashed line at 0.5 marks the threshold for a "strong differentiator"—policies beyond this line were particularly decisive in distinguishing the candidate's supporters from the opponent's. Green bars (>0.5) represent the strongest differentiators, blue bars (0.2-0.5) show moderate differentiation, and gray bars (<0.2) indicate issues where both groups had similar views.

```js
(() => {
  const scatterPlot = Plot.plot({
    title: "GOTV Effectiveness: Turnout Rate vs. Doors Knocked",
    subtitle: "Each circle is a district. Color = income level.",
    width: 700,
    height: 450,
    grid: true,
    x: {
      label: "Turnout Rate (%)",
    },
    y: {
      label: "Doors Knocked (GOTV Effort)",
      tickFormat: "s"
    },
    color: {
      domain: ["Low", "Middle", "High"],
      range: ["#c7e9c0", "#41ab5d", "#00441b"],
      legend: true
    },
    marks: [
      Plot.dot(district_performance, {
        x: "turnout_rate",
        y: "gotv_doors_knocked",
        r: 6,
        fill: "income_category",
        fillOpacity: 0.8,
        stroke: "#333",
        strokeWidth: 0.5,
        tip: true,
        title: d => `${d.district_label}\nTurnout: ${d.turnout_rate.toFixed(1)}%\nDoors Knocked: ${d.gotv_doors_knocked?.toLocaleString()}\nVote Share: ${d.vote_share.toFixed(1)}%`
      })
    ]
  });

  const correlation = (arr, xKey, yKey) => {
    const xMean = d3.mean(arr, d => d[xKey]);
    const yMean = d3.mean(arr, d => d[yKey]);
    const numerator = d3.sum(arr, d => (d[xKey] - xMean) * (d[yKey] - yMean));
    const denominator = Math.sqrt(
      d3.sum(arr, d => (d[xKey] - xMean) ** 2) *
      d3.sum(arr, d => (d[yKey] - yMean) ** 2)
    );
    return numerator / denominator;
  };

  const gotvCorr = correlation(district_performance, "gotv_doors_knocked", "turnout_rate");

  const insights = document.createElement("div");
  insights.style.marginTop = "20px";
  insights.style.padding = "16px";
  insights.style.backgroundColor = "#f0f7ff";
  insights.style.borderRadius = "8px";
  insights.style.fontFamily = "sans-serif";
  insights.style.fontSize = "14px";
  insights.innerHTML = `
    <h3 style="margin-top: 0;">GOTV Analysis Summary</h3>
    <p><strong>Doors Knocked ↔ Turnout Correlation:</strong> ${gotvCorr.toFixed(3)} 
      ${gotvCorr > 0.3 ? "✅ Positive relationship" : gotvCorr > 0 ? "↗️ Weak positive" : "⚠️ No clear relationship"}</p>
    <p style="margin-bottom: 0; color: #666; font-size: 12px;">
      Correlation ranges from -1 to 1. Values above 0.3 suggest a meaningful positive relationship.
    </p>
  `;

  const container = document.createElement("div");
  container.appendChild(scatterPlot);
  container.appendChild(insights);

  return container;
})()
```
```js
Plot.plot({
  title: "Candidate Vote Share by Community District with Campaign Events",
  subtitle: "Darker blue = higher vote share | Red dots = campaign events",
  projection: {
    type: "mercator",
    domain: districts_with_results
  },
  width: 800,
  height: 800,
  color: {
    scheme: "blues",
    label: "Vote Share",
    legend: true,
    tickFormat: ".0%"
  },
  marks: [
    // District polygons
    Plot.geo(districts_with_results, {
      fill: d => d.properties.vote_share,
      stroke: "white",
      strokeWidth: 0.5,
      tip: true,
      title: d => `${d.properties.district_label}\nVotes for Candidate: ${d.properties.votes_candidate?.toLocaleString()}\nVote Share: ${(d.properties.vote_share * 100).toFixed(1)}%\nIncome Level: ${d.properties.income_category}`
    }),
    
    // Campaign events - structure depends on your data
    Plot.dot(events, {
      x: "longitude",  // adjust field name as needed
      y: "latitude",   // adjust field name as needed
      r: 5,
      fill: "#ef4444",
      stroke: "white",
      strokeWidth: 1.5,
      tip: true,
      title: d => `${d.event_type}\n${d.location}\n${d.date}` // adjust fields as needed
    })
  ]
})
```