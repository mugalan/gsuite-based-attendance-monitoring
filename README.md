# 🛡️ Secure QR Attendance System

A professional-grade, automated attendance tracking system using **Google Colab**, **Google Apps Script**, and **Google Sheets**.

### 🚀 Key Features

* **Anti-Cheat Security:** Dynamic QR codes rotate every 30 seconds using MD5 hashing to prevent screenshots or link sharing.
* **Automatic Daily Logs:** Automatically creates a new tab in Google Sheets for each day (e.g., `28-03-2026`).
* **Localized Time:** Specifically tuned for **Asia/Colombo (GMT+5:30)**.
* **Identity Verification:** Forces users to sign in with their Google/School account to verify their email address.
* **Real-time Feedback:** Displays the student's name and their "Check-in Number" (e.g., *"You are person #5 today!"*) upon successful scan.

---

## 🛠️ Step 1: The Google Apps Script (The Backend)

1. Create a new **Google Sheet**.
2. Navigate to **Extensions > Apps Script**.
3. Delete all existing code and paste the following:

```javascript
// Change this to your own secret password
var SECRET_PASSWORD = "SCHOOL_SECURITY_2026";

function doGet(e) {
  var userKey = e.parameter.key;
  var isValid = checkKey(userKey);
  var html = '<html><body style="font-family:sans-serif; text-align:center; padding:50px; background:#f4f7f6;">';
  
  if (!isValid) {
    html += '<h1 style="color:#d93025;">❌ Invalid or Expired QR</h1>';
    html += '<p>Please scan the live QR code displayed on the screen.</p>';
  } else {
    html += '<div style="background:white; padding:30px; border-radius:15px; display:inline-block; box-shadow:0 4px 6px rgba(0,0,0,0.1);">' +
            '<h2>Verify Attendance</h2><p>Click below to confirm your identity.</p>' +
            '<button style="padding:15px 30px; font-size:18px; background:#1a73e8; color:white; border:none; border-radius:8px; cursor:pointer;" onclick="record()">Confirm Check-in</button></div>' +
            '<script>function record(){google.script.run.withSuccessHandler(done).saveEmail("' + userKey + '");}' +
            'function done(i){document.body.innerHTML="<h1>✅ Success!</h1><p>Thanks, <b>"+i.name+"</b>!</p><p>You are person <b>#"+i.count+"</b> today.</p>";}</script>';
  }
  return HtmlService.createHtmlOutput(html + '</body></html>');
}

function checkKey(key) {
  var now = new Date();
  var timeSlot = Math.floor(now.getTime() / 30000); // 30-second window
  var expectedKey = Utilities.computeDigest(Utilities.DigestAlgorithm.MD5, SECRET_PASSWORD + timeSlot);
  var expectedKeyStr = expectedKey.map(function(byt){return ('0' + (byt & 0xFF).toString(16)).slice(-2)}).join('');
  return key === expectedKeyStr;
}

function saveEmail(key) {
  if (!checkKey(key)) throw new Error("Expired");
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheetName = Utilities.formatDate(new Date(), "GMT+5:30", "dd-MM-yyyy");
  var sheet = ss.getSheetByName(sheetName) || ss.insertSheet(sheetName, 0);
  if (sheet.getLastRow() === 0) sheet.appendRow(["Time", "Email", "Name"]);
  
  var email = Session.getActiveUser().getEmail();
  var name = email.split('@')[0];
  var time = Utilities.formatDate(new Date(), "GMT+5:30", "HH:mm:ss");
  
  sheet.appendRow([time, email, name]);
  return { name: name, count: (sheet.getLastRow() - 1) };
}

```

4. **Deploy:**
* Click **Deploy > New Deployment**.
* Select **Web app**.
* Execute as: **Me**.
* Who has access: **Anyone** (Users will still be prompted to log in to capture their email).
* **Copy the Web App URL** for Step 2.



---

## 🐍 Step 2: Google Colab QR Generation (The Frontend)

1. Open a new notebook in [Google Colab](https://colab.research.google.com).
2. Paste and run the following code:

```python
!pip install qrcode[pil] --quiet

import qrcode
import hashlib
import time
from IPython.display import display, Image, clear_output

# --- CONFIGURATION ---
WEB_APP_URL = "YOUR_APPS_SCRIPT_URL_HERE"
SECRET_PASSWORD = "SCHOOL_SECURITY_2026" # Must match Apps Script

def get_security_key():
    time_slot = int(time.time() // 30) # 30-second sync
    raw_str = f"{SECRET_PASSWORD}{time_slot}"
    return hashlib.md5(raw_str.encode()).hexdigest()

try:
    while True:
        clear_output(wait=True)
        secure_key = get_security_key()
        secure_url = f"{WEB_APP_URL}?key={secure_key}"
        
        qr_img = qrcode.make(secure_url)
        qr_img.save("qr.png")
        
        print(f"🛡️ SECURE ATTENDANCE ACTIVE | {time.strftime('%H:%M:%S')}")
        print("QR rotates every 30 seconds to prevent cheating.")
        display(Image("qr.png"))
        time.sleep(10)
except KeyboardInterrupt:
    print("System Stopped.")

```

---

## 📖 How it Works

1. **Synchronization:** Both Python (Colab) and Apps Script calculate an MD5 hash based on a `SECRET_PASSWORD` and the current `30-second window` of time.
2. **Verification:** When a student scans the QR, the script checks if the `key` in the URL matches the server's current key.
3. **Data Logging:** If valid, the user's authenticated email is retrieved, formatted, and appended to a date-specific tab in the Google Sheet.

---

**Would you like me to help you write a "Troubleshooting" section for the GitHub repo as well?**
