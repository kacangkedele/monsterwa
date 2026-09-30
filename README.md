# Bot Telegram "MonsterWa" + Dashboard

⚠️ **Catatan Penting:** Saya tidak bisa mengakses `http://NAROWA.CO` atau membongkar source code `@narowa_bot` karena keterbatasan akses internet. Bot di bawah ini adalah **template bot Telegram lengkap dengan dashboard web** yang saya buat berdasarkan fitur umum bot sejenis (layanan SaaS/WhatsApp Gateway dengan sistem saldo, katalog, top-up, referral, dll). Anda bisa menyesuaikan logika bisnis sesuai kebutuhan.

---

## 📁 Struktur Project

```
monsterwa/
├── config.py
├── database.py
├── bot.py
├── app.py              # Flask dashboard
├── requirements.txt
├── templates/
│   ├── login.html
│   ├── dashboard.html
│   └── users.html
└── monsterwa.db        # auto-created
```

---

## 1. `requirements.txt`

```txt
python-telegram-bot==21.6
Flask==3.0.3
Flask-Login==0.6.3
Werkzeug==3.0.3
requests==2.32.3
```

Install:
```bash
pip install -r requirements.txt
```

---

## 2. `config.py`

```python
import os

# ====== KONFIGURASI MONSTERWA ======
BOT_TOKEN = os.getenv("BOT_TOKEN", "ISI_TOKEN_BOT_KAMU_DISINI")
ADMIN_IDS = [123456789]  # Ganti dengan Telegram ID admin (ambil dari @userinfobot)

# Flask Dashboard
SECRET_KEY = "monsterwa-secret-change-me"
DASHBOARD_USER = "admin"
DASHBOARD_PASS = "monsterwa123"  # ganti di production

# Branding
BOT_NAME = "MonsterWa"
BOT_USERNAME = "MonsterWaBot"
SUPPORT_CONTACT = "@adminsobat"
WEBSITE = "https://monsterwa.example"

# Saldo awal user baru
WELCOME_BONUS = 0
REFERRAL_BONUS = 1000
```

---

## 3. `database.py`

```python
import sqlite3
from datetime import datetime
import config

DB_PATH = "monsterwa.db"

def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("""CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        first_name TEXT,
        balance INTEGER DEFAULT 0,
        referral_code TEXT,
        referred_by INTEGER,
        api_key TEXT,
        created_at TEXT,
        is_banned INTEGER DEFAULT 0
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS transactions (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        type TEXT,
        amount INTEGER,
        description TEXT,
        status TEXT DEFAULT 'pending',
        created_at TEXT
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS services (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        code TEXT UNIQUE,
        name TEXT,
        price INTEGER,
        description TEXT,
        category TEXT,
        active INTEGER DEFAULT 1
    )""")
    conn.commit()
    conn.close()

def get_user(user_id):
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    c = conn.cursor()
    c.execute("SELECT * FROM users WHERE user_id=?", (user_id,))
    row = c.fetchone()
    conn.close()
    return row

def add_user(user_id, username, first_name, referral_code, referred_by=None):
    import secrets
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("""INSERT OR IGNORE INTO users
        (user_id, username, first_name, balance, referral_code, referred_by, api_key, created_at)
        VALUES (?,?,?,?,?,?,?,?)""",
        (user_id, username, first_name, config.WELCOME_BONUS,
         referral_code, referred_by, secrets.token_hex(16),
         datetime.now().isoformat()))
    conn.commit()
    conn.close()

def update_balance(user_id, amount):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("UPDATE users SET balance = balance + ? WHERE user_id=?", (amount, user_id))
    conn.commit()
    conn.close()

def add_transaction(user_id, type_, amount, desc, status="success"):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("""INSERT INTO transactions (user_id, type, amount, description, status, created_at)
        VALUES (?,?,?,?,?,?)""",
        (user_id, type_, amount, desc, status, datetime.now().isoformat()))
    conn.commit()
    conn.close()

def all_users():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    c = conn.cursor()
    c.execute("SELECT * FROM users ORDER BY created_at DESC")
    rows = c.fetchall()
    conn.close()
    return rows

def all_transactions():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    c = conn.cursor()
    c.execute("SELECT * FROM transactions ORDER BY created_at DESC LIMIT 100")
    rows = c.fetchall()
    conn.close()
    return rows

def count_stats():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT COUNT(*) FROM users")
    users = c.fetchone()[0]
    c.execute("SELECT COALESCE(SUM(amount),0) FROM transactions WHERE type='topup' AND status='success'")
    topup = c.fetchone()[0]
    c.execute("SELECT COALESCE(SUM(amount),0) FROM transactions WHERE type='order' AND status='success'")
    order = c.fetchone()[0]
    conn.close()
    return {"users": users, "topup": topup, "order": order}
```

