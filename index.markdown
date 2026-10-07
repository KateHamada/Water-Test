---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: single
classes: wide
---

<style>
  /* minimal-mistakes: keep the content in a centered 740px column (the floating boxes are
     positioned relative to it), hide the page title, and keep labels inline. */
  .page__title { display: none; }
  .page__content { max-width: 740px; margin: 0 auto; }
  .page__content label { display: inline-block; margin: 0; }
</style>

{% include quick-links.html %}

<div><strong>STARS Filter</strong></div>
<div id="stars-filter">
  <label><input type="radio" name="stars-filter" value="stars"> Only STARS</label>
  <label><input type="radio" name="stars-filter" value="all" checked> Every Account</label>
</div>
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
  #stars-filter label { margin-right: 1em; }
</style>
<div class="stat-box">
  <div class="stat-label">Total Water Usage</div>
  <div class="stat-value" id="fy-total">Loading...</div>
</div>

<table id="cost-table">
  <thead><tr><th>Charges</th><th>Total</th></tr></thead>
  <tbody>
    <tr><td>Water</td><td id="cost-water">–</td></tr>
    <tr><td>Sewer</td><td id="cost-sewer">–</td></tr>
    <tr><td><strong>Water + Sewer</strong></td><td id="cost-both"><strong>–</strong></td></tr>
  </tbody>
</table>

<h2>Top 10 customers by water + sewer charges</h2>
<table id="top-table">
  <thead><tr><th>#</th><th>Account</th><th>Usage (kgals)</th><th>Water</th><th>Sewer</th><th>Total</th></tr></thead>
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

  const datasets = {
    stars: { totals: {}, costs: {}, byAcct: {} },
    all: { totals: {}, costs: {}, byAcct: {} }
  };
  const totalEl = document.getElementById("fy-total");
  const money = v => v.toLocaleString("en-US", { style: "currency", currency: "USD" });

  // "$1,234.56" -> 1234.56, "-$2.00" -> -2, blank -> 0
  const parseMoney = s => parseFloat((s || "").replace(/[$,]/g, "")) || 0;

  function render() {
    const fyInput = document.querySelector('input[name="fy"]:checked');
    if (!fyInput) return;
    const fy = fyInput.value;
    const dataset = datasets[document.querySelector('input[name="stars-filter"]:checked').value];
    const sum = dataset.totals[fy] || 0;
    totalEl.textContent = sum.toLocaleString();

    const c = dataset.costs[fy] || { water: 0, sewer: 0 };
    document.getElementById("cost-water").textContent = money(c.water);
    document.getElementById("cost-sewer").textContent = money(c.sewer);
    document.getElementById("cost-both").innerHTML = "<strong>" + money(c.water + c.sewer) + "</strong>";

    // Top 10 accounts (by description) for the selected year, highest combined charges first.
    const top = Object.entries(dataset.byAcct[fy] || {})
      .map(([desc, v]) => ({ desc, usage: v.usage, water: v.water, sewer: v.sewer, total: v.water + v.sewer }))
      .sort((a, b) => b.total - a.total)
      .slice(0, 10);
    const body = document.querySelector("#top-table tbody");
    body.replaceChildren();
    top.forEach((t, i) => {
      const tr = document.createElement("tr");
      [i + 1, t.desc, t.usage.toLocaleString() + " kgals", money(t.water), money(t.sewer), money(t.total)].forEach(val => {
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
  function buildOptions(selectedFY) {
    const container = document.getElementById("fy-options");
    const dataset = datasets[document.querySelector('input[name="stars-filter"]:checked').value];
    const years = Object.keys(dataset.totals).sort().reverse();
    const activeFY = years.includes(selectedFY) ? selectedFY : years[0];
    container.replaceChildren();
    years.forEach((fy, i) => {
      const label = document.createElement("label");
      label.style.marginRight = "1em";
      const input = document.createElement("input");
      input.type = "radio";
      input.name = "fy";
      input.value = fy;
      input.checked = fy === activeFY;
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
      const sti = header.indexOf("stars_include");
      for (const r of rows.slice(1)) {
        const fy = (r[fi] || "").trim().toLowerCase();
        if (!fy) continue;
        const desc = (r[di] || "").trim();
        const gal = parseFloat((r[gi] || "").replace(/,/g, ""));
        const isStars = (r[sti] || "").trim() !== "0";
        const targets = isStars ? [datasets.all, datasets.stars] : [datasets.all];
        targets.forEach(dataset => {
          dataset.costs[fy] = dataset.costs[fy] || { water: 0, sewer: 0 };
          dataset.costs[fy].water += parseMoney(r[wi]);
          dataset.costs[fy].sewer += parseMoney(r[si]);
          if (desc) {
            dataset.byAcct[fy] = dataset.byAcct[fy] || {};
            const account = dataset.byAcct[fy][desc] = dataset.byAcct[fy][desc] || { usage: 0, water: 0, sewer: 0 };
            account.water += parseMoney(r[wi]);
            account.sewer += parseMoney(r[si]);
          }
          if (!isNaN(gal)) {
            dataset.totals[fy] = (dataset.totals[fy] || 0) + gal;
            if (desc) dataset.byAcct[fy][desc].usage += gal;
          }
        });
      }
      buildOptions();
      document.querySelectorAll('input[name="stars-filter"]').forEach(input => {
        input.addEventListener("change", () => {
          const selectedInput = document.querySelector('input[name="fy"]:checked');
          const selectedFY = selectedInput ? selectedInput.value : undefined;
          buildOptions(selectedFY);
          render();
        });
      });
      render();
    })
    .catch(() => { totalEl.textContent = "Failed to load data"; });
</script>
