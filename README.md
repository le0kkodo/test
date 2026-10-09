<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Barcode Scanner App</title>
    <!-- HTML5 QR Code Scanner Library via CDN -->
    <script src="https://cloudflare.com"></script>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background-color: #f2f2f7; margin: 0; padding: 16px; color: #1c1c1e; }
        .app-container { max-width: 450px; margin: 0 auto; background: #ffffff; border-radius: 12px; padding: 20px; box-shadow: 0 4px 12px rgba(0,0,0,0.08); }
        h2 { text-align: center; color: #007aff; margin-top: 0; }
        .step { display: none; text-align: center; }
        .step.active { display: block; }
        .scanner-placeholder { width: 100%; min-height: 250px; background: #000; border-radius: 8px; margin: 15px 0; overflow: hidden; position: relative; }
        input[type="text"], textarea { width: 100%; padding: 12px; box-sizing: border-box; border: 1px solid #c7c7cc; border-radius: 8px; font-size: 16px; margin: 10px 0; background: #fafafa; }
        .btn { display: block; width: 100%; background: #007aff; color: white; border: none; padding: 14px; border-radius: 8px; font-size: 16px; font-weight: 600; margin-top: 12px; cursor: pointer; -webkit-appearance: none; }
        .btn-secondary { background: #e5e5ea; color: #1c1c1e; }
        .btn-success { background: #34c759; }
        .summary { text-align: left; background: #f2f2f7; padding: 14px; border-radius: 8px; margin: 15px 0; }
        .summary p { margin: 6px 0; font-size: 15px; }
        .label { font-weight: bold; color: #8e8e93; display: inline-block; width: 110px; }
    </style>
</head>
<body>

<div class="app-container">
    <h2>Barcode Logger</h2>

    <!-- STEP 1: SERIAL NUMBER -->
    <div id="step1" class="step active">
        <h3>Step 1: Scan Serial Number</h3>
        <div id="reader-serial" class="scanner-placeholder"></div>
        <input type="text" id="manual-serial" placeholder="Or type Serial manually">
        <button class="btn" onclick="nextStep(2)">Next Step</button>
    </div>

    <!-- STEP 2: PART NUMBER -->
    <div id="step2" class="step">
        <h3>Step 2: Scan Part Number</h3>
        <div id="reader-part" class="scanner-placeholder"></div>
        <input type="text" id="manual-part" placeholder="Or type Part Number manually">
        <button class="btn" onclick="nextStep(3)">Next Step</button>
        <button class="btn btn-secondary" onclick="backToStep(1)">Back</button>
    </div>

    <!-- STEP 3: REMARKS -->
    <div id="step3" class="step">
        <h3>Step 3: Enter Remarks</h3>
        <textarea id="remarks" rows="4" placeholder="Type context, location, or notes here..."></textarea>
        <button class="btn" onclick="nextStep(4)">Review Log</button>
        <button class="btn btn-secondary" onclick="backToStep(2)">Back</button>
    </div>

    <!-- STEP 4: REVIEW & SAVE -->
    <div id="step4" class="step">
        <h3>Step 4: Save Entry?</h3>
        <div class="summary">
            <p><span class="label">Serial No:</span> <span id="view-serial"></span></p>
            <p><span class="label">Part No:</span> <span id="view-part"></span></p>
            <p><span class="label">Remarks:</span> <span id="view-remarks"></span></p>
        </div>
        <button class="btn btn-success" onclick="saveData()">Confirm & Export Excel/CSV</button>
        <button class="btn btn-secondary" onclick="backToStep(3)">Back</button>
    </div>
</div>

<script>
    let logData = { serial: '', part: '', remarks: '' };
    let sessionLogs = [];
    let html5QrcodeScanner = null;

    function startScanner(elementId, successCallback) {
        if (html5QrcodeScanner) {
            html5QrcodeScanner.clear().catch(err => console.log(err));
        }

        html5QrcodeScanner = new Html5Qrcode(elementId);
        const config = { fps: 10, qrbox: { width: 250, height: 150 } };

        html5QrcodeScanner.start(
            { facingMode: "environment" }, 
            config, 
            (decodedText) => {
                html5QrcodeScanner.stop().then(() => {
                    successCallback(decodedText);
                });
            },
            (errorMessage) => { /* Ignore noisy frame match errors */ }
        ).catch(err => {
            console.log("Camera failed to start, typing manually allowed.", err);
        });
    }

    function stopScanner() {
        if (html5QrcodeScanner && html5QrcodeScanner.isScanning) {
            return html5QrcodeScanner.stop();
        }
        return Promise.resolve();
    }

    function initStep1() {
        startScanner("reader-serial", (code) => {
            document.getElementById('manual-serial').value = code;
            nextStep(2);
        });
    }

    function nextStep(stepNum) {
        if (stepNum === 2) {
            const val = document.getElementById('manual-serial').value.trim();
            if (!val) { alert("Please scan or type a Serial Number first."); return; }
            logData.serial = val;
            
            document.querySelectorAll('.step').forEach(s => s.classList.remove('active'));
            document.getElementById('step2').classList.add('active');
            
            stopScanner().then(() => {
                startScanner("reader-part", (code) => {
                    document.getElementById('manual-part').value = code;
                    nextStep(3);
                });
            });
        } 
        else if (stepNum === 3) {
            const val = document.getElementById('manual-part').value.trim();
            if (!val) { alert("Please scan or type a Part Number first."); return; }
            logData.part = val;
            
            stopScanner().then(() => {
                document.querySelectorAll('.step').forEach(s => s.classList.remove('active'));
                document.getElementById('step3').classList.add('active');
            });
        } 
        else if (stepNum === 4) {
            logData.remarks = document.getElementById('remarks').value.trim() || 'N/A';
            
            document.getElementById('view-serial').innerText = logData.serial;
            document.getElementById('view-part').innerText = logData.part;
            document.getElementById('view-remarks').innerText = logData.remarks;
            
            document.querySelectorAll('.step').forEach(s => s.classList.remove('active'));
            document.getElementById('step4').classList.add('active');
        }
    }

    function backToStep(stepNum) {
        document.querySelectorAll('.step').forEach(s => s.classList.remove('active'));
        document.getElementById('step' + stepNum).classList.add('active');
        
        stopScanner().then(() => {
            if (stepNum === 1) initStep1();
            if (stepNum === 2) {
                startScanner("reader-part", (code) => {
                    document.getElementById('manual-part').value = code;
                    nextStep(3);
                });
            }
        });
    }

    function saveData() {
        const timestamp = new Date().toLocaleString();
        sessionLogs.push({
            timestamp: timestamp,
            serial: logData.serial,
            part: logData.part,
            remarks: logData.remarks
        });

        alert("Data logged! Downloading your updated spreadsheet file...");
        exportToCSV();

        // Reset App Form
        document.getElementById('manual-serial').value = '';
        document.getElementById('manual-part').value = '';
        document.getElementById('remarks').value = '';
        backToStep(1);
    }

    function exportToCSV() {
        let csvContent = "data:text/csv;charset=utf-8,Timestamp,Serial Number,Part Number,Remarks\n";
        sessionLogs.forEach(row => {
            csvContent += `"${row.timestamp}","${row.serial}","${row.part}","${row.remarks}"\n`;
        });
        
        const encodedUri = encodeURI(csvContent);
        const link = document.createElement("a");
        link.setAttribute("href", encodedUri);
        link.setAttribute("download", "scanned_inventory_log.csv");
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    }

    // Start scanner on initial load
    window.onload = () => { initStep1(); };
</script>

</body>
</html>