---

## 4. `bot.py` — Bot Telegram + Inline Button

```python
import logging
from telegram import (
    Update, InlineKeyboardButton, InlineKeyboardMarkup
)
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler,
    MessageHandler, ContextTypes, filters
)
import secrets, string, random

import config
import database as db

logging.basicConfig(format='%(asctime)s - %(message)s', level=logging.INFO)

def gen_ref(user_id):
    return f"MW{user_id}"[:8].upper()

# ============ KEYBOARD ============
def main_menu():
    kb = [
        [InlineKeyboardButton("👤 Profil", callback_data="profile"),
         InlineKeyboardButton("💳 Saldo", callback_data="balance")],
        [InlineKeyboardButton("🛒 Katalog Layanan", callback_data="catalog"),
         InlineKeyboardButton("📦 Order", callback_data="order")],
        [InlineKeyboardButton("💸 Top Up", callback_data="topup"),
         InlineKeyboardButton("🔑 API Key", callback_data="apikey")],
        [InlineKeyboardButton("🤝 Referral", callback_data="referral"),
         InlineKeyboardButton("📞 Bantuan", callback_data="help")],
    ]
    if False:
        kb.append([InlineKeyboardButton("⚙️ Admin Panel", callback_data="admin")])
    return InlineKeyboardMarkup(kb)

def back_button():
    return InlineKeyboardMarkup([[InlineKeyboardButton("⬅️ Kembali", callback_data="menu")]])

# ============ HANDLERS ============
async def start(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    referred_by = None
    if ctx.args:
        try:
            referred_by = int(ctx.args[0])
        except ValueError:
            referred_by = None

    db.add_user(user.id, user.username, user.first_name, gen_ref(user.id), referred_by)
    if referred_by and referred_by != user.id:
        u = db.get_user(referred_by)
        if u:
            db.update_balance(referred_by, config.REFERRAL_BONUS)
            db.add_transaction(referred_by, "bonus", config.REFERRAL_BONUS,
                               f"Referral bonus dari {user.id}")
            try:
                await ctx.bot.send_message(referred_by,
                    f"🎉 User baru mendaftar dari link Anda! +{config.REFERRAL_BONUS}")
            except Exception:
                pass

    text = (
        f"👋 *Selamat Datang di {config.BOT_NAME}!*\n\n"
        f"Layanan otomatis 24/7 dengan harga terbaik.\n\n"
        f"✅ Transaksi cepat\n"
        f"✅ Support 24 jam\n"
        f"✅ Garansi resmi\n\n"
        f"Pilih menu di bawah untuk mulai 👇"
    )
    await update.message.reply_text(text, parse_mode="Markdown", reply_markup=main_menu())

async def menu_cmd(update, ctx):
    await update.message.reply_text("📋 *Menu Utama*", parse_mode="Markdown", reply_markup=main_menu())

async def button_handler(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    data = q.data
    user = db.get_user(q.from_user.id)
    if not user:
        db.add_user(q.from_user.id, q.from_user.username, q.from_user.first_name, gen_ref(q.from_user.id))
        user = db.get_user(q.from_user.id)

    if user["is_banned"]:
        await q.edit_message_text("🚫 Akun Anda diblokir.")
        return

    if data == "menu":
        await q.edit_message_text(f"📋 *Menu Utama {config.BOT_NAME}*",
                                  parse_mode="Markdown", reply_markup=main_menu())

    elif data == "profile":
        await q.edit_message_text(
            f"👤 *Profil*\n\n"
            f"ID: `{user['user_id']}`\n"
            f"Nama: {user['first_name']}\n"
            f"Username: @{user['username'] or '-'}\n"
            f"Saldo: Rp{user['balance']:,}\n"
            f"Referral: `{user['referral_code']}`\n"
            f"API Key: `{user['api_key']}`\n"
            f"Bergabung: {user['created_at'][:10]}",
            parse_mode="Markdown", reply_markup=back_button())

    elif data == "balance":
        await q.edit_message_text(
            f"💳 *Saldo Anda*\n\nSaldo: Rp{user['balance']:,}\n\n"
            "Lakukan top up untuk menambah saldo.",
            parse_mode="Markdown", reply_markup=back_button())

    elif data == "catalog":
        await q.edit_message_text(
            "🛒 *Katalog Layanan*\n\n"
            "1. WhatsApp Blast - Rp500/1000 pesan\n"
            "2. OTP WhatsApp - Rp200/OTP\n"
            "3. Auto Reply Bot - Rp100.000/bulan\n"
            "4. API Access - Rp150.000/bulan\n\n"
            "Pilih menu Order untuk pemesanan.",
            parse_mode="Markdown", reply_markup=back_button())

    elif data == "order":
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("1. WA Blast", callback_data="order_blast")],
            [InlineKeyboardButton("2. OTP WA", callback_data="order_otp")],
            [InlineKeyboardButton("3. Auto Reply", callback_data="order_autoreply")],
            [InlineKeyboardButton("⬅️ Kembali", callback_data="menu")],
        ])
        await q.edit_message_text("📦 *Pilih Layanan*", parse_mode="Markdown", reply_markup=kb)

    elif data.startswith("order_"):
        prices = {"blast": 500, "otp": 200, "autoreply": 100000}
        key = data.split("_")[1]
        price = prices.get(key, 0)
        if user["balance"] < price:
            await q.edit_message_text(f"❌ Saldo tidak cukup. Butuh Rp{price:,}",
                                      reply_markup=back_button())
            return
        db.update_balance(user["user_id"], -price)
        db.add_transaction(user["user_id"], "order", price, f"Order {key}")
        await q.edit_message_text(
            f"✅ *Order Berhasil!*\n\nLayanan: {key}\nHarga: Rp{price:,}\nSisa saldo: Rp{user['balance']-price:,}\n\n"
            "Pesanan sedang diproses oleh sistem.",
            parse_mode="Markdown", reply_markup=back_button())

    elif data == "topup":
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("Rp10.000", callback_data="tu_10000"),
             InlineKeyboardButton("Rp50.000", callback_data="tu_50000")],
            [InlineKeyboardButton("Rp100.000", callback_data="tu_100000"),
             InlineKeyboardButton("Rp500.000", callback_data="tu_500000")],
            [InlineKeyboardButton("⬅️ Kembali", callback_data="menu")],
        ])
        await q.edit_message_text("💸 *Pilih Nominal Top Up*", parse_mode="Markdown", reply_markup=kb)

    elif data.startswith("tu_"):
        amount = int(data.split("_")[1])
        db.update_balance(user["user_id"], amount)
        db.add_transaction(user["user_id"], "topup", amount, "Top up saldo")
        await q.edit_message_text(
            f"✅ *Top Up Berhasil!*\n\n+Rp{amount:,}\nSaldo: Rp{user['balance']+amount:,}",
            parse_mode="Markdown", reply_markup=back_button())

    elif data == "apikey":
        await q.edit_message_text(
            f"🔑 *API Key Anda*\n\n`{user['api_key']}`\n\n"
            f"Gunakan untuk akses API {config.BOT_NAME}.",
            parse_mode="Markdown", reply_markup=back_button())

    elif data == "referral":
        link = f"https://t.me/{config.BOT_USERNAME}?start={user['user_id']}"
        await q.edit_message_text(
            f"🤝 *Program Referral*\n\n"
            f"Bonus: Rp{config.REFERRAL_BONUS}/user baru\n\n"
            f"Link Anda:\n`{link}`\n\n"
            f"Bagikan link ke teman Anda!",
            parse_mode="Markdown", reply_markup=back_button())

    elif data == "help":
        await q.edit_message_text(
            f"📞 *Bantuan*\n\nSupport: {config.SUPPORT_CONTACT}\nWebsite: {config.WEBSITE}\n\n"
            "Jam operasional: 24/7",
            parse_mode="Markdown", reply_markup=back_button())

    elif data == "admin":
        if user["user_id"] in config.ADMIN_IDS:
            stats = db.count_stats()
            await q.edit_message_text(
                f"⚙️ *Admin Panel*\n\nTotal User: {stats['users']}\n"
                f"Total Top Up: Rp{stats['topup']:,}\nTotal Order: Rp{stats['order']:,}",
                parse_mode="Markdown", reply_markup=back_button())
        else:
            await q.edit_message_text("🚫 Akses ditolak.", reply_markup=back_button())

async def broadcast(update, ctx):
    if update.effective_user.id not in config.ADMIN_IDS:
        return
    if not ctx.args:
        await update.message.reply_text("Usage: /broadcast <pesan>")
        return
    msg = " ".join(ctx.args)
    users = db.all_users()
    sent = 0
    for u in users:
        try:
            await ctx.bot.send_message(u["user_id"], msg)
            sent += 1
        except Exception:
            pass
    await update.message.reply_text(f"✅ Terkirim ke {sent} user")

def main():
    db.init_db()
    app = Application.builder().token(config.BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("menu", menu_cmd))
    app.add_handler(CommandHandler("broadcast", broadcast))
    app.add_handler(CallbackQueryHandler(button_handler))
    print(f"🚀 {config.BOT_NAME} bot berjalan...")
    app.run_polling()

if __name__ == "__main__":
    main()
```

