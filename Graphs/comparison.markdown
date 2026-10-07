---
layout: page
title:  "Comparison"
---

<style>
  /* Hide the page title heading; page.title is still used by the navbar. */
  .post-header { display: none; }
  /* Fixed height so the chart never resizes (and pushes the table) when the year changes. */
  #usage-chart { max-width: 800px; height: 450px; }
  #acct-chart { max-width: 800px; height: 450px; }
  #acct-msg { padding: 1em 0; color: #6b7280; }
  #acct-input { width: 100%; max-width: 400px; padding: 4px 8px; }
  /* Floating fiscal year picker that stays beside the charts while scrolling.
     The theme's content column is 740px wide and centered, so anchor the box's
     right edge 16px left of that column instead of the screen edge. */
  #fy-float {
    position: fixed; top: 90px; right: calc(50% + 370px + 16px); z-index: 10;
    background: #ffffff; color: #1f2937;
    border: 1px solid #e3e6ea; border-radius: 6px; padding: 8px 12px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);
  }
  #usage-fy label { display: block; }
  #fy-toggle { display: none; }
  #fy-toggle:focus-visible { outline: 2px solid #2a6fb0; outline-offset: 2px; }
  /* On narrow screens, collapse the picker below the navbar to keep it out of the way. */
  @media (max-width: 1100px) {
    #fy-float { top: 72px; left: 12px; right: auto; padding: 6px; }
    #usage-heading { margin-top: 96px; }
    #fy-toggle {
      display: block; padding: 8px 12px; border: 0; border-radius: 4px;
      background: transparent; color: inherit; font: inherit; cursor: pointer;
    }
    #fy-title, #usage-fy { display: none; }
    #fy-float.is-open #usage-fy { display: block; padding: 0 12px 8px; }
  }
  #fy-totals-table, #acct-fy-totals-table, #usage-table, #acct-usage-table {
    border-collapse: collapse; margin-top: .5em;
  }
  #fy-totals-table th, #fy-totals-table td, #acct-fy-totals-table th, #acct-fy-totals-table td,
  #usage-table th, #usage-table td, #acct-usage-table th, #acct-usage-table td {
    padding: 2px 12px; text-align: right;
  }
</style>

<script src="https://cdn.jsdelivr.net/npm/plotly.js-dist-min@2.35.2/plotly.min.js"></script>

{% include quick-links.html %}

<h3 id="usage-heading">Total Water Usage by Fiscal Year</h3>
<table id="fy-totals-table">
  <thead><tr><th>Fiscal Year</th><th>Total Usage (kgals)</th></tr></thead>
  <tbody></tbody>
</table>

<h3>Monthly Water Usage For All Accounts</h3>
<div id="fy-float">
  <button id="fy-toggle" type="button" aria-expanded="false" aria-controls="usage-fy">Select FY</button>
  <div id="fy-title"><strong>Fiscal years</strong></div>
  <div id="usage-fy"></div>
</div>
<script>
  (() => {
    const picker = document.getElementById("fy-float");
    const toggle = document.getElementById("fy-toggle");
    toggle.addEventListener("click", () => {
      const open = toggle.getAttribute("aria-expanded") !== "true";
      picker.classList.toggle("is-open", open);
      toggle.setAttribute("aria-expanded", String(open));
    });
  })();
</script>
<div id="chart-msg"></div>
<div id="usage-chart"></div>
<details>
  <summary>Show data table</summary>
  <table id="usage-table"></table>
</details>

<h3>Monthly Water Usage by Account</h3>
<div>Uses the fiscal years selected above. Type to search, then pick one account to view its monthly usage.</div>
<input id="acct-input" list="acct-list" placeholder="Search for an account..." autocomplete="off">
<datalist id="acct-list"></datalist>
<h4 id="acct-title" hidden></h4>
<h3>Selected Account Usage by Fiscal Year</h3>
<table id="acct-fy-totals-table">
  <thead><tr><th>Fiscal Year</th><th>Total Usage (kgals)</th></tr></thead>
  <tbody></tbody>
