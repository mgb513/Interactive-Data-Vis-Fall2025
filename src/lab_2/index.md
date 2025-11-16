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
# Q2
As is visble in the chart below, Columbus Circle, West 4th Street, Bowling Greene, Canal Street and Union Square show the slowest average response times

Fulton & Houston Street, Times Square, and Penn Station respond the fastest to incidents.

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
    scheme: "Reds",  
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

# Q3


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
      label: "Total Attendance"
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

In the summer of 2026, Canal Street, Penn Station, and Chambers Street are projected to field the most events attendees. 

Based on Canal Street's projected additional traffic and the stations current placement in the top 5 for longest incident response times, I would prioritize it to receive additional staffing.



# Q1

<!--
const ridership = await FileAttachment(".ridership.csv").csv({ typed: true })
const events = await FileAttachment("./stock_data/stock_events.csv").csv({ typed: true })
display(stocks[0])
display(events[0]) -->


```js
Plot.plot({
  height: 400,
  width: width,
  marks: [
    Plot.dot(ridership, {
      x: "entrances",
      y: "station",
      tip: true
    })
  ]
})
```
<!-->
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
```js
display(combined[0])
```  
```js
const ridershipWithTotal = ridership.map(r => ({
  date: r.date,
  station: r.station,
  total_ridership: r.entrances + r.exits
}));
```
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
```js
const eventDates = local_events.map(e => ({
  date: e.date,
  event_name: e.event_name
}));
```
```js
Plot.plot({
  marks: [
    // Line showing daily total ridership
    Plot.line(dailyData, {x: "date", y: "total_ridership", stroke: "black"}),
    
    // Dots marking event dates
    Plot.dot(eventDates, {
      x: "date", 
      y: (d) => {
        // Find the ridership for this event date
        const dayData = dailyData.find(day => day.date === d.date);
        return dayData?.total_ridership;
      },
      fill: "red",
      r: 5  // radius of dots
    }),
    
    // Optional: add labels for events
    Plot.text(eventDates, {
      x: "date",
      y: (d) => {
        const dayData = dailyData.find(day => day.date === d.date);
        return dayData?.total_ridership;
      },
      text: "event_name",
      dy: -10  // offset above the dot
    })
  ],
  
  // Styling
  y: {label: "Total Ridership"},
  x: {label: "Date"},
  title: "NYC Subway Ridership - Summer 2025"
})
```
```js
eventDates.map(e => ({
  event_date: e.date,
  has_ridership: dailyData.find(day => day.date === e.date) ? "YES" : "NO"
}))
```
-->


On July 15th, 2025, the subway fare increased from $2.75 to $3.00.

## Q1, Part 2: Overall Ridership Trend

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
Plot.plot({
  title: "Daily Total Ridership: June 1 - August 14, 2025",
  x: { label: "Date" },
  y: { label: "Total Daily Ridership" },
  marks: [
    
    Plot.line(dailyRidership, {
      x: "date",
      y: "total_ridership"
    }),
  
    Plot.ruleX([new Date("2025-07-15")], {
      stroke: "red",
      strokeWidth: 2,
      strokeDasharray: "5,5"
    })
  ],
  width: 800,
  height: 400
})
```


Some comparisons about ridership before and after the fare increase:

- **Before fare increase (June 1 - July 14)**: Average daily ridership was approximately ${Math.round(avgPreFare).toLocaleString()}
- **After fare increase (July 15 - August 14)**: Average daily ridership dropped to approximately ${Math.round(avgPostFare).toLocaleString()}

### Key Finding

The fare increase from $2.75 to $3.00 resulted in an immediate drop in ridership of approximately ${Math.round((1 - avgPostFare/avgPreFare) * 100)}%. 

# PLAYGROUND Q1, part 1

### days with events and days without events 
I have to use local_events and ridership. The date range already matches. 
1. I have to filter ridership for stations that match stations in local_events to get something like event_stations. 
2. Once I have these, I can then look at how ridership at these stations on events days compares to ridership at these stations on non-event days. 

So, there could be a bar chart that makes two bars for every station. Which means I'd have 50 bars.
I can also imagine a cell plot that marks a cell for every station on every day and colors it according to ridership. BUt that wouldn't be as informative. 
But probably better a line plot that has date on the x axis and ridership on the y axis and little dots to make events days. 

<!--
```js
const event_stations = ridership.map(personObject => {
  const matchingCity = city_to_state.find(cityObject => {
    return personObject.city == cityObject.city
  })
  return ({
    ...personObject, // spread to keep the existing object data
    state: matchingCity.state // get the state from the matching city object 
  })
})
display(people_with_state)


const anotherSelectedStock = view(Inputs.select(allTickers))
```

```js
Plot.plot({
  height: 200, 
  width,
  marks: [
    // filter the STOCK data to only the selected tooltip, which is in the "Ticker" column
    Plot.line(stocks.filter(d => d.Ticker === anotherSelectedStock), {
      x: "Date",
      y: "Close",
      z: "Ticker",
      stroke: "Ticker",
    }),
    // filter the EVENTS data to check if the "Related Tickers" (in the format AAPL|META|GOOG) includes the selected stock
    Plot.ruleX(events.filter(d => d["Related Tickers"].includes(anotherSelectedStock)), {
      x: "Date", 
      tip: true,
      channels: {
        "Event": "Event Name", 
        "Notes": "Notes"
      }
    })
  ]
})
```
-->


