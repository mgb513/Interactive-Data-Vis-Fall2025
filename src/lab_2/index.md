---
title: "Lab 2: Subway Staffing"
toc: true
---

This page is where you can iterate. Follow the lab instructions in the [readme.md](./README.md).


<!-- Import Data -->
```js
const incidents = FileAttachment("./data/incidents.csv").csv({ typed: true })
const local_events = FileAttachment("./data/local_events.csv").csv({ typed: true })
const upcoming_events = FileAttachment("./data/upcoming_events.csv").csv({ typed: true })
const ridership = FileAttachment("./data/ridership.csv").csv({ typed: true })
```

<!-- Include current staffing counts from the prompt -->

```js
const currentStaffing = {
  "Times Sq-42 St": 19,
  "Grand Central-42 St": 18,
  "34 St-Penn Station": 15,
  "14 St-Union Sq": 4,
  "Fulton St": 17,
  "42 St-Port Authority": 14,
  "Herald Sq-34 St": 15,
  "Canal St": 4,
  "59 St-Columbus Circle": 6,
  "125 St": 7,
  "96 St": 19,
  "86 St": 19,
  "72 St": 10,
  "66 St-Lincoln Center": 15,
  "50 St": 20,
  "28 St": 13,
  "23 St": 8,
  "Christopher St": 15,
  "Houston St": 18,
  "Spring St": 12,
  "Chambers St": 18,
  "Wall St": 9,
  "Bowling Green": 6,
  "West 4 St-Wash Sq": 4,
  "Astor Pl": 7
}
```

# Subway Staffing Report
1. How did local events impact ridership in summer 2025? What effect did the July 15th fare increase have?
2. How do the stations compare when it comes to response time? Which are the best, which are the worst? 
3. Which three stations need the most staffing help for next summer based on the 2026 event calendar?
4. If you had to prioritize _one_ station to get increased staffing, which would it be and why?



```js
display(ridership[0]) 
display(local_events[0]) 
display(incidents[0])
```

# Factors Impcating Ridership

### What effect did the July 15th fare increase have on ridership overall?

<!--
const ridership = await FileAttachment(".ridership.csv").csv({ typed: true })
const events = await FileAttachment("./stock_data/stock_events.csv").csv({ typed: true })
display(stocks[0])
display(events[0]) -->


```js
const combined = local_events.map(event => {
  const matchingRidership = ridership.find(r => 
    r.station === event.nearby_station && 
    r.date === event.date
  );
  
  return {
    date: event.date,
    station: event.nearby_station,
    event_name: event.event_name,
    estimated_attendance: event.estimated_attendance,
    entrances: matchingRidership?.entrances,
    exits: matchingRidership?.exits
  };
});
```
<!--
```js
display(combined[0])
```  -->

```js
const dailyTotals = d3.rollup(
  ridershipWithTotal,
  v => d3.sum(v, d => d.total_ridership),
  d => d.date
);
```
```js
// Convert to array for plotting
const dailyData = Array.from(dailyTotals, ([date, total]) => ({
  date: date,
  total_ridership: total
}));
```
<!--
```js
Plot.plot({
  marks: [
    // Line showing daily total ridership
    Plot.line(dailyData, {
      x: "date", 
      y: "total_ridership", 
      stroke: "black"
    }),
    // Vertical rules for event dates
    Plot.ruleX(local_events, {
      x: "date",
      stroke: "red",
      strokeWidth: 2,
      strokeOpacity: 0.5,
      tip: true,
      channels: {
        "Event": "event_name",
        "Station": "nearby_station",
        "Attendance": "estimated_attendance"
      }
    })
  ],
  y: {label: "Total Ridership"},
  x: {label: "Date"},
  title: "NYC Subway Ridership - Summer 2025"
})
```
-->

