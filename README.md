app.pyfrom flask import Flask, request, jsonify, render_template, redirect, url_for, session
import requests
import sqlite3
import base64
import datetime

app = Flask(__name__)
app.secret_key = "supersecretkey"  # change this later for security

# ==========================
# DATABASE SETUP
# ==========================
def init_db():
    conn = sqlite3.connect("database.db")
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS transactions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            phone TEXT,
            amount TEXT,
            status TEXT,
            mpesa_receipt TEXT,
            date TEXT
        )
    """)
    c.execute("""
        CREATE TABLE IF NOT EXISTS admin (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE,
            password TEXT
        )
    """)
    # default admin (username: admin, password: admin)
    c.execute("INSERT OR IGNORE INTO admin (id, username, password) VALUES (1, 'admin', 'admin')")
    conn.commit()
    conn.close()

init_db()

# ==========================
# MPESA CONFIG
# ==========================
CONSUMER_KEY = "YOUR_CONSUMER_KEY"
CONSUMER_SECRET = "YOUR_CONSUMER_SECRET"
SHORTCODE = "YOUR_TILL_NUMBER"
PASSKEY = "YOUR_PASSKEY"   # from Safaricom Daraja
CALLBACK_URL = "https://your-backend-url.com/callback"

def get_access_token():
    api_url = "https://sandbox.safaricom.co.ke/oauth/v1/generate?grant_type=client_credentials"
    r = requests.get(api_url, auth=(CONSUMER_KEY, CONSUMER_SECRET))
    return r.json()["access_token"]

# ==========================
# ROUTES
# ==========================
@app.route("/")
def home():
    return "Bramtech Backend Running ✅"

# STK PUSH
@app.route("/pay", methods=["POST"])
def pay():
    data = request.json
    phone = data["phone"]
    amount = data["amount"]

    access_token = get_access_token()
    timestamp = datetime.datetime.now().strftime("%Y%m%d%H%M%S")
    password = base64.b64encode((SHORTCODE + PASSKEY + timestamp).encode()).decode("utf-8")

    payload = {
        "BusinessShortCode": SHORTCODE,
        "Password": password,
        "Timestamp": timestamp,
        "TransactionType": "CustomerPayBillOnline",
        "Amount": amount,
        "PartyA": phone,
        "PartyB": SHORTCODE,
        "PhoneNumber": phone,
        "CallBackURL": CALLBACK_URL,
        "AccountReference": "Bramtech",
        "TransactionDesc": "Payment"
    }

    headers = {"Authorization": f"Bearer {access_token}"}
    response = requests.post("https://sandbox.safaricom.co.ke/mpesa/stkpush/v1/processrequest",
                             json=payload, headers=headers)

    # Save transaction as "Pending"
    conn = sqlite3.connect("database.db")
    c = conn.cursor()
    c.execute("INSERT INTO transactions (phone, amount, status, mpesa_receipt, date) VALUES (?, ?, ?, ?, ?)",
              (phone, amount, "Pending", "-", str(datetime.datetime.now())))
    conn.commit()
    conn.close()

    return jsonify(response.json())

# CALLBACK
@app.route("/callback", methods=["POST"])
def callback():
    data = request.json
    result_code = data["Body"]["stkCallback"]["ResultCode"]
    mpesa_receipt = "-"
    status = "Failed"

    if result_code == 0:
        status = "Success"
        mpesa_receipt = data["Body"]["stkCallback"]["CallbackMetadata"]["Item"][1]["Value"]

    phone = data["Body"]["stkCallback"]["CallbackMetadata"]["Item"][4]["Value"]
    amount = data["Body"]["stkCallback"]["CallbackMetadata"]["Item"][0]["Value"]

    conn = sqlite3.connect("database.db")
    c = conn.cursor()
    c.execute("INSERT INTO transactions (phone, amount, status, mpesa_receipt, date) VALUES (?, ?, ?, ?, ?)",
              (phone, str(amount), status, mpesa_receipt, str(datetime.datetime.now())))
    conn.commit()
    conn.close()

    return jsonify({"ResultCode": 0, "ResultDesc": "Accepted"})

# ADMIN LOGIN
@app.route("/admin", methods=["GET", "POST"])
def admin_login():
    if request.method == "POST":
        username = request.form["username"]
        password = request.form["password"]

        conn = sqlite3.connect("database.db")
        c = conn.cursor()
        c.execute("SELECT * FROM admin WHERE username=? AND password=?", (username, password))
        user = c.fetchone()
        conn.close()

        if user:
            session["admin"] = username
            return redirect(url_for("dashboard"))
    return render_template("login.html")

# DASHBOARD
@app.route("/dashboard")
def dashboard():
    if "admin" not in session:
        return redirect(url_for("admin_login"))

    conn = sqlite3.connect("database.db")
    c = conn.cursor()
    c.execute("SELECT * FROM transactions ORDER BY id DESC")
    transactions = c.fetchall()
    conn.close()

    return render_template("dashboard.html", transactions=transactions)

# LOGOUT
@app.route("/logout")
def logout():
    session.pop("admin", None)
    return redirect(url_for("admin_login"))

if __name__ == "__main__":
    app.run(debug=True)# brammtech-frontend
Flask backend for Bramtech communication app
