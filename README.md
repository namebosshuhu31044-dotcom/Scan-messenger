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
  </style>
</head>
<body>

  <h2>ค้นหาข้อมูลสินค้า</h2>
  <label for="orderCode">กรอก Order Code:</label><br/>
  <input type="text" id="orderCode" placeholder="เช่น 2509-475-00679" size="30" />
  <button id="searchBtn">ค้นหา</button>

  <div id="results"></div>

  <script>
    const apiUrl = 'https://script.google.com/macros/s/AKfycbwt3QcIwpL2qGmWQe9R72O6SLdc-f5xZc7oZFd3wcJatBdZarhv4a-Oeardi1RuMOnL/exec';

    // ฟังก์ชันเติม Order Code ที่หายไป
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

        // สร้างตาราง
        let html = `
          <table>
            <thead>
              <tr>
                <th>No</th>
                <th>Item Number</th>
                <th>Item Name</th>
                <th>Quantity</th>
                <th>นับได้จริง (Counted)</th>
                <th>สถานะ</th>
              </tr>
            </thead>
            <tbody>
        `;

        filtered.forEach((item, index) => {
          html += `
            <tr>
              <td>${item['No']}</td>
              <td>${item['Item Number']}</td>
              <td>${item['Item Name']}</td>
              <td>${item['Quantity']}</td>
              <td>
                <input type="number" min="0" class="counted" data-expected="${item['Quantity']}" />
              </td>
              <td class="status">-</td>
            </tr>
          `;
        });

        html += `
            </tbody>
          </table>
          <button id="checkBtn">ตรวจสอบจำนวน</button>
        `;

        resultsDiv.innerHTML = html;

        // เช็คค่าที่กรอก
        document.getElementById('checkBtn').addEventListener('click', () => {
          const inputs = document.querySelectorAll('.counted');
          inputs.forEach((input, i) => {
            const expected = parseInt(input.dataset.expected);
            const actual = parseInt(input.value);
            const statusCell = input.parentElement.nextElementSibling;

            if (isNaN(actual)) {
              statusCell.textContent = 'กรุณากรอกจำนวน';
              statusCell.className = 'status mismatch';
            } else if (actual === expected) {
              statusCell.textContent = '✔️ ตรงกัน';
              statusCell.className = 'status match';
            } else {
              statusCell.textContent = `❌ ไม่ตรง (ควรเป็น ${expected})`;
              statusCell.className = 'status mismatch';
            }
          });
        });

      } catch (error) {
        console.error(error);
        resultsDiv.textContent = 'เกิดข้อผิดพลาดในการโหลดข้อมูล';
      }
    });
  </script>

</body>
</html>
