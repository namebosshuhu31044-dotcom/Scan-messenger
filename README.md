<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8" />
  <title>นับสินค้า</title>
  <style>
    body { font-family: sans-serif; padding: 20px; }
    table { border-collapse: collapse; width: 100%; margin-top: 20px; }
    th, td { border: 1px solid #ccc; padding: 6px; text-align: left; }
    .match { background: #c8f7c5; }     /* ครบ */
    .mismatch { background: #f7c5c5; } /* เกิน */
    #results { margin-top: 20px; }
    .order-info { font-weight: bold; margin-bottom: 10px; }
  </style>
</head>
<body>
  <h2>📦 ระบบนับสินค้า</h2>

  <button onclick="window.location.href='results.html'">📋 ดูรายงานผล</button>
  <br><br>

  <label>กรอกเลขออเดอร์: <input type="text" id="orderCode" /></label>
  <button id="searchBtn">🔍 โหลดข้อมูล</button>

  <div id="results"></div>

  <script>
    const apiUrl = "https://script.google.com/macros/s/AKfycbwH36554JHLEeFGvpRHm0FDtw1iemeX_ISL-lbJN0n7_20IoYFelOWRiB4kxzvn3iTT/exec";

    // 📌 โหลดออเดอร์
    async function loadOrder(code) {
      const resultsDiv = document.getElementById("results");
      localStorage.setItem("currentOrderCode", code);

      resultsDiv.innerHTML = "⏳ กำลังโหลด...";

      try {
        const response = await fetch(apiUrl);
        const data = await response.json();

        let currentCode = null;
        const rows = data
          .map(row => {
            if (row["Order Code"]) currentCode = row["Order Code"];
            else row["Order Code"] = currentCode;
            return row;
          })
          .filter(row => row["Order Code"] === code);

        if (rows.length === 0) {
          resultsDiv.innerHTML = "❌ ไม่พบข้อมูล";
          return;
        }

        // โหลด progress ล่าสุด
        const progress = JSON.parse(localStorage.getItem("scanProgress") || "{}");
        const orderProgress = progress[code] || [];

        let html = `<div class="order-info">Order: ${code}</div>
          <input type="text" id="scanInput" placeholder="สแกนสินค้า" autofocus />
          <table>
            <thead>
              <tr>
                <th>Item Number</th>
                <th>ชื่อ</th>
                <th>ควรได้</th>
                <th>นับได้</th>
                <th>ลบ</th>
                <th>สถานะ</th>
              </tr>
            </thead>
            <tbody>
        `;

        rows.forEach(item => {
          // ค่าจาก Google Sheet
          let expected = parseInt(item["Quantity"]);

          // ถ้ามี progress เก่า → ใช้ expected ที่ถูกอัปเดตแล้ว
          const old = orderProgress.find(r => r.item === item["Item Number"]);
          if (old) {
            expected = old.expected;
          }

          // ถ้า expected = 0 แล้ว ไม่ต้องโชว์
          if (expected <= 0) return;

          const countedValue = old ? old.counted : 0;

          let status = "-";
          let cls = "";
          if (countedValue === expected) {
            status = "✔️ ครบ";
            cls = "match";
          } else if (countedValue > expected) {
            status = "❌ เกิน";
            cls = "mismatch";
          } else if (countedValue > 0) {
            status = `${countedValue}/${expected}`;
          }

          html += `
            <tr data-code="${item["Item Number"]}" class="${cls}">
              <td>${item["Item Number"]}</td>
              <td>${item["Item Name"]}</td>
              <td>${expected}</td>
              <td>
                <input type="number" readonly value="${countedValue}"
                  class="counted" data-expected="${expected}" />
              </td>
              <td><button onclick="adjustCount('${item["Item Number"]}', -1)">➖</button></td>
              <td class="status">${status}</td>
            </tr>
          `;
        });

        html += `</tbody></table><br/><button onclick="saveResults()">💾 บันทึกผล</button>`;
        resultsDiv.innerHTML = html;

        document.getElementById("scanInput").addEventListener("keydown", e => {
          if (e.key === "Enter") {
            const scanCode = e.target.value.trim();
            e.target.value = "";
            handleScan(scanCode);
          }
        });

      } catch (err) {
        resultsDiv.innerHTML = "❌ โหลดข้อมูลผิดพลาด";
        console.error(err);
      }
    }

    // 📌 จัดการสแกน
    function handleScan(scanCode) {
      let code = scanCode;
      let decrease = false;

      if (code.startsWith("-")) {
        decrease = true;
        code = code.substring(1);
      }

      const row = document.querySelector(`tr[data-code="${code}"]`);
      if (!row) {
        alert("❌ ไม่พบสินค้า");
        return;
      }

      const input = row.querySelector(".counted");
      const expected = parseInt(input.dataset.expected);
      let current = parseInt(input.value) || 0;

      if (decrease && current > 0) current--;
      else if (!decrease) current++;

      input.value = current;

      const status = row.querySelector(".status");
      row.classList.remove("match", "mismatch");

      if (current === expected) {
        status.textContent = "✔️ ครบ";
        row.classList.add("match");
      } else if (current > expected) {
        status.textContent = "❌ เกิน";
        row.classList.add("mismatch");
      } else {
        status.textContent = `${current}/${expected}`;
      }

      updateStorage();
    }

    // 📌 ปรับจำนวน
    function adjustCount(code, change) {
      const row = document.querySelector(`tr[data-code="${code}"]`);
      if (!row) return;

      const input = row.querySelector(".counted");
      const expected = parseInt(input.dataset.expected);
      let current = parseInt(input.value) || 0;

      current += change;
      if (current < 0) current = 0;
      input.value = current;

      const status = row.querySelector(".status");
      row.classList.remove("match", "mismatch");

      if (current === expected) {
        status.textContent = "✔️ ครบ";
        row.classList.add("match");
      } else if (current > expected) {
        status.textContent = "❌ เกิน";
        row.classList.add("mismatch");
      } else {
        status.textContent = `${current}/${expected}`;
      }

      updateStorage();
    }

    // 📌 อัปเดต storage
    function updateStorage() {
      const currentOrderCode = localStorage.getItem("currentOrderCode");
      if (!currentOrderCode) return;

      const rows = document.querySelectorAll("tr[data-code]");
      const results = [];

      rows.forEach(row => {
        const input = row.querySelector(".counted");
        const expected = parseInt(input.dataset.expected);
        const counted = parseInt(input.value);

        results.push({
          item: row.dataset.code,
          name: row.children[1].textContent,
          expected: expected,
          counted: counted
        });
      });

      const progress = JSON.parse(localStorage.getItem("scanProgress") || "{}");
      progress[currentOrderCode] = results;
      localStorage.setItem("scanProgress", JSON.stringify(progress));
    }

    // 📌 บันทึก (ย้ายไป Carton-Box)
    function saveResults() {
      const currentOrderCode = localStorage.getItem("currentOrderCode");
      if (!currentOrderCode) return;

      const progress = JSON.parse(localStorage.getItem("scanProgress") || "{}");
      const orderProgress = progress[currentOrderCode] || [];

      // เอาเฉพาะสินค้าที่นับได้ > 0
      const completed = orderProgress.filter(it => it.counted > 0);
      if (completed.length === 0) {
        alert("❌ ยังไม่มีสินค้าที่นับ");
        return;
      }

      // โหลด countedResults
      const allResults = JSON.parse(localStorage.getItem("countedResults") || "{}");
      if (!allResults[currentOrderCode]) {
        allResults[currentOrderCode] = { boxes: [] };
      }

      const boxCount = allResults[currentOrderCode].boxes.length + 1;
      allResults[currentOrderCode].boxes.push({
        cartonBox: "Carton-Box " + boxCount,
        items: completed
      });

      localStorage.setItem("countedResults", JSON.stringify(allResults));

      const updatedProgress = orderProgress.map(it => {
  const remaining = it.expected - it.counted;
  return {
    ...it,
    expected: remaining,
    counted: 0
  };
});

      progress[currentOrderCode] = updatedProgress;
      localStorage.setItem("scanProgress", JSON.stringify(progress));

      alert("✅ บันทึกผลเรียบร้อยแล้ว (Carton-Box " + boxCount + ")");
      loadOrder(currentOrderCode);
    }

    // ปุ่มค้นหา
    document.getElementById("searchBtn").addEventListener("click", () => {
      const code = document.getElementById("orderCode").value.trim();
      if (!code) {
        alert("กรุณากรอกเลขออเดอร์");
        return;
      }
      loadOrder(code);
    });

    // โหลดต่ออัตโนมัติ
    window.addEventListener("DOMContentLoaded", () => {
      const currentOrderCode = localStorage.getItem("currentOrderCode");
      if (currentOrderCode) {
        loadOrder(currentOrderCode);
      }
    });
  </script>
</body>
</html>
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8" />
  <title>📦 รายงานผลการนับสินค้า</title>
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; }
    h2 { margin-bottom: 15px; }
    .btn { margin: 5px; padding: 6px 12px; border: 1px solid #888; border-radius: 4px; cursor: pointer; }
    .btn-back { background: #e3f2fd; }
    .btn-export { background: #ede7f6; }
    .btn-reset { background: #e8f5e9; }
    .no-data { color: red; margin-top: 20px; }
    table { width: 100%; border-collapse: collapse; margin: 15px 0; }
    th, td { border: 1px solid #ccc; padding: 6px; text-align: left; }
    tr.complete { background-color: #c8e6c9; }
    tr.over { background-color: #ffcdd2; }
    .order-header { font-weight: bold; margin-top: 30px; font-size: 18px; text-align: left; }
    .carton-header { font-weight: bold; margin-top: 15px; font-size: 16px; text-align: right; }
  </style>
</head>
<body>
  <h2>📋 รายงานผลการนับสินค้า</h2>
  <button class="btn btn-back" onclick="window.location.href='index.html'">⬅ กลับไปหน้าสแกน</button>
  <button class="btn btn-export" onclick="exportCSV()">📤 Export CSV</button>
  <button class="btn btn-reset" onclick="resetCarton()">♻ รีเซ็ตกล่อง</button>

  <div id="reportContainer"></div>
  <div id="noData" class="no-data" style="display:none;">❌ ไม่มีข้อมูลที่บันทึกไว้</div>

  <script>
    const container = document.getElementById("reportContainer");
    const noData = document.getElementById("noData");

    const allResults = JSON.parse(localStorage.getItem("countedResults")) || {};

    if (Object.keys(allResults).length === 0) {
      noData.style.display = "block";
    } else {
      noData.style.display = "none";
      for (let order in allResults) {
        const orderData = allResults[order];

        const orderDiv = document.createElement("div");
        orderDiv.classList.add("order-header");
        orderDiv.textContent = "Order: " + order;
        container.appendChild(orderDiv);

        orderData.boxes.forEach(box => {
          const cartonDiv = document.createElement("div");
          cartonDiv.classList.add("carton-header");
          cartonDiv.textContent = box.cartonBox;
          container.appendChild(cartonDiv);

          const table = document.createElement("table");
          table.innerHTML = `
            <thead>
              <tr>
                <th>Item Number</th>
                <th>ชื่อ</th>
                <th>นับได้</th>
              </tr>
            </thead>
            <tbody></tbody>
          `;

          const tbody = table.querySelector("tbody");
          box.items.forEach(item => {
            const tr = document.createElement("tr");
            tr.innerHTML = `
              <td>${item.item}</td>
              <td>${item.name}</td>
              <td>${item.counted}</td>
            `;
            tbody.appendChild(tr);
          });

          container.appendChild(table);
        });
      }
    }

    function exportCSV() {
      let csvContent = "data:text/csv;charset=utf-8,";
      csvContent += "Order,Carton-Box,Item Number,Name,Counted\n";

      for (let order in allResults) {
        const orderData = allResults[order];
        orderData.boxes.forEach(box => {
          box.items.forEach(item => {
            csvContent += `${order},${box.cartonBox},${item.item},${item.name},${item.counted}\n`;
          });
        });
      }

      const encodedUri = encodeURI(csvContent);
      const link = document.createElement("a");
      link.setAttribute("href", encodedUri);
      link.setAttribute("download", "report.csv");
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    }

    function resetCarton() {
      if (confirm("คุณต้องการรีเซ็ตข้อมูลทั้งหมดหรือไม่?")) {
        localStorage.removeItem("countedResults");
        localStorage.removeItem("scanProgress");
        localStorage.removeItem("currentOrderCode");
        location.reload();
      }
    }
  </script>
</body>
</html>
