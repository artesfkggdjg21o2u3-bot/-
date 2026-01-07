const express = require('express');
const sqlite3 = require('sqlite3').verbose();
const bcrypt = require('bcrypt');
const app = express();

app.use(express.json());
app.use(express.static('public'));

const db = new sqlite3.Database('./bank.db');
let currentOTP = null;
let pendingTransaction = null;

// تهيئة قاعدة البيانات والأمان
db.serialize(async () => {
    const saltRounds = 10;
    const initialHash = await bcrypt.hash('1234', saltRounds); // كلمة المرور الافتراضية

    db.run("CREATE TABLE IF NOT EXISTS account (id INTEGER PRIMARY KEY, name TEXT, password TEXT, balance REAL)");
    db.run("CREATE TABLE IF NOT EXISTS logs (id INTEGER PRIMARY KEY AUTOINCREMENT, type TEXT, amount REAL, timestamp DATETIME DEFAULT CURRENT_TIMESTAMP)");
    
    db.get("SELECT count(*) as count FROM account", (err, row) => {
        if (row.count === 0) {
            db.run("INSERT INTO account (id, name, password, balance) VALUES (1, 'أحمد', ?, 50000.00)", [initialHash]);
        }
    });
});

// مسار تسجيل الدخول المشفر
app.post('/login', (req, res) => {
    const { user, pass } = req.body;
    db.get("SELECT * FROM account WHERE name = ?", [user], async (err, row) => {
        if (row && await bcrypt.compare(pass, row.password)) {
            res.json({ success: true, userId: row.id, name: row.name });
        } else {
            res.status(401).json({ success: false, message: "بيانات خاطئة!" });
        }
    });
});

// مسار تحديث الرصيد مع نظام OTP للمبالغ الكبيرة
app.post('/update-balance', (req, res) => {
    const { amount } = req.body;
    if (amount < -10000) {
        currentOTP = Math.floor(1000 + Math.random() * 9000);
        pendingTransaction = amount;
        console.log(`\n🔐 كود التأمين (OTP): ${currentOTP}\n`);
        return res.json({ requireOTP: true, message: "هذا مبلغ كبير! أدخل الكود الظاهر في الـ Terminal" });
    }
    executeTx(amount, res);
});

// تأكيد الكود
app.post('/verify-otp', (req, res) => {
    if (parseInt(req.body.otp) === currentOTP) {
        executeTx(pendingTransaction, res);
        currentOTP = null;
    } else {
        res.status(401).json({ error: "الكود خاطئ!" });
    }
});

function executeTx(amount, res) {
    db.run("UPDATE account SET balance = balance + ? WHERE id = 1", [amount], () => {
        const type = amount > 0 ? 'إيداع' : 'سحب';
        db.run("INSERT INTO logs (type, amount) VALUES (?, ?)", [type, Math.abs(amount)], () => {
            db.get("SELECT balance FROM account WHERE id = 1", (err, row) => res.json({ newBalance: row.balance }));
        });
    });
}

// مسار الرسم البياني
app.get('/balance-history', (req, res) => {
    db.all("SELECT amount, type, timestamp FROM logs ORDER BY timestamp ASC", (err, rows) => {
        let bal = 50000;
        let history = rows.map(r => {
            bal += (r.type === 'سحب' ? -r.amount : r.amount);
            return { date: new Date(r.timestamp).toLocaleTimeString('ar-EG'), balance: bal };
        });
        res.json(history);
    });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`🚀 البنك يعمل: http://localhost:${PORT}`));
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>بنكي الرقمي الاحترافي</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body { background: #f0f2f5; font-family: 'Segoe UI', Tahoma, sans-serif; }
        .balance-card { background: linear-gradient(135deg, #0d47a1 0%, #1976d2 100%); color: white; border-radius: 20px; padding: 40px; }
        .glass-card { background: white; border-radius: 15px; border: none; box-shadow: 0 4px 12px rgba(0,0,0,0.05); }
    </style>
</head>
<body class="p-4">
    <div id="loginScreen" class="container mt-5" style="max-width: 400px;">
        <div class="glass-card p-4 text-center">
            <h3>🏦 تسجيل دخول</h3>
            <input type="text" id="user" class="form-control mb-2" placeholder="أحمد">
            <input type="password" id="pass" class="form-control mb-3" placeholder="1234">
            <button onclick="login()" class="btn btn-primary w-100">دخول آمن</button>
        </div>
    </div>

    <div id="bankApp" class="container" style="display:none;">
        <div class="row g-4">
            <div class="col-md-5">
                <div class="balance-card shadow-lg text-center">
                    <small>الرصيد المتوفر</small>
                    <h1 class="display-4 fw-bold" id="balDisplay">50,000 SDG</h1>
                    <p id="usdVal" class="text-warning fw-bold">-- $</p>
                    <button onclick="convert()" class="btn btn-sm btn-outline-light">تحديث سعر الدولار</button>
                </div>
                <div class="mt-4">
                    <button onclick="handleTx(1000)" class="btn btn-success w-100 mb-2 p-3">➕ إيداع 1,000</button>
                    <button onclick="handleTx(-15000)" class="btn btn-danger w-100 p-3">➖ سحب 15,000 (OTP)</button>
                </div>
            </div>
            <div class="col-md-7">
                <div class="glass-card p-4">
                    <h5>نمو الرصيد 📈</h5>
                    <canvas id="chart"></canvas>
                </div>
            </div>
        </div>
    </div>

    <script>
        let myChart;
        function login() {
            const user = document.getElementById('user').value;
            const pass = document.getElementById('pass').value;
            fetch('/login', { method: 'POST', headers: {'Content-Type': 'application/json'}, body: JSON.stringify({user, pass}) })
            .then(res => res.json()).then(data => {
                if(data.success) {
                    document.getElementById('loginScreen').style.display = 'none';
                    document.getElementById('bankApp').style.display = 'block';
                    updateUI();
                } else alert("بيانات خاطئة!");
            });
        }

        function handleTx(amt) {
            fetch('/update-balance', { method: 'POST', headers: {'Content-Type': 'application/json'}, body: JSON.stringify({amount: amt}) })
            .then(res => res.json()).then(data => {
                if(data.requireOTP) {
                    const otp = prompt(data.message);
                    fetch('/verify-otp', { method: 'POST', headers: {'Content-Type': 'application/json'}, body: JSON.stringify({otp}) })
                    .then(res => res.ok ? updateUI() : alert("كود خاطئ!"));
                } else updateUI();
            });
        }

        function convert() {
            const sdg = parseFloat(document.getElementById('balDisplay').innerText.replace(/,/g, ''));
            document.getElementById('usdVal').innerText = (sdg/600).toFixed(2) + " USD";
        }

        function updateUI() {
            fetch('/balance-history').then(res => res.json()).then(data => {
                const last = data[data.length - 1];
                if(last) document.getElementById('balDisplay').innerText = last.balance.toLocaleString() + " SDG";
                
                const ctx = document.getElementById('chart').getContext('2d');
                if(myChart) myChart.destroy();
                myChart = new Chart(ctx, {
                    type: 'line',
                    data: {
                        labels: data.map(d => d.date),
                        datasets: [{ label: 'الرصيد', data: data.map(d => d.balance), borderColor: '#0d47a1', tension: 0.3 }]
                    }
                });
            });
        }
    </script>
</body>
</html>
