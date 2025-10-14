---
title: "Lab 0: Getting Started"
toc: true
---

This page is where you can iterate. Follow the lab instructions in the [readme.md](./README.md).

# outer orbit
## inner orbit
<h1>Hello Maria's Labs</h1>
<h2>and so it begins<h2>
<p>This is where you can figure things out.</p>
  <table>
    <thead>
      <tr>
        <th>Difficulty</th>
        <th>Hours</th>
        <th>Lab</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>high</td>
        <td>12</td>
        <td>Lab 0</td>
      </tr>
      <tr>
        <td>higher</td>
        <td>tbd</td>
        <td>Lab 1</td>
      </tr>
      <tr>
        <td>tbd</td>
        <td>tbd</td>
        <td>Lab 2</td>
      </tr>
    </tbody>
  </table>

<img src="https://cdn.stocksnap.io/img-thumbs/960w/forest-nature_OHVUEJMMYT.jpg" alt="Example image">

<h3>image associations</h3>
<ul>
  <li>trees</li>
  <li>light</li>
  <li>seasons</li>
  <li>perspective</li>
</ul>

<a href="https://www.gc.cuny.edu/">CUNY Gradcenter</a>

```js run=false
1 + 2
```

The current time is ${new Date(now).toLocaleTimeString("en-US")}.

<div class="card">
  ${resize((width) => Plot.barX([9, 4, 8, 1, 11, 3, 4, 2, 7, 5]).plot({width}))}
</div>


```js
const team = view(Inputs.radio(["NY Liberty", "Las Vegas Aces", "Phoenix Mercury", "Indiana Fever"], {label: "Favorite team:", value: "NY Liberty"}));
```
**My favorite WNBA team is the ${team}!**