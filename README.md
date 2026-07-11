১. app.py

import os
import sqlite3
from flask import Flask, render_template, request, redirect, session, url_for
from telegram import Update
from telegram.ext import Application, CommandHandler, ContextTypes

BOT_TOKEN = os.getenv('BOT_TOKEN', 'YOUR_BOT_TOKEN')
SECRET_KEY = os.getenv('SECRET_KEY', 'secret123')

app = Flask(__name__)
app.secret_key = SECRET_KEY

# ---------------- DATABASE ----------------
conn = sqlite3.connect('database.db', check_same_thread=False)
cur = conn.cursor()

cur.execute('''
CREATE TABLE IF NOT EXISTS users(
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE,
    password TEXT,
    ref_code TEXT UNIQUE,
    referred_by TEXT
)
''')

conn.commit()

# ---------------- HELPER ----------------
def get_user(username):
    cur.execute('SELECT * FROM users WHERE username=?', (username,))
    return cur.fetchone()

# ---------------- WEB ROUTES ----------------
@app.route('/')
def home():
    return render_template('index.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        referred_by = request.form.get('ref', '')

        ref_code = username.upper()

        try:
            cur.execute(
                'INSERT INTO users(username,password,ref_code,referred_by) VALUES(?,?,?,?)',
                (username, password, ref_code, referred_by)
            )
            conn.commit()
            return redirect('/login')
        except:
            return 'Username already exists'

    return render_template('register.html', ref=request.args.get('ref', ''))

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']

        user = get_user(username)

        if user and user[2] == password:
            session['user'] = username
            return redirect('/dashboard')

        return 'Invalid credentials'

    return render_template('login.html')

@app.route('/dashboard')
def dashboard():
    if 'user' not in session:
        return redirect('/login')

    user = get_user(session['user'])
    ref_link = request.host_url + 'register?ref=' + user[3]

    cur.execute('SELECT COUNT(*) FROM users WHERE referred_by=?', (user[3],))
    total_refs = cur.fetchone()[0]

    return render_template(
        'dashboard.html',
        username=user[1],
        ref_link=ref_link,
        total_refs=total_refs
    )

@app.route('/logout')
def logout():
    session.clear()
    return redirect('/')

@app.route('/admin')
def admin():
    cur.execute('SELECT username, ref_code, referred_by FROM users')
    users = cur.fetchall()
    return render_template('admin.html', users=users)

# ---------------- TELEGRAM BOT ----------------
bot = Application.builder().token(BOT_TOKEN).build()

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user

    me = await bot.bot.get_me()
    link = f'https://t.me/{me.username}?start={user.id}'

    await update.message.reply_text(
        f'স্বাগতম {user.first_name}!\\n\\n'
        f'আপনার Referral Link:\\n{link}'
    )

bot.add_handler(CommandHandler('start', start))

# ---------------- RUN ----------------
if __name__ == '__main__':
    import threading

    def run_bot():
        bot.run_polling()

    threading.Thread(target=run_bot).start()

    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 10000)))

---

২. requirements.txt

flask
python-telegram-bot==21.6
gunicorn

---

৩. render.yaml

services:
  - type: web
    name: forhad-freelancer
    runtime: python
    buildCommand: pip install -r requirements.txt
    startCommand: gunicorn app:app
    envVars:
      - key: BOT_TOKEN
        sync: false
      - key: SECRET_KEY
        sync: false

---

৪. templates/index.html

<!DOCTYPE html>
<html>
<head>
    <title>Forhad Freelancer</title>
</head>
<body>
    <h1>Forhad Freelancer</h1>
    <a href='/register'>Register</a> |
    <a href='/login'>Login</a>
</body>
</html>

---

৫. templates/register.html

<!DOCTYPE html>
<html>
<head>
    <title>Register</title>
</head>
<body>
<h2>Register</h2>

<form method='post'>
    <input name='username' placeholder='Username'><br><br>
    <input type='password' name='password' placeholder='Password'><br><br>

    <input name='ref' value='{{ ref }}' placeholder='Referral Code'><br><br>

    <button type='submit'>Create Account</button>
</form>

</body>
</html>

---

৬. templates/login.html

<!DOCTYPE html>
<html>
<head>
    <title>Login</title>
</head>
<body>
<h2>Login</h2>

<form method='post'>
    <input name='username' placeholder='Username'><br><br>
    <input type='password' name='password' placeholder='Password'><br><br>
    <button type='submit'>Login</button>
</form>

</body>
</html>

---

৭. templates/dashboard.html

<!DOCTYPE html>
<html>
<head>
    <title>Dashboard</title>
</head>
<body>

<h2>Welcome {{ username }}</h2>

<p><b>Your Referral Link:</b></p>

<input value='{{ ref_link }}' style='width:300px' readonly>

<p><b>Total Referrals:</b> {{ total_refs }}</p>

<br>

<a href='/logout'>Logout</a>

</body>
</html>

---

৮. templates/admin.html

<!DOCTYPE html>
<html>
<head>
    <title>Admin</title>
</head>
<body>

<h2>All Users</h2>

<table border='1' cellpadding='8'>
<tr>
    <th>Username</th>
    <th>Ref Code</th>
    <th>Referred By</th>
</tr>

{% for u in users %}
<tr>
    <td>{{ u[0] }}</td>
    <td>{{ u[1] }}</td>
    <td>{{ u[2] }}</td>
</tr>
{% endfor %}

</table>

</body>
</html>

---

৯. Render-এ ডিপ্লয়

1. GitHub-এ নতুন Repository তৈরি করুন।
2. সব ফাইল আপলোড করুন।
3. Render.com → New Web Service।
4. Repository Connect করুন।
5. Environment Variables দিন:
   - "BOT_TOKEN" = BotFather এর টোকেন
   - "SECRET_KEY" = যেকোনো গোপন স্ট্রিং
6. Deploy চাপুন।

---

১০. ব্যবহার

- "/register" → নতুন অ্যাকাউন্ট তৈরি
- "/login" → লগইন
- "/dashboard" → নিজের রেফারেল লিংক দেখুন
- "/admin" → সব ইউজার দেখুন
- Telegram Bot এ "/start" দিলে রেফারেল লিংক পাবেন

এটি একটি শিক্ষামূলক স্টার্টার প্রজেক্ট। বাস্তবে ব্যবহার করার আগে অবশ্যই পাসওয়ার্ড হ্যাশিং, ইমেইল ভেরিফিকেশন, CSRF সুরক্ষা এবং সঠিক অ্যাডমিন অথেন্টিকেশন যোগ করা উচিত.# forhad
This is for my android apps 