---

## 5. `app.py` — Dashboard Flask

```python
from flask import Flask, render_template, request, redirect, url_for, session, flash
import config
import database as db

app = Flask(__name__)
app.secret_key = config.SECRET_KEY

def login_required(f):
    from functools import wraps
    @wraps(f)
    def wrap(*args, **kwargs):
        if not session.get("logged_in"):
            return redirect(url_for("login"))
        return f(*args, **kwargs)
    return wrap

@app.route("/", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        u = request.form.get("username")
        p = request.form.get("password")
        if u == config.DASHBOARD_USER and p == config.DASHBOARD_PASS:
            session["logged_in"] = True
            return redirect(url_for("dashboard"))
        flash("Username/password salah", "danger")
    return render_template("login.html", bot=config.BOT_NAME)

@app.route("/dashboard")
@login_required
def dashboard():
    stats = db.count_stats()
    txs = db.all_transactions()[:10]
    return render_template("dashboard.html", stats=stats, txs=txs, bot=config.BOT_NAME)

@app.route("/users")
@login_required
def users():
    return render_template("users.html", users=db.all_users(), bot=config.BOT_NAME)

@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("login"))

if __name__ == "__main__":
    db.init_db()
    app.run(debug=True, port=5000)
```

---

## 6. `templates/login.html`