</table>
<div id="acct-msg"></div>
<div id="acct-chart"></div>
<details>
  <summary>Show data table</summary>
  <table id="acct-usage-table"></table>
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
  const byAcct = {}; // byAcct[description][fy][monthPosition] = summed thousands_gal
  const chartEl = document.getElementById("usage-chart");
  const acctEl = document.getElementById("acct-chart");
  const acctInput = document.getElementById("acct-input");
  const acctMsg = document.getElementById("acct-msg");

  // Fiscal years currently checked, oldest first so the order is stable.
  function selectedFYs() {
    return [...document.querySelectorAll('input[name="usage-fy"]:checked')].map(i => i.value).sort();
  }

  // Each fiscal year keeps its own color no matter which others are checked.
  const PALETTE = ["#3b82c4", "#e08a2e", "#3da36b", "#9b6bc9", "#d65a7a", "#8a8f98"];
  let allFYs = [];
  const colorFor = fy => PALETTE[allFYs.indexOf(fy) % PALETTE.length];

  // Draw one line per entry in series ([{ name, vals, color }]) into the given element.
  function drawLines(el, series) {
   // dark mode looks weird
    // const dark = window.matchMedia("(prefers-color-scheme: dark)").matches;
    const ink = "#1f2937";
    const grid = "#3a3f47";

    // null values leave a gap in the line instead of being drawn as zero.
    Plotly.react(el, series.map(s => ({
      x: MONTHS,
      y: s.vals,
      type: "scatter",
      mode: "lines+markers",
      name: s.name,
      line: { color: s.color, width: 2, dash: s.dash || "solid" },
      marker: { size: 8 },
      connectgaps: false,
      hovertemplate: "%{y:,} kgal"
    })), {
      height: 450,
      margin: { t: 20, r: 24, b: 40, l: 70 },
      paper_bgcolor: "rgba(0,0,0,0)",
      plot_bgcolor: "rgba(0,0,0,0)",
      font: { color: ink },
      hoverlabel: { bgcolor: "#ffffff", bordercolor: "#d1d5db", font: { color: "#1f2937" } },
      hovermode: "x unified",
      showlegend: series.length > 1,
      legend: { orientation: "h", y: -0.12 },
      xaxis: { categoryorder: "array", categoryarray: MONTHS, gridcolor: grid, fixedrange: true },
      yaxis: { title: "kgal", rangemode: "tozero", gridcolor: grid, fixedrange: true }
    }, { responsive: true, displaylogo: false });
  }

  // The single account selected for the monthly usage chart.
  const chosen = [];
  const acctTitle = document.getElementById("acct-title");
  const acctFYTotalsTable = document.querySelector("#acct-fy-totals-table tbody");
  const acctUsageTable = document.getElementById("acct-usage-table");

  // One line per selected fiscal year for the chosen account.
  function renderAccount(fys) {
    Plotly.purge(acctEl);
    acctUsageTable.replaceChildren();
    acctFYTotalsTable.replaceChildren();
    if (!chosen.length) {
      acctMsg.textContent = "Search for an account and pick it to view its usage.";
      return;
    }
    const desc = chosen[0];
    allFYs.slice().reverse().forEach(fy => {
      const row = document.createElement("tr");
      const yearCell = document.createElement("td");
      yearCell.textContent = fy.toUpperCase();
      row.append(yearCell);
      const values = byAcct[desc][fy];
      const usageCell = document.createElement("td");
      usageCell.textContent = values
        ? values.reduce((sum, value) => sum + (value || 0), 0).toLocaleString()
        : "–";
      row.append(usageCell);
      acctFYTotalsTable.append(row);
    });
    const series = [];
    fys.forEach(fy => {
      const vals = byAcct[desc][fy];
      if (!vals) return;
      series.push({
        name: fy.toUpperCase(),
        vals,
        color: colorFor(fy)
      });
    });
    if (!series.length) { acctMsg.textContent = "The chosen account has no data for the selected fiscal years."; return; }
    acctMsg.textContent = "";
    drawLines(acctEl, series);

    const header = document.createElement("tr");
    ["Month", ...series.map(s => s.name + " (kgal)")].forEach(value => {
      const th = document.createElement("th");
      th.textContent = value;
      header.append(th);
    });
    acctUsageTable.append(header);
    MONTHS.forEach((month, i) => {
      const row = document.createElement("tr");
      const monthCell = document.createElement("td");
      monthCell.textContent = month;
      row.append(monthCell);
      series.forEach(s => {
        const cell = document.createElement("td");
        cell.textContent = s.vals[i] == null ? "–" : s.vals[i].toLocaleString();
        row.append(cell);
      });
      acctUsageTable.append(row);
    });
  }

  // Add the account typed/picked in the search box (fires on picking from the list, Enter, or leaving the box).
  function addAccount() {
    const typed = acctInput.value.trim().toLowerCase();
    if (!typed) return;
    const desc = Object.keys(byAcct).find(d => d.toLowerCase() === typed);
    if (!desc) { acctMsg.textContent = "No account matches \"" + acctInput.value + "\"."; return; }
    chosen.splice(0, chosen.length, desc);
    acctTitle.textContent = desc;
    acctTitle.hidden = false;
    acctInput.value = "";
    renderAccount(selectedFYs());
  }

  function render() {
    const fys = selectedFYs();
    const table = document.getElementById("usage-table");
    table.replaceChildren();
    const chartMsg = document.getElementById("chart-msg");

    if (!fys.length) {
      Plotly.purge(chartEl);
      chartMsg.textContent = "Select at least one fiscal year.";
    } else {
      chartMsg.textContent = "";
      drawLines(chartEl, fys.map(fy => ({ name: fy.toUpperCase(), vals: data[fy], color: colorFor(fy) })));

      // Table view: one column per selected fiscal year
      table.insertAdjacentHTML("beforeend",
        "<tr><th>Month</th>" + fys.map(fy => "<th>" + fy.toUpperCase() + "</th>").join("") + "</tr>");
      MONTHS.forEach((m, i) => {
        table.insertAdjacentHTML("beforeend",
          "<tr><td>" + m + "</td>" + fys.map(fy => "<td>" + (data[fy][i] == null ? "–" : data[fy][i].toLocaleString()) + "</td>").join("") + "</tr>");
      });
    }
    renderAccount(fys);
  }

  // One checkbox per fiscal year found in the data, newest first. The newest two start checked.
  function buildOptions() {
    const container = document.getElementById("usage-fy");
    allFYs = Object.keys(data).sort();
    allFYs.slice().reverse().forEach((fy, i) => {
      const label = document.createElement("label");
      const input = document.createElement("input");
      input.type = "checkbox"; input.name = "usage-fy"; input.value = fy; input.checked = i < 2;
      input.addEventListener("change", render);
      label.append(input, " " + fy.toUpperCase());
      container.append(label);
    });
  }

  function renderFYTotals() {
    const body = document.querySelector("#fy-totals-table tbody");
    body.replaceChildren();
    allFYs.slice().reverse().forEach(fy => {
      const row = document.createElement("tr");
      const total = data[fy].reduce((sum, value) => sum + (value || 0), 0);
      [fy.toUpperCase(), total.toLocaleString()].forEach(value => {
        const cell = document.createElement("td");
        cell.textContent = value;
        row.append(cell);
      });
      body.append(row);
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
      acctInput.addEventListener("change", addAccount);
      buildOptions();
      renderFYTotals();
      render();
    })
    .catch(err => {
      console.error(err);
      chartEl.textContent = "Failed to load chart: " + err.message;
    });
</script>
