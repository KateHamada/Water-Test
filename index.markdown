---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

<h1>Summary for a specific fiscal year</h1>
<div>Select a fiscal year</div>
<div id="fy-options"></div>
<p>Total water use: <strong id="fy-total">Loading...</strong></p>

<table id="cost-table">
  <thead><tr><th>Charges</th><th>Total</th></tr></thead>
  <tbody>
    <tr><td>Water (adjusted)</td><td id="cost-water">–</td></tr>
    <tr><td>Sewer (adjusted)</td><td id="cost-sewer">–</td></tr>
    <tr><td><strong>Water + Sewer</strong></td><td id="cost-both"><strong>–</strong></td></tr>
  </tbody>
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
      const wi = header.indexOf("water_charges_adjusted");
      const si = header.indexOf("sewer_charges_adjusted");
      for (const r of rows.slice(1)) {
        const fy = (r[fi] || "").trim().toLowerCase();
        if (!fy) continue;
        costs[fy] = costs[fy] || { water: 0, sewer: 0 };
        costs[fy].water += parseMoney(r[wi]);
        costs[fy].sewer += parseMoney(r[si]);
        const gal = parseFloat((r[gi] || "").replace(/,/g, ""));
        if (isNaN(gal)) continue;
        totals[fy] = (totals[fy] || 0) + gal;
      }
      buildOptions();
      render();
    })
    .catch(() => { totalEl.textContent = "Failed to load data"; });
</script>
