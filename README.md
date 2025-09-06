# Scan Messenger
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <title>เช็คสินค้า</title>
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; }
    input, button { padding: 8px; font-size: 16px; }
    table { border-collapse: collapse; width: 100%; margin-top: 20px; }
    th, td { border: 1px solid #ccc; padding: 8px; text-align: left; }
    th { background-color: #f5f5f5; }
    .match { color: green; }
    .mismatch { color: red; }
    .status { font-weight: bold; }
    .scanning { margin-top: 30px; }
    .btn-group { display: flex; gap: 5px; }
    .btn-count {
      padding: 5px 10px;
      font-size: 16px;
      cursor: pointer;
    }
  </style>
</head>
<body>

  <h2>ค้นหาข้อมูลสินค้า</h2>
  <label for="orderCode">กรอก Order Code:</label><br/>
  <input type="text" id="orderCode" placeholder="เช่น 2509-475-00679" size="30" />
  <button id="searchBtn">ค้นหา</button>

  <div id="results"></div>

  <script>
    const apiUrl = 'https://script.google.com/macros/s/AKfycbwH36554JHLEeFGvpRHm0FDtw1iemeX_ISL-lbJN0n7_20IoYFelOWRiB4kxzvn3iTT/exec';

    // เติม Order Code ที่หายไป
    function fillOrderCode(data) {
      let lastOrderCode = null;
      return data.map(row => {
        if (row["Order Code"]) {
          lastOrderCode = row["Order Code"];
        } else {
          row["Order Code"] = lastOrderCode;
        }
        return row;
      });
    }

    document.getElementById('searchBtn').addEventListener('click', async () => {
      const code = document.getElementById('orderCode').value.trim();
      const resultsDiv = document.getElementById('results');
      resultsDiv.innerHTML = 'กำลังโหลดข้อมูล...';

      if (!code) {
        resultsDiv.textContent = 'กรุณากรอก Order Code ก่อน';
        return;
      }

      try {
        const response = await fetch(apiUrl);
        const rawData = await response.json();
        const data = fillOrderCode(rawData);
        const filtered = data.filter(item => item['Order Code'] === code);

        if (filtered.length === 0) {
          resultsDiv.textContent = 'ไม่พบข้อมูลสำหรับ Order Code นี้';
          return;
        }

        let html = `
          <div class="scanning">
            <label for="scanInput"><strong>สแกนสินค้า:</strong></label><br/>
            <input type="text" id="scanInput" placeholder="สแกน Item Number หรือ -Item เพื่อลด" autofocus />
          </div>

          <table>
            <thead>
              <tr>
                <th>No</th>
                <th>Item Number</th>
                <th>Item Name</th>
                <th>Quantity</th>
                <th>นับได้จริง</th>
                <th>ลดยอด</th>
                <th>สถานะ</th>
              </tr>
            </thead>
            <tbody>
        `;

        filtered.forEach((item, index) => {
          html += `
            <tr data-code="${item['Item Number']}" data-index="${index}">
              <td>${item['No']}</td>
              <td>${item['Item Number']}</td>
              <td>${item['Item Name']}</td>
              <td>${item['Quantity']}</td>
              <td>
                <input type="number" min="0" class="counted" data-expected="${item['Quantity']}" value="0" readonly />
              </td>
              <td>
                <div class="btn-group">
                  <button class="btn-count" onclick="adjustCount('${item['Item Number']}', -1)">➖</button>
                </div>
              </td>
              <td class="status">-</td>
            </tr>
          `;
        });

        html += `
            </tbody>
          </table>
        `;

        resultsDiv.innerHTML = html;

        // ช่องสแกนสินค้า
        document.getElementById('scanInput').addEventListener('keydown', e => {
          if (e.key === 'Enter') {
            let scanned = e.target.value.trim();
            e.target.value = '';

            let decrease = false;
            if (scanned.startsWith('-')) {
              decrease = true;
              scanned = scanned.substring(1);
            }

            const row = document.querySelector(`tr[data-code="${scanned}"]`);
            if (row) {
              const input = row.querySelector('.counted');
              const expected = parseInt(input.dataset.expected);
              let current = parseInt(input.value) || 0;

              if (decrease) {
                if (current > 0) current -= 1;
              } else {
                current += 1;
              }

              input.value = current;

              const status = row.querySelector('.status');
              if (current === expected) {
                status.textContent = '✔️ ครบแล้ว';
                status.className = 'status match';
              } else if (current > expected) {
                status.textContent = `❌ เกิน (ควรเป็น ${expected})`;
                status.className = 'status mismatch';
              } else {
                status.textContent = `➕ นับได้ ${current} / ${expected}`;
                status.className = 'status';
              }
            } else {
              alert('ไม่พบสินค้านี้ในรายการ');
            }
          }
        });

      } catch (error) {
        console.error(error);
        resultsDiv.textContent = 'เกิดข้อผิดพลาดในการโหลดข้อมูล';
      }
    });

    // ฟังก์ชันลดจำนวน
    function adjustCount(itemCode, change) {
      const row = document.querySelector(`tr[data-code="${itemCode}"]`);
      if (!row) return;

      const input = row.querySelector('.counted');
      const expected = parseInt(input.dataset.expected);
      let current = parseInt(input.value) || 0;

      current += change;
      if (current < 0) current = 0;

      input.value = current;

      const status = row.querySelector('.status');
      if (current === expected) {
        status.textContent = '✔️ ครบแล้ว';
        status.className = 'status match';
      } else if (current > expected) {
        status.textContent = `❌ เกิน (ควรเป็น ${expected})`;
        status.className = 'status mismatch';
      } else {
        status.textContent = `➕ นับได้ ${current} / ${expected}`;
        status.className = 'status';
      }
    }
  </script>

</body>
</html>
