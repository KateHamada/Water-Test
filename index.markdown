---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

<h1>Summary for a specific fiscal year</h1>
<div>Select a fiscal year</div>
<div id="fy-options"></div>
<style>
  .stat-box {
    display: inline-block; margin: 1em 0; padding: 12px 20px;
    background: #eef4fb; border: 1px solid #c5d9ee; border-left: 5px solid #2a6fb0;
    border-radius: 6px;
  }
  .stat-box .stat-label { font-size: 0.85em; color: #4b5563; }
  .stat-box .stat-value { font-size: 1.6em; font-weight: bold; color: #1f2937; }
  @media (prefers-color-scheme: dark) {
    .stat-box { background: #1f2a38; border-color: #34475e; border-left-color: #6aa9e9; }
    .stat-box .stat-label { color: #9aa3af; }
    .stat-box .stat-value { color: #e5e7eb; }
  }
</style>
<div class="stat-box">
  <div class="stat-label">Total Water Usage</div>
  <div class="stat-value" id="fy-total">Loading...</div>
</div>

<table id="cost-table">
  <thead><tr><th>Charges</th><th>Total</th></tr></thead>
  <tbody>
    <tr><td>Water (adjusted)</td><td id="cost-water">–</td></tr>
    <tr><td>Sewer (adjusted)</td><td id="cost-sewer">–</td></tr>
    <tr><td><strong>Water + Sewer</strong></td><td id="cost-both"><strong>–</strong></td></tr>
  </tbody>
</table>

<h3>Top 10 customers by water + sewer charges</h3>
<table id="top-table">
  <thead><tr><th>#</th><th>Account</th><th>Water (adjusted)</th><th>Sewer (adjusted)</th><th>Total</th></tr></thead>
  <tbody></tbody>
</table>

<script>
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

  const totals = {};
  const costs = {}; // costs[fy] = { water, sewer } summed from the adjusted columns
  const byAcct = {}; // byAcct[fy][description] = { water, sewer }
  const totalEl = document.getElementById("fy-total");
  const money = v => v.toLocaleString("en-US", { style: "currency", currency: "USD" });

  // "$1,234.56" -> 1234.56, "-$2.00" -> -2, blank -> 0
  const parseMoney = s => parseFloat((s || "").replace(/[$,]/g, "")) || 0;

  function render() {
    const fy = document.querySelector('input[name="fy"]:checked').value;
    const sum = totals[fy] || 0;
    totalEl.textContent = sum.toLocaleString() + " kgals";

    const c = costs[fy] || { water: 0, sewer: 0 };
    document.getElementById("cost-water").textContent = money(c.water);
    document.getElementById("cost-sewer").textContent = money(c.sewer);
    document.getElementById("cost-both").innerHTML = "<strong>" + money(c.water + c.sewer) + "</strong>";

    // Top 10 accounts (by description) for the selected year, highest combined charges first.
    const top = Object.entries(byAcct[fy] || {})
      .map(([desc, v]) => ({ desc, water: v.water, sewer: v.sewer, total: v.water + v.sewer }))
      .sort((a, b) => b.total - a.total)
      .slice(0, 10);
    const body = document.querySelector("#top-table tbody");
    body.replaceChildren();
    top.forEach((t, i) => {
      const tr = document.createElement("tr");
      [i + 1, t.desc, money(t.water), money(t.sewer), money(t.total)].forEach(val => {
        const td = document.createElement("td");
        td.textContent = val;
        tr.append(td);
      });
      body.append(tr);
    });
  }

  // Build one radio button per fiscal year found in the data, newest first.
  // So that you don't have to keep updating the hardcoded options and will
  // auto adjust based off of the data
  function buildOptions() {
    const container = document.getElementById("fy-options");
    const years = Object.keys(totals).sort().reverse();
    years.forEach((fy, i) => {
      const label = document.createElement("label");
      label.style.marginRight = "1em";
      const input = document.createElement("input");
      input.type = "radio";
      input.name = "fy";
      input.value = fy;
      input.checked = i === 0;
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
      const di = header.indexOf("description");
      const wi = header.indexOf("water_charges_adjusted");
      const si = header.indexOf("sewer_charges_adjusted");
      for (const r of rows.slice(1)) {
        const fy = (r[fi] || "").trim().toLowerCase();
        if (!fy) continue;
        costs[fy] = costs[fy] || { water: 0, sewer: 0 };
        costs[fy].water += parseMoney(r[wi]);
        costs[fy].sewer += parseMoney(r[si]);
        const desc = (r[di] || "").trim();
        if (desc) {
          byAcct[fy] = byAcct[fy] || {};
          const a = byAcct[fy][desc] = byAcct[fy][desc] || { water: 0, sewer: 0 };
          a.water += parseMoney(r[wi]);
          a.sewer += parseMoney(r[si]);
        }
        const gal = parseFloat((r[gi] || "").replace(/,/g, ""));
        if (isNaN(gal)) continue;
        totals[fy] = (totals[fy] || 0) + gal;
      }
      buildOptions();
      render();
    })
    .catch(() => { totalEl.textContent = "Failed to load data"; });
</script>
