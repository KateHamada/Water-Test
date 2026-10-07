---
layout: single
classes: wide
title:  "Account Charge Comparison"
---

<style>
  /* minimal-mistakes: keep the content in a centered 740px column (the floating boxes are
     positioned relative to it), hide the page title, and keep labels inline. */
  .page__title { display: none; }
  .page__content { max-width: 740px; margin: 0 auto; }
  .page__content label { display: inline-block; margin: 0; }
  /* Fixed height so the chart never resizes (and pushes the table) when the selection changes. */
  #acct-chart { max-width: 100%; height: 450px; }
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
  #charge-type-control { margin-bottom: 1em; }
  #charge-type label { margin-right: 1em; }
  /* On narrow screens, collapse the picker below the navbar to keep it out of the way. */
  @media (max-width: 1100px) {
    #fy-float { top: 72px; left: 12px; right: auto; padding: 6px; }
    #usage-heading { margin-top: 120px; }
    /* "right" is updated by script so this box slides left when the quick links menu opens. */
    #charge-type-control {
      position: fixed; top: 72px; right: 70px; z-index: 10; margin: 0;
      padding: 6px 10px; background: #ffffff; color: #1f2937;
      border: 1px solid #e3e6ea; border-radius: 6px;
      box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);
    }
    #charge-type label { display: block; margin-right: 0; }
    #fy-toggle {
      display: block; padding: 8px 12px; border: 0; border-radius: 4px;
      background: transparent; color: inherit; font: inherit; cursor: pointer;
    }
    #fy-title, #usage-fy { display: none; }
    #fy-float.is-open #usage-fy { display: block; padding: 0 12px 8px; }
  }
  #acct-table { border-collapse: collapse; margin-top: .5em; }
  #acct-table th, #acct-table td { padding: 2px 12px; text-align: right; }
  /* Make "Show data table" look like a button, with an arrow that flips when open. */
  details > summary {
    display: inline-block; list-style: none; cursor: pointer; user-select: none;
    padding: 6px 14px; border: 1px solid currentColor; border-radius: 6px;
    background: rgba(127, 127, 127, 0.12); font-weight: 600;
  }
  details > summary::-webkit-details-marker { display: none; }
  details > summary::before { content: "\25B8"; display: inline-block; margin-right: 8px; transition: transform .15s; }
  details[open] > summary::before { transform: rotate(90deg); }
  details > summary:hover { background: rgba(127, 127, 127, 0.25); }
  details > summary:focus-visible { outline: 2px solid #2a6fb0; outline-offset: 2px; }
</style>

<script src="https://cdn.jsdelivr.net/npm/plotly.js-dist-min@2.35.2/plotly.min.js"></script>

{% include quick-links.html %}

<div id="charge-type-control">
  <div><strong>Charge Type</strong></div>
  <div id="charge-type">
    <label><input type="radio" name="charge-type" value="water" checked> Water</label>
    <label><input type="radio" name="charge-type" value="sewer"> Sewer</label>
    <label><input type="radio" name="charge-type" value="both"> Both</label>
  </div>
</div>
<script>
  // On narrow screens the quick links menu grows leftward when opened, so keep the
  // charge type box just to its left instead of underneath it (where it hid the close button).
  (() => {
    const nav = document.getElementById("quick-links");
    const box = document.getElementById("charge-type-control");
    const narrow = window.matchMedia("(max-width: 1100px)");
    function place() {
      box.style.right = narrow.matches ? (nav.offsetWidth + 12 + 8) + "px" : "";
    }
    new ResizeObserver(place).observe(nav);
    narrow.addEventListener("change", place);
    place();
  })();
</script>

<h3 id="usage-heading">Account <span class="ct">Water Charges</span> Comparison</h3>
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

  const records = []; // every usable CSV row: { fy, month, desc, water, sewer }
  let byAcct = {}; // byAcct[description][fy][monthPosition] = summed charges ($) for the chosen charge type
  const fyList = new Set();
  const acctEl = document.getElementById("acct-chart");
  const acctInput = document.getElementById("acct-input");
  const acctMsg = document.getElementById("acct-msg");
  const acctChips = document.getElementById("acct-chips");

  // Accounts picked in the search box. Each keeps its color while it stays on the chart.
  const PALETTE = ["#3b82c4", "#e08a2e", "#3da36b", "#9b6bc9", "#d65a7a", "#8a8f98"];
  const chosen = [];
  const acctColor = {};

  const money = v => v.toLocaleString("en-US", { style: "currency", currency: "USD" });
  // "$1,234.56" -> 1234.56, "-$2.00" -> -2, blank -> NaN
  const parseMoney = s => parseFloat((s || "").replace(/[$,]/g, ""));

  // Which charges the page is showing: water, sewer, or both added together.
  const CHARGE_LABELS = { water: "Water Charges", sewer: "Sewer Charges", both: "Water + Sewer Charges" };
  const chargeType = () => document.querySelector('input[name="charge-type"]:checked').value;
  const chargeLabel = () => CHARGE_LABELS[chargeType()];

  // Amount for one record under the current charge type; null when it has no value (blank in the CSV).
  function amountFor(rec) {
    const type = chargeType();
    const value = type === "water" ? rec.water : type === "sewer" ? rec.sewer
      : (Number.isNaN(rec.water) && Number.isNaN(rec.sewer)) ? NaN : (rec.water || 0) + (rec.sewer || 0);
    return Number.isNaN(value) ? null : value;
  }

  // Refresh every heading that names the charge type.
  function updateLabels() {
    document.querySelectorAll(".ct").forEach(el => { el.textContent = chargeLabel(); });
  }

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
      hovertemplate: "%{y:$,.2f}"
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
      yaxis: { title: chargeLabel(), tickprefix: "$", rangemode: "tozero", gridcolor: grid, fixedrange: true }
    }, { responsive: true, displaylogo: false });

    // Table view: one column per account
    const esc = s => s.replace(/&/g, "&amp;").replace(/</g, "&lt;");
    table.insertAdjacentHTML("beforeend",
      "<tr><th>Month</th>" + series.map(s => "<th>" + esc(s.name) + "</th>").join("") + "</tr>");
    MONTHS.forEach((m, i) => {
      table.insertAdjacentHTML("beforeend",
        "<tr><td>" + m + "</td>" + series.map(s => "<td>" + (s.vals[i] == null ? "–" : money(s.vals[i])) + "</td>").join("") + "</tr>");
    });
    table.insertAdjacentHTML("beforeend",
      "<tr><th>Total</th>" + series.map(s =>
        "<td><strong>" + money(s.vals.reduce((sum, value) => sum + (value || 0), 0)) + "</strong></td>"
      ).join("") + "</tr>");
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

  // Rebuild the per-account totals from the records for the current charge type.
  function rebuildData() {
    byAcct = {};
    fyList.clear();
    records.forEach(rec => {
      const { fy, month, desc } = rec;
      // Keep every fiscal year listed even if it has no value for this charge type.
      fyList.add(fy);
      const amount = amountFor(rec);
      if (amount === null) return;
      byAcct[desc] = byAcct[desc] || {};
      byAcct[desc][fy] = byAcct[desc][fy] || new Array(12).fill(null);
      const pos = monthIndex(month);
      byAcct[desc][fy][pos] = (byAcct[desc][fy][pos] || 0) + amount;
    });
  }

  function renderAccountOptions() {
    const list = document.getElementById("acct-list");
    list.replaceChildren();
    Object.keys(byAcct).sort((a, b) => a.localeCompare(b)).forEach(d => {
      const opt = document.createElement("option");
      opt.value = d;
      list.append(opt);
    });
  }

  // One radio button per fiscal year found in the data, newest first.
  // keep is the fiscal year to leave selected (e.g. across a charge type change), if it still exists.
  function buildOptions(keep) {
    const container = document.getElementById("usage-fy");
    container.replaceChildren();
    const years = [...fyList].sort().reverse();
    const selected = years.includes(keep) ? keep : years[0];
    years.forEach(fy => {
      const label = document.createElement("label");
      const input = document.createElement("input");
      input.type = "radio"; input.name = "usage-fy"; input.value = fy; input.checked = fy === selected;
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
      const wi = header.indexOf("water_charges_adjusted");
      const sci = header.indexOf("sewer_charges_adjusted");
      const fi = header.indexOf("fiscal_year");
      const di = header.indexOf("date");
      const ai = header.indexOf("description");
      for (const r of rows.slice(1)) {
        const fy = (r[fi] || "").trim().toLowerCase();
        const desc = (r[ai] || "").trim();
        const water = parseMoney(r[wi]);
        const sewer = parseMoney(r[sci]);
        const month = parseInt((r[di] || "").split("-")[1], 10); // dates look like 2022-7-18
        if (!fy || !desc || isNaN(month)) continue;
        records.push({ fy, month, desc, water, sewer });
      }
      document.getElementById("charge-type").addEventListener("change", () => {
        const keep = currentFY();
        rebuildData();
        renderAccountOptions();
        buildOptions(keep);
        // Drop chosen accounts that have no values for this charge type.
        for (let i = chosen.length - 1; i >= 0; i--) {
          if (!byAcct[chosen[i]]) chosen.splice(i, 1);
        }
        updateLabels();
        renderChips();
        render();
      });
      acctInput.addEventListener("change", addAccount);
      updateLabels();
      rebuildData();
      renderAccountOptions();
      buildOptions();
      render();
    })
    .catch(err => {
      console.error(err);
      acctMsg.textContent = "Failed to load chart: " + err.message;
    });
</script>