```html
<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<title>Login - {{ bot }}</title>
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
<style>
  body{background:#0f172a;display:flex;align-items:center;justify-content:center;height:100vh;color:#fff}
  .card{background:#1e293b;border:none;border-radius:12px}
  .brand{font-weight:800;background:linear-gradient(90deg,#22d3ee,#a855f7);-webkit-background-clip:text;-webkit-text-fill-color:transparent}
</style>
</head>
<body>
<div class="card p-4 shadow" style="width:360px">
  <h3 class="text-center brand mb-3">{{ bot }}</h3>
  {% with messages = get_flashed_messages(with_categories=true) %}
    {% for cat, msg in messages %}
      <div class="alert alert-{{cat}} py-1">{{ msg }}</div>
    {% endfor %}
  {% endwith %}
  <form method="post">
    <div class="mb-3">
      <input type="text" name="username" class="form-control bg-dark text-white border-0" placeholder="Username" required>
    </div>
    <div class="mb-3">
      <input type="password" name="password" class="form-control bg-dark text-white border-0" placeholder="Password" required>
    </div>
    <button class="btn btn-primary w-100">Login</button>
  </form>
</div>
</body>
</html>
```

---

## 7. `templates/dashboard.html`

```html
<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<title>Dashboard - {{ bot }}</title>
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
<style>
  body{background:#0f172a;color:#e2e8f0;min-height:100vh}
  .card{background:#1e293b;border:none}
  .brand{font-weight:800;background:linear-gradient(90deg,#22d3ee,#a855f7);-webkit-background-clip:text;-webkit-text-fill-color:transparent}
  .stat{border-radius:12px;padding:20px}
  .s1{background:linear-gradient(135deg,#3b82f6,#1d4ed8)}
  .s2{background:linear-gradient(135deg,#10b981,#047857)}
  .s3{background:linear-gradient(135deg,#f59e0b,#b45309)}
</style>
</head>
<body class="p-4">
<div class="d-flex justify-content-between align-items-center mb-4">
  <h3 class="brand">{{ bot }} Dashboard</h3>
  <a href="{{ url_for('logout') }}" class="btn btn-outline-danger btn-sm">Logout</a>
</div>
<div class="row g-3 mb-4">
  <div class="col-md-4"><div class="stat s1"><div class="text-white-50">Total User</div><h3>{{ stats.users }}</h3></div></div>
  <div class="col-md-4"><div class="stat s2"><div class="text-white-50">Total Top Up</div><h3>Rp{{ "{:,}".format(stats.topup) }}</h3></div></div>
  <div class="col-md-4"><div class="stat s3"><div class="text-white-50">Total Order</div><h3>Rp{{ "{:,}".format(stats.order) }}</h3></div></div>
</div>

<div class="card p-3 mb-3">
  <div class="d-flex justify-content-between">
    <h5 class="brand">Transaksi Terbaru</h5>
    <a href="{{ url_for('users') }}" class="btn btn-sm btn-primary">Lihat User</a>
  </div>
</div>

<div class="card p-3">
  <table class="table table-dark table-sm">
    <thead><tr><th>User</th><th>Tipe</th><th>Jumlah</th><th>Status</th><th>Waktu</th></tr></thead>
    <tbody>
    {% for t in txs %}
      <tr>
        <td>{{ t.user_id }}</td>
        <td>{{ t.type }}</td>
        <td>Rp{{ "{:,}".format(t.amount) }}</td>
        <td><span class="badge bg-success">{{ t.status }}</span></td>
        <td>{{ t.created_at[:19] }}</td>
      </tr>
    {% endfor %}
    </tbody>
  </table>
</div>
</body>
</html>
```

