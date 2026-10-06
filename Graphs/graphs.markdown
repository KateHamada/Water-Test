---
layout: page
title:  "Graphs"
---

<style>
  /* Hide the page title heading; page.title is still used by the navbar. */
  .post-header { display: none; }
  /* Fixed height so the chart never resizes (and pushes the table) when the year changes. */
  #usage-chart { max-width: 800px; height: 450px; }
  #acct-chart { max-width: 800px; height: 450px; }
  #acct-msg { padding: 1em 0; color: #6b7280; }
  #acct-input { width: 100%; max-width: 400px; padding: 4px 8px; }
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

<h3>Monthly water use by account</h3>
<div>Uses the fiscal year selected above. Click the box and type to search accounts.</div>
<input id="acct-input" list="acct-list" placeholder="Search for an account..." autocomplete="off">
<datalist id="acct-list"></datalist>
<div id="acct-msg"></div>
<div id="acct-chart"></div>

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
  const byAcct = {}; // byAcct[description][fy][monthPosition] = summed thousands_gal
  const chartEl = document.getElementById("usage-chart");
  const acctEl = document.getElementById("acct-chart");
  const acctInput = document.getElementById("acct-input");
  const acctMsg = document.getElementById("acct-msg");

  function currentFY() {
    return document.querySelector('input[name="usage-fy"]:checked').value;
  }

  // Draw a single-series line chart of monthly usage into the given element.
  function drawLine(el, vals, name) {
    const dark = window.matchMedia("(prefers-color-scheme: dark)").matches;
    const ink = dark ? "#e5e7eb" : "#1f2937";
    const grid = dark ? "#3a3f47" : "#e3e6ea";

    // null values leave a gap in the line instead of being drawn as zero.
    Plotly.react(el, [{
      x: MONTHS,
      y: vals,
      type: "scatter",
      mode: "lines+markers",
      name: name,
      line: { color: dark ? "#6aa9e9" : "#2a6fb0", width: 2 },
      marker: { size: 8 },
      connectgaps: false,
      hovertemplate: "%{x}: %{y:,} thousand gal<extra></extra>"
    }], {
      height: 450,
      margin: { t: 20, r: 24, b: 40, l: 70 },
      paper_bgcolor: "rgba(0,0,0,0)",
      plot_bgcolor: "rgba(0,0,0,0)",
      font: { color: ink },
      hovermode: "x",
      xaxis: { categoryorder: "array", categoryarray: MONTHS, gridcolor: grid, fixedrange: true },
      yaxis: { title: "thousand gal", rangemode: "tozero", gridcolor: grid, fixedrange: true }
    }, { responsive: true, displaylogo: false });
  }

  // Chart for the account typed/selected in the search box, for the selected fiscal year.
  function renderAccount() {
    const fy = currentFY();
    const typed = acctInput.value.trim().toLowerCase();
    const desc = Object.keys(byAcct).find(d => d.toLowerCase() === typed);
    Plotly.purge(acctEl);
    if (!typed) { acctMsg.textContent = "Select an account to see its usage."; return; }
    if (!desc) { acctMsg.textContent = "No account matches \"" + acctInput.value + "\"."; return; }
    const vals = byAcct[desc][fy];
    if (!vals) { acctMsg.textContent = desc + " has no data for " + fy.toUpperCase() + "."; return; }
    acctMsg.textContent = "";
    drawLine(acctEl, vals, desc);
  }

  function render() {
    const fy = currentFY();
    const vals = data[fy];
    drawLine(chartEl, vals, fy.toUpperCase());
    renderAccount();

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
      const ai = header.indexOf("description");
      for (const r of rows.slice(1)) {
        const fy = (r[fi] || "").trim().toLowerCase();
        const gal = parseFloat((r[gi] || "").replace(/,/g, ""));
        const month = parseInt((r[di] || "").split("-")[1], 10); // dates look like 2022-7-18
        if (!fy || isNaN(gal) || isNaN(month)) continue;
        data[fy] = data[fy] || new Array(12).fill(null);
        const pos = monthIndex(month);
        data[fy][pos] = (data[fy][pos] || 0) + gal;

        const desc = (r[ai] || "").trim();
        if (desc) {
          byAcct[desc] = byAcct[desc] || {};
          byAcct[desc][fy] = byAcct[desc][fy] || new Array(12).fill(null);
          byAcct[desc][fy][pos] = (byAcct[desc][fy][pos] || 0) + gal;
        }
      }
      const list = document.getElementById("acct-list");
      Object.keys(byAcct).sort((a, b) => a.localeCompare(b)).forEach(d => {
        const opt = document.createElement("option");
        opt.value = d;
        list.append(opt);
      });
      acctInput.addEventListener("input", renderAccount);
      buildOptions();
      render();
    })
    .catch(err => {
      console.error(err);
      chartEl.textContent = "Failed to load chart: " + err.message;
    });
</script>
