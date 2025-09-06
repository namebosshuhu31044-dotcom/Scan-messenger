# Scan Messenger
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <title>ค้นหาข้อมูลสินค้า</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; }
    input, button { padding: 8px; font-size: 16px; }
    button { cursor: pointer; }
    #results { margin-top: 20px; white-space: pre-wrap; }
  </style>
</head>
<body>

  <h1>ค้นหาข้อมูลสินค้าใน Google Sheet</h1>

  <label for="orderCode">กรอก Order Code:</label><br/>
  <input type="text" id="orderCode" placeholder="เช่น 2509-475-00679" size="30" />
  <button id="searchBtn">ค้นหา</button>

  <div id="results"></div>

  <script>
    const apiUrl = 'https://script.google.com/macros/s/AKfycbx4r90IvX-QOH-egVjmJQp6ddnv6JpFHkA9PAdvZ0nH8xzDrjPDYBDDj6s1_W1RQHxz/exec';

    document.getElementById('searchBtn').addEventListener('click', async () => {
      const code = document.getElementById('orderCode').value.trim();
      const resultsDiv = document.getElementById('results');
      resultsDiv.textContent = 'กำลังค้นหา...';

      if (!code) {
        resultsDiv.textContent = 'กรุณากรอก Order Code ก่อน';
        return;
      }

      try {
        const response = await fetch(apiUrl);
        const data = await response.json();

        // กรองข้อมูลที่มี Order Code ตรงกับที่กรอก
        const filtered = data.filter(item => item['Order Code'] === code);

        if (filtered.length === 0) {
          resultsDiv.textContent = 'ไม่พบข้อมูลสำหรับ Order Code นี้';
          return;
        }

        // แสดงข้อมูลเป็นข้อความสวยๆ
        let output = `ผลการค้นหา Order Code: ${code}\n\n`;
        filtered.forEach(item => {
          output += `No: ${item['No']}\n`;
          output += `Item Number: ${item['Item Number']}\n`;
          output += `Sokochan Code: ${item['Sokochan Code']}\n`;
          output += `Item Name: ${item['Item Name']}\n`;
          output += `Quantity: ${item['Quantity']}\n`;
          output += '-----------------------\n';
        });

        resultsDiv.textContent = output;

      } catch (error) {
        console.error(error);
        resultsDiv.textContent = 'เกิดข้อผิดพลาดในการโหลดข้อมูล';
      }
    });
  </script>

</body>
</html>