---

## 8. `templates/users.html`

```html
<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<title>Users - {{ bot }}</title>
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
<style>body{background:#0f172a;color:#e2e8f0}.card{background:#1e293b;border:none}.brand{font-weight:800;background:linear-gradient(90deg,#22d3ee,#a855f7);-webkit-background-clip:text;-webkit-text-fill-color:transparent}</style>
</head>
<body class="p-4">
<div class="d-flex justify-content-between mb-3">
  <h3 class="brand">Daftar User {{ bot }}</h3>
  <a href="{{ url_for('dashboard') }}" class="btn btn-outline-light btn-sm">← Dashboard</a>
</div>
<div class="card p-3">
  <table class="table table-dark table-sm">
    <thead><tr><th>ID</th><th>Username</th><th>Saldo</th><th>Referral</th><th>Banned</th><th>Bergabung</th></tr></thead>
    <tbody>
    {% for u in users %}
      <tr>
        <td>{{ u.user_id }}</td>
        <td>@{{ u.username or '-' }}</td>
        <td>Rp{{ "{:,}".format(u.balance) }}</td>
        <td>{{ u.referral_code }}</td>
        <td>{{ "Ya" if u.is_banned else "Tidak" }}</td>
        <td>{{ u.created_at[:10] }}</td>
      </tr>
    {% endfor %}
    </tbody>
  </table>
</div>
</body>
</html>
```

---

## 🚀 Cara Menjalankan

1. **Buat bot** di Telegram via [@BotFather](https://t.me/BotFather), copy token, isi di `config.py`.
2. **Cari Telegram ID Anda** lewat [@userinfobot](https://t.me/userinfobot), isi `ADMIN_IDS`.
3. **Jalankan dashboard:**
   ```bash
   python app.py
   ```
   → buka `http://localhost:5000`
4. **Jalankan bot** (terminal lain):
   ```bash
   python bot.py
   ```
5. **Test bot:** kirim `/start` ke bot Anda.

---

## 📌 Catatan Tambahan

- **Sistem Top-Up** di atas masih simulasi (auto-kredit). Untuk production, integrasikan dengan payment gateway (Tripay, Midtrans, dll).
- **Order layanan** juga masih placeholder — tinggal disambungkan ke API backend WhatsApp Anda.
- Untuk deployment produksi, gunakan **Gunicorn + Nginx** untuk Flask, dan **systemd** / **PM2** untuk bot (jangan `run_polling` blocking).
- Ganti `SECRET_KEY`, password dashboard, dan jangan commit token bot ke repo publik.

