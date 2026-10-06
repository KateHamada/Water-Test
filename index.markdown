---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

<h3>Select a fiscal year</h3>
<div id="fy-options">
  <label><input type="radio" name="fy" value="fy26" checked> FY26</label>
  <label><input type="radio" name="fy" value="fy25"> FY25</label>
  <label><input type="radio" name="fy" value="fy24"> FY24</label>
  <label><input type="radio" name="fy" value="fy23"> FY23</label>
</div>
<p>Total water use: <strong id="fy-total">Loading...</strong></p>

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
  const totalEl = document.getElementById("fy-total");

  function render() {
    const fy = document.querySelector('input[name="fy"]:checked').value;
    const sum = totals[fy] || 0;
    totalEl.textContent = sum.toLocaleString() + " thousand gallons";
  }

  fetch("{{ '/water_view.csv' | relative_url }}")
    .then(r => r.text())
    .then(text => {
      const rows = parseCSV(text);
      const header = rows[0];
      const gi = header.indexOf("thousands_gal");
      const fi = header.indexOf("fiscal_year");
      for (const r of rows.slice(1)) {
        const fy = (r[fi] || "").trim().toLowerCase();
        const gal = parseFloat((r[gi] || "").replace(/,/g, ""));
        if (!fy || isNaN(gal)) continue;
        totals[fy] = (totals[fy] || 0) + gal;
      }
      render();
    })
    .catch(() => { totalEl.textContent = "Failed to load data"; });

  document.querySelectorAll('input[name="fy"]').forEach(el => el.addEventListener("change", render));
</script>
