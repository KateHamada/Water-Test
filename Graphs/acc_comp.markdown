---
layout: page
title:  "Account Comparison"
---

<style>
  /* Hide the page title heading; page.title is still used by the navbar. */
  .post-header { display: none; }
  /* Fixed height so the chart never resizes (and pushes the table) when the selection changes. */
  #acct-chart { max-width: 800px; height: 450px; }
  #acct-msg { padding: 1em 0; color: #6b7280; }
  #acct-input { width: 100%; max-width: 400px; padding: 4px 8px; }
  #acct-chips { margin-top: .5em; }
  #acct-chips .chip {
    margin: 0 .5em .5em 0; padding: 2px 10px; cursor: pointer;
    background: transparent; color: inherit; border: 2px solid; border-radius: 999px; font: inherit;
  }
  /* Floating fiscal year picker that stays beside the chart while scrolling.
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
  #acct-table { border-collapse: collapse; margin-top: .5em; }
  #acct-table th, #acct-table td { padding: 2px 12px; text-align: right; }
</style>

<script src="https://cdn.jsdelivr.net/npm/plotly.js-dist-min@2.35.2/plotly.min.js"></script>

{% include quick-links.html %}

<h3>Account Water Usage Comparison</h3>
<div id="fy-float">
  <button id="fy-toggle" type="button" aria-expanded="false" aria-controls="usage-fy">Select FY</button>
  <div id="fy-title"><strong>Fiscal year</strong></div>
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
<div>Uses the fiscal year selected on the left. Type to search, then pick an account to add it. Add several to compare them.</div>
<input id="acct-input" list="acct-list" placeholder="Search for an account..." autocomplete="off">
<datalist id="acct-list"></datalist>
<div id="acct-chips"></div>
<div id="acct-msg"></div>
<div id="acct-chart"></div>
<details>
  <summary>Show data table</summary>
  <table id="acct-table"></table>
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

  const byAcct = {}; // byAcct[description][fy][monthPosition] = summed thousands_gal
  const fyList = new Set();
  const acctEl = document.getElementById("acct-chart");
  const acctInput = document.getElementById("acct-input");
  const acctMsg = document.getElementById("acct-msg");
  const acctChips = document.getElementById("acct-chips");

  // Accounts picked in the search box. Each keeps its color while it stays on the chart.
  const PALETTE = ["#3b82c4", "#e08a2e", "#3da36b", "#9b6bc9", "#d65a7a", "#8a8f98"];
  const chosen = [];
  const acctColor = {};

  function currentFY() {
    return document.querySelector('input[name="usage-fy"]:checked').value;
  }

  function renderChips() {
    acctChips.replaceChildren();
    chosen.forEach(desc => {
      const b = document.createElement("button");
      b.type = "button";
      b.className = "chip";
      b.style.borderColor = acctColor[desc];
      b.title = "Remove";
      b.textContent = desc + " ×";
      b.addEventListener("click", () => {
        chosen.splice(chosen.indexOf(desc), 1);
        renderChips();
        render();
      });
      acctChips.append(b);
    });
  }

  // One line per chosen account for the selected fiscal year.
  function render() {
    const fy = currentFY();
    const table = document.getElementById("acct-table");
    table.replaceChildren();
    Plotly.purge(acctEl);

    if (!chosen.length) { acctMsg.textContent = "Search for an account and pick it to add it to the chart."; return; }
    const series = chosen.filter(d => byAcct[d][fy])
      .map(d => ({ name: d, vals: byAcct[d][fy], color: acctColor[d] }));
    if (!series.length) { acctMsg.textContent = "The chosen accounts have no data for " + fy.toUpperCase() + "."; return; }
    acctMsg.textContent = chosen.length > series.length ? "Some chosen accounts have no data for " + fy.toUpperCase() + "." : "";

    // dark mode looks weird
    // const dark = window.matchMedia("(prefers-color-scheme: dark)").matches;
    const ink = "#1f2937";
    const grid = "#3a3f47";

    // null values leave a gap in the line instead of being drawn as zero.
    Plotly.react(acctEl, series.map(s => ({
      x: MONTHS,
      y: s.vals,
      type: "scatter",
      mode: "lines+markers",
      name: s.name,
      line: { color: s.color, width: 2 },
      marker: { size: 8 },
      connectgaps: false,
      hovertemplate: "%{y:,} thousand gal"
    })), {
      height: 450,
      margin: { t: 20, r: 24, b: 40, l: 70 },
      paper_bgcolor: "rgba(0,0,0,0)",
      plot_bgcolor: "rgba(0,0,0,0)",
      font: { color: ink },
      hovermode: "x unified",
      showlegend: series.length > 1,
      legend: { orientation: "h", y: -0.12 },
      xaxis: { categoryorder: "array", categoryarray: MONTHS, gridcolor: grid, fixedrange: true },
      yaxis: { title: "thousand gal", rangemode: "tozero", gridcolor: grid, fixedrange: true }
    }, { responsive: true, displaylogo: false });

    // Table view: one column per account
    const esc = s => s.replace(/&/g, "&amp;").replace(/</g, "&lt;");
    table.insertAdjacentHTML("beforeend",
      "<tr><th>Month</th>" + series.map(s => "<th>" + esc(s.name) + "</th>").join("") + "</tr>");
    MONTHS.forEach((m, i) => {
      table.insertAdjacentHTML("beforeend",
        "<tr><td>" + m + "</td>" + series.map(s => "<td>" + (s.vals[i] == null ? "–" : s.vals[i].toLocaleString()) + "</td>").join("") + "</tr>");
    });
  }

  // Add the account typed/picked in the search box (fires on picking from the list, Enter, or leaving the box).
  function addAccount() {
    const typed = acctInput.value.trim().toLowerCase();
    if (!typed) return;
    const desc = Object.keys(byAcct).find(d => d.toLowerCase() === typed);
    if (!desc) { acctMsg.textContent = "No account matches \"" + acctInput.value + "\"."; return; }
    if (!chosen.includes(desc)) {
      const color = PALETTE.find(c => !chosen.some(d => acctColor[d] === c));
      if (!color) { acctMsg.textContent = "You can compare up to " + PALETTE.length + " accounts. Remove one to add another."; return; }
      acctColor[desc] = color;
      chosen.push(desc);
    }
    acctInput.value = "";
    renderChips();
    render();
  }

  // One radio button per fiscal year found in the data, newest first.
  function buildOptions() {
    const container = document.getElementById("usage-fy");
    [...fyList].sort().reverse().forEach((fy, i) => {
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
        const desc = (r[ai] || "").trim();
        const gal = parseFloat((r[gi] || "").replace(/,/g, ""));
        const month = parseInt((r[di] || "").split("-")[1], 10); // dates look like 2022-7-18
        if (!fy || !desc || isNaN(gal) || isNaN(month)) continue;
        fyList.add(fy);
        byAcct[desc] = byAcct[desc] || {};
        byAcct[desc][fy] = byAcct[desc][fy] || new Array(12).fill(null);
        const pos = monthIndex(month);
        byAcct[desc][fy][pos] = (byAcct[desc][fy][pos] || 0) + gal;
      }
      const list = document.getElementById("acct-list");
      Object.keys(byAcct).sort((a, b) => a.localeCompare(b)).forEach(d => {
        const opt = document.createElement("option");
        opt.value = d;
        list.append(opt);
      });
      acctInput.addEventListener("change", addAccount);
      buildOptions();
      render();
    })
    .catch(err => {
      console.error(err);
      acctMsg.textContent = "Failed to load chart: " + err.message;
    });
</script>