```js
  const eventDatesSet = new Set(local_events.map(e => e.date));
  
  const eventDays = dailyData.filter(d => eventDatesSet.has(d.date));
  const nonEventDays = dailyData.filter(d => !eventDatesSet.has(d.date));
  
  const avgEventRidership = d3.mean(eventDays, d => d.total_ridership);
  const avgNonEventRidership = d3.mean(nonEventDays, d => d.total_ridership);


  Plot.plot({
    marks: [
      Plot.barY([
        {type: "Event Days", ridership: avgEventRidership},
        {type: "Non-Event Days", ridership: avgNonEventRidership}
      ], {
        x: "type",
        y: "ridership",
        fill: "type",
        tip: true
      })
    ],
    y: {label: "Average Total Ridership"},
    color: {scheme: "category10"},
    title: "Event vs Non-Event Day Ridership"
  })
```

On July 15th, 2025, the subway fare increased from $2.75 to $3.00.


```js
const ridershipWithTotal = ridership.map(d => ({
  ...d,
  total_ridership: d.entrances + d.exits
}));

const preFareIncrease = ridershipWithTotal.filter(d => d.date < new Date("2025-07-15"));
const postFareIncrease = ridershipWithTotal.filter(d => d.date >= new Date("2025-07-15"));

const dailyRidership = d3.rollups(
  ridershipWithTotal,
  v => d3.sum(v, d => d.total_ridership),
  d => d.date
).map(([date, total_ridership]) => ({ date, total_ridership }));

// Calculate average ridership for each period
const avgPreFare = d3.mean(preFareIncrease, d => d.total_ridership);
const avgPostFare = d3.mean(postFareIncrease, d => d.total_ridership);
```
```js
const eventDatesSet = new Set(local_events.map(e => e.date));
  
const dataWithType = dailyData.map(d => ({
    ...d,
    day_type: eventDatesSet.has(d.date) ? "Event Day" : "Non-Event Day"
  }));
```

```js
html`<div class="card">
  ${Plot.plot({
    title: "Daily Total Ridership: June 1 - August 14, 2025",
    x: { label: "Date" },
    y: { label: "Total Daily Ridership" },
    marginTop: 40,
    marks: [
      Plot.line(dailyRidership, {
        x: "date",
        y: "total_ridership",
        stroke: "steelblue"  
      }),
      Plot.ruleX([new Date("2025-07-15")], {
        stroke: "red",  
        strokeWidth: 2,
        strokeDasharray: "5,5"
      }),
      Plot.text([{date: new Date("2025-07-15"), ridership: Math.max(...dailyRidership.map(d => d.total_ridership))}], {
        x: "date",
        y: "ridership",
        text: ["July 15th Fare Increase"],
        dy: -20,
        fill: "red",  
        fontSize: 12,
        fontWeight: "bold"
      })
    ],
    width: 800,
    height: 400
  })}
</div>`
```

Some comparisons about ridership before and after the fare increase:

- **Before fare increase (June 1 - July 14)**: Average daily ridership was approximately ${Math.round(avgPreFare).toLocaleString()}
- **After fare increase (July 15 - August 14)**: Average daily ridership dropped to approximately ${Math.round(avgPostFare).toLocaleString()}

#### Key Finding

The fare increase from $2.75 to $3.00 resulted in an immediate drop in ridership of approximately ${Math.round((1 - avgPostFare/avgPreFare) * 100)}%. 

### How did local events impact ridership in summer 2025?

<!--days with events and days without events 
I have to use local_events and ridership. The date range already matches. 
1. I have to filter ridership for stations that match stations in local_events to get something like event_stations. 
2. Once I have these, I can then look at how ridership at these stations on events days compares to ridership at these stations on non-event days. 

So, there could be a bar chart that makes two bars for every station. Which means I'd have 50 bars.
I can also imagine a cell plot that marks a cell for every station on every day and colors it according to ridership. BUt that wouldn't be as informative. 
But probably better a line plot that has date on the x axis and ridership on the y axis and little dots to make events days. 
-->

```js
  const eventStations = new Set(local_events.map(e => e.nearby_station));
```
```js
  const eventDates = new Set(local_events.map(d => new Date(d.date).toDateString()));
```


