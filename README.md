<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="UTF-8">
<title>Scan Orders</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<style>
    body { font-family: sans-serif; padding: 20px; }
    input, select, button { width: 100%; padding: 12px; margin: 10px 0; font-size: 18px; }
    #result { font-size: 20px; margin-top: 20px; }
    video { width: 100%; border-radius: 10px; margin-top: 10px; }
</style>
</head>
<body>

<h2>📦 Scan Order System</h2>

<label>Order ID:</label>
<input type="text" id="orderId" placeholder="เช่น 2512-001-00001">

<label>Box No.:</label>
<input type="text" id="boxNo" placeholder="ใส่เลขกล่อง เช่น 1">

<button onclick="startScanner()">🎥 เปิดกล้องเพื่อสแกน</button>
<video id="preview"></video>

<div id="result"></div>

<script src="https://unpkg.com/@zxing/library@latest"></script>
<script>
let codeReader;

function startScanner() {
    const videoElem = document.getElementById('preview');

    if (!codeReader) {
        codeReader = new ZXing.BrowserBarcodeReader();
    }

    codeReader.decodeFromVideoDevice(null, videoElem, (result, err) => {
        if (result) {
            sendScan(result.getText());
        }
    });
}

function sendScan(barcode) {
    const orderId = document.getElementById("orderId").value.trim();
    const boxNo = document.getElementById("boxNo").value.trim();

    if (!orderId) { alert("กรุณาใส่เลขออเดอร์"); return; }
    if (!boxNo) { alert("กรุณาใส่เลขกล่อง"); return; }

    const payload = {
        order_id: orderId,
        box_no: boxNo,
        barcode: barcode,
        qty: 1
    };

    fetch("https://script.google.com/macros/s/AKfycbwFAVGzDWuEXoSGUKoGpXzoH24dY9svYzT5XBYBlhyy53Eg_Gi29k0v_esHBcU2wQal/exec", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(payload)
    })
    .then(r => r.json())
    .then(res => {
        document.getElementById("result").innerHTML =
            "✔ สแกนแล้ว: " + barcode +
            "<br>สินค้า: " + res.sku_name;
    });
}
</script>

</body>
</html>
