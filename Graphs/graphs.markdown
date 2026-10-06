---
layout: page
title:  "Graphs"
---

<style>
  /* Hide the page title heading; page.title is still used by the navbar. */
  .post-header { display: none; }
  #usage-chart { max-width: 800px; }
  #usage-fy label { margin-right: 1em; }
  #usage-table { border-collapse: collapse; margin-top: .5em; }
  #usage-table th, #usage-table td { padding: 2px 12px; text-align: right; }
</style>

<script src="https://cdn.jsdelivr.net/npm/plotly.js-dist-min@2.35.2/plotly.min.js"></script>

<h3>Monthly water use by fiscal year</h3>
<div>Select a fiscal year</div>
<div id="usage-fy"></div>
<div id="usage-chart"></div>
<details>
  <summary>Show data table</summary>
  <table id="usage-table"></table>
</details>

<script>
  // Fiscal year runs July -> June.
  const MONTHS = ["Jul","Aug","Sep","Oct","Nov","Dec","Jan","Feb","Mar","Apr","May","Jun"];
  const monthIndex = m => (m + 5) % 12; // calendar month (1-12) -> position in fiscal year (0-11)

  // Minimal CSV parser that handles quoted fields containing commas.
  function parseCSV(text) {
    const rows = [];
    let row = [], field = "", inQuotes = false;
    for (let i = 0; i < text.length; i++) {
      const c = text[i];
      if (inQuotes) {
        if (c === '"' && text[i + 1] === '"') { field += '"'; i++; }
        else if (c === '"') inQuotes = false;
        else field += c;
      } else if (c === '"') inQuotes = true;
      else if (c === ",") { row.push(field); field = ""; }
      else if (c === "\n" || c === "\r") {
        if (c === "\r" && text[i + 1] === "\n") i++;
        row.push(field); field = "";
        rows.push(row); row = [];
      } else field += c;
    }
    if (field !== "" || row.length) { row.push(field); rows.push(row); }
    return rows;
  }

  const data = {}; // data[fy][monthPosition] = summed thousands_gal
  const chartEl = document.getElementById("usage-chart");

  function render() {
    const fy = document.querySelector('input[name="usage-fy"]:checked').value;
    const vals = data[fy];
    const dark = window.matchMedia("(prefers-color-scheme: dark)").matches;
    const ink = dark ? "#e5e7eb" : "#1f2937";
    const grid = dark ? "#3a3f47" : "#e3e6ea";

    // null values leave a gap in the line instead of being drawn as zero.
    Plotly.react(chartEl, [{
      x: MONTHS,
      y: vals,
      type: "scatter",
      mode: "lines+markers",
      name: fy.toUpperCase(),
      line: { color: dark ? "#6aa9e9" : "#2a6fb0", width: 2 },
      marker: { size: 8 },
      connectgaps: false,
      hovertemplate: "%{x}: %{y:,} thousand gal<extra></extra>"
    }], {
      margin: { t: 20, r: 24, b: 40, l: 70 },
      paper_bgcolor: "rgba(0,0,0,0)",
      plot_bgcolor: "rgba(0,0,0,0)",
      font: { color: ink },
      hovermode: "x",
      xaxis: { categoryorder: "array", categoryarray: MONTHS, gridcolor: grid, fixedrange: true },
      yaxis: { title: "thousand gal", rangemode: "tozero", gridcolor: grid, fixedrange: true }
    }, { responsive: true, displaylogo: false });

    // Table view
    const table = document.getElementById("usage-table");
    table.replaceChildren();
    table.insertAdjacentHTML("beforeend", "<tr><th>Month</th><th>thousand gal</th></tr>");
    MONTHS.forEach((m, i) => {
      table.insertAdjacentHTML("beforeend", "<tr><td>" + m + "</td><td>" + (vals[i] == null ? "–" : vals[i].toLocaleString()) + "</td></tr>");
    });
  }

  // One radio button per fiscal year found in the data, newest first.
  function buildOptions() {
    const container = document.getElementById("usage-fy");
    Object.keys(data).sort().reverse().forEach((fy, i) => {
      const label = document.createElement("label");
      const input = document.createElement("input");
      input.type = "radio"; input.name = "usage-fy"; input.value = fy; input.checked = i === 0;
      input.addEventListener("change", render);
      label.append(input, " " + fy.toUpperCase());
      container.append(label);
    });
  }

  fetch("{{ '/water_view.csv' | relative_url }}")
    .then(r => r.text())
    .then(text => {
      const rows = parseCSV(text);
      const header = rows[0];
      const gi = header.indexOf("thousands_gal");
      const fi = header.indexOf("fiscal_year");
      const di = header.indexOf("date");
      for (const r of rows.slice(1)) {
        const fy = (r[fi] || "").trim().toLowerCase();
        const gal = parseFloat((r[gi] || "").replace(/,/g, ""));
        const month = parseInt((r[di] || "").split("-")[1], 10); // dates look like 2022-7-18
        if (!fy || isNaN(gal) || isNaN(month)) continue;
        data[fy] = data[fy] || new Array(12).fill(null);
        const pos = monthIndex(month);
        data[fy][pos] = (data[fy][pos] || 0) + gal;
      }
      buildOptions();
      render();
    })
    .catch(err => {
      console.error(err);
      chartEl.textContent = "Failed to load chart: " + err.message;
    });
</script>