```js 
html`<div class="card"> 
  <h2>Daily Ridership with Event Days</h2>
  ${Plot.plot({
    marks: [
      Plot.line(dailyRidership, {
        x: "date",
        y: "total_ridership",
        stroke: "steelblue",
        strokeWidth: 2
      }),
      Plot.dot(
        dailyRidership.filter(d => eventDates.has(new Date(d.date).toDateString())),
        {
          x: "date",
          y: "total_ridership",
          fill: "red",
          r: 5
        }
      ),
      Plot.tip(dailyRidership, Plot.pointerX({
        x: "date",
        y: "total_ridership",
        title: d => `Date: ${new Date(d.date).toLocaleDateString()}
Ridership: ${d.total_ridership.toLocaleString()}
Event Day: ${eventDates.has(new Date(d.date).toDateString()) ? "Yes" : "No"}`
      }))
    ],
    x: { label: "Date" },
    y: { label: "Total Ridership", grid: true },
    width: 800,
    height: 400
  })}
</div>`
```
The red dots on the ridership line mark dates with events. Events are disproportionallty positioned on the ridership lines' peaks, indicating that ridership on events days is often higher than on non-event days.

# Incident Response Times

### How do the stations compare when it comes to response time? Which are the best, which are the worst?


<!--
```js
Plot.plot({
  marginLeft: 120,
  x: { label: "Average Response Time (minutes)" },
  y: { label: "Station" },
  marks: [
    Plot.barX(incidents, 
      Plot.groupY(
        { x: "mean" },
        { 
          y: "station",
          x: "response_time_minutes",
          sort: { y: "x", reverse: true },
          fill: "pink",  
          tip: true
        }
      )
    )
  ]
})
```
I'm wondering what's happening here but I think it's another pseudo bar chart.
```js
Plot.plot({
  marginLeft: 100,
  x: { 
    label: "Average Response Time (minutes)",
    zero: true  
  },
  y: { label: "Station" },
  color: { 
    scheme: "RdYlGn",  // Red-Yellow-Green
    reverse: true,     // Red = high/bad, Green = low/good
    legend: true,
    label: "Avg Response Time"
  },
  marks: [
    Plot.barX(incidents, 
      Plot.groupY(
        { x: "mean" },
        { 
          y: "station",
          x: "response_time_minutes",
          sort: { y: "x", reverse: true },
          fill: "response_time_minutes",  // Color by response time
          tip: true  // Add tooltips on hover
        }
      )
    )
  ]
})
```

```js
const stationAverages = Array.from(
  d3.group(incidents, d => d.station),
  ([station, incidents]) => ({
    station: station,
    avgResponseTime: d3.mean(incidents, d => d.response_time_minutes)
  })
).sort((a, b) => b.avgResponseTime - a.avgResponseTime);
```
```js
Plot.plot({
  marginLeft: 120,
  x: { 
    label: "Average Response Time (minutes)",
    zero: true  // FIX: Start scale at 0 (no negatives, no fixed domain)
  },
  y: { label: "Station" },
  color: { 
    scheme: "YlOrRd",  
    reverse: false,     
    legend: true,
    label: "Avg Response Time"
  },
  marks: [
    Plot.barX(stationAverages, {  // Use pre-calculated averages (not raw incidents)
      x: "avgResponseTime",        
      y: "station",
      fill: "avgResponseTime",     
      sort: { y: "x", reverse: true },
      tip: true
    })
  ]
})
```
-->

```js
Plot.plot({
  marginLeft: 150,
  title: "Response Times by Incident Severity Since 2020",
  x: {label: "Response time (minutes)"},
  y: {
    label: null,
    reverse: true,
    tickFormat: (station) => {
      const count = incidents.filter(d => {
        const incidentDate = new Date(d.date);
        return d.station === station && 
               incidentDate >= new Date("2020-01-01") && 
               incidentDate <= new Date("2025-08-14");
      }).length;
      return `${station} (${count})`;
    }
  },
  color: {
    legend: true,
    scheme: "YlOrRd",
    domain: ["low", "medium", "high"]
  },
  marks: [
    Plot.ruleX([0]),
    Plot.tickX(
      incidents.filter(d => {
        const incidentDate = new Date(d.date);
        return incidentDate >= new Date("2020-01-01") && incidentDate <= new Date("2025-08-14");
      }),
      {
        x: "response_time_minutes", 
        y: "station", 
        stroke: "severity", 
        strokeOpacity: 0.5,
        strokeWidth: 4,  // Add this to make markers fatter
        tip: {
          format: {
            y: true,
            x: true,
            stroke: true
          }
        },
        channels: {
          "Date": "date"
        }
      }
    ),
    Plot.tickX(
      incidents.filter(d => {
        const incidentDate = new Date(d.date);
        return incidentDate >= new Date("2020-01-01") && incidentDate <= new Date("2025-08-14");
      }),
      Plot.groupY(
        {x: "mean"},
        {x: "response_time_minutes", y: "station", strokeWidth: 4, sort: {y: "x"}}
      )
    )
  ]
})
```


<i>The toal number of incidents since 2020 at each station is listed in parenthesis next to the station name.
Average response times for each station is marked in black.</i>

#### Key Finding
As is visble in the chart above, which considers all incidents since 2020, Columbus Circle, Bowling Green, Canal Street and Union Square show the slowest average response times. 
On average, Times Square, and Penn Station respond to incidents most quickly.


# Recommended Staffing Increases for 2026 

### Which three stations need the most staffing help for next summer based on the 2026 event calendar?

```js
(() => {
  const stationTotals = {};
  
  upcoming_events.forEach(event => {
    const station = event.nearby_station;
    if (!stationTotals[station]) {
      stationTotals[station] = {
        station: station,
        totalAttendance: 0,
        eventCount: 0
      };
    }
    stationTotals[station].totalAttendance += event.expected_attendance;
    stationTotals[station].eventCount += 1;
  });
  
  const stations = Object.values(stationTotals);
  stations.sort((a, b) => b.totalAttendance - a.totalAttendance);
  
  return Plot.plot({
    marginLeft: 150,
    height: stations.length * 25,
    x: { 
      label: "Total Expected Attendance 2026",
      zero: true
    },
    y: { label: "Station" },
    color: {
      scheme: "Blues",
      legend: true,
      label: "Expected Attendance "
    },
    marks: [
      Plot.barX(stations, {
        x: "totalAttendance",
        y: "station",
        fill: "totalAttendance",
        sort: { y: "-x" },
        tip: true
      })
    ]
  });
})()
```
<!--
```js
(() => {
  const stationTotals = {};
  
  upcoming_events.forEach(event => {
    const station = event.nearby_station;
    if (!stationTotals[station]) {
      stationTotals[station] = {
        station: station,
        totalAttendance: 0,
        eventCount: 0
      };
    }
    stationTotals[station].totalAttendance += event.expected_attendance;
    stationTotals[station].eventCount += 1;
  });
  
  const stations = Object.values(stationTotals);
  stations.sort((a, b) => b.totalAttendance - a.totalAttendance);
  
  return Plot.plot({
    marginLeft: 150,
    height: stations.length * 25,
    x: { 
      label: "Total Expected Attendance",
      zero: true
    },
    y: { label: "Station" },
    color: {
      scheme: "Blues",
      legend: true,
      label: "Expected Attendance"
    },
    marks: [
      Plot.barX(stations, {
        x: "totalAttendance",
        y: "station",
        fill: "totalAttendance",
        sort: { y: "-x" },
        tip: {
          format: {
            x: true,     // Show totalAttendance
            y: true,     // Show station name
            fill: false  // Hide the fill value (since it's the same as x)
          }
        },
        title: d => `${d.station}\nTotal: ${d.totalAttendance.toLocaleString()}\nEvents: ${d.eventCount}`
      })
    ]
  });
})()
```
-->

#### Key Finding
In the summer of 2026, Canal Street, Penn Station, and Chambers Street are projected to field the most event attendees. Events near Canal Street are expected to draw over 70,000 attendees, Penn Station will likely accommodoate almost 60,000 attendees and CHambers Street station is located near events expected to draw close to 50,000 attendees.

Based on Canal Street's projected additional traffic and the stations current placement in the top 5 for longest incident response times, I would prioritize it to receive additional staffing.

