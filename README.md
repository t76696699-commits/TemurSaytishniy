Ushbu talablarga to‘liq javob beradigan Flask dasturining to‘liq kodi, fayllar strukturasi va bonus qismi (takroriy xabarlarni bloklash) bilan birgalikdagi yechimi.

📁 Loyiha strukturasi
Plaintext
guestbook_app/
├── app.py
├── templates/
│   └── index.html
└── login.html (yoki bitta shablon ichida birlashtirilgan)
1. Backend (app.py)
Flask yordamida yozilgan, sessiya, xavfsizlik va xabarlarni vaqt bo'yicha saqlash logikasini o'z ichiga olgan asosiy fayl.

Python
from flask import Flask, render_template, request, redirect, url_for, session, flash
from datetime import datetime

app = Flask(__name__)
app.secret_key = 'maxfiy_kalit_soz_bu_yerda'  # Session va flash uchun zarur

# Xabarlarni va foydalanuvchilarni vaqtincha xotirada saqlash uchun ro'yxatlar
messages = []

@app.route('/')
def index():
    # Oxirgi 20 ta xabarni olish va ularni teskari tartibda (eng yangisi tepada) chiqarish
    recent_messages = messages[-20:][::-1]
    return render_template('index.html', messages=recent_messages)

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        if username:
            session['user'] = username
            flash(f"Xush kelibsiz, {username}!", "success")
            return redirect(url_for('index'))
        else:
            flash("Iltimos, ismingizni kiriting!", "danger")
            
    return render_template('login.html')

@app.route('/post', methods=['POST'])
def post_message():
    # Login bo'lmagan foydalanuvchi xabar yoza olmasligi
    if 'user' not in session:
        flash("Xabar qoldirish uchun avval tizimga kiring!", "warning")
        return redirect(url_for('login'))
    
    text = request.form.get('text', '').strip()
    
    if text:
        author = session['user']
        current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        # BONUS: Bir xil matnli xabarni ketma-ket ikki marta yuborishni bloklash
        if messages and messages[-1]['text'] == text and messages[-1]['author'] == author:
            flash("Bir xil matnli xabarni ketma-ket ikki marta yuborish mumkin emas!", "warning")
        else:
            # Xabarni ro'yxatga qo'shish
            messages.append({
                'author': author,
                'text': text,
                'time': current_time
            })
            flash("Xabaringiz muvaffaqiyatli qo'shildi!", "success")
    else:
        flash("Xabar matni bo'sh bo'lishi mumkin emas!", "danger")
        
    return redirect(url_for('index'))

@app.route('/logout')
def logout():
    session.pop('user', None)
    flash("Siz tizimdan chiqdingiz.", "info")
    return redirect(url_for('index'))

if __name__ == '__main__':
    app.run(debug=True)
2. Bosh sahifa shabloni (templates/index.html)
Foydalanuvchi xabarlari ro'yxati, flash xabarlari va xabar yuborish formasini o'z ichiga olgan asosiy sahifa.

HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Mehmonlar Kitobi</title>
    <style>
        body { font-family: Arial, sans-serif; background: #f4f7f6; margin: 0; padding: 40px; display: flex; justify-content: center; }
        .container { background: #fff; padding: 30px; border-radius: 10px; width: 600px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        .header { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #ddd; padding-bottom: 15px; margin-bottom: 20px; }
        .flash { padding: 10px; margin-bottom: 15px; border-radius: 5px; background: #e2f0cb; color: #2d5a27; }
        .flash.danger { background: #ffccd5; color: #990000; }
        .flash.warning { background: #ffe8d6; color: #b75d00; }
        .message-box { background: #f9f9f9; padding: 12px; border-left: 4px solid #007bff; margin-bottom: 10px; border-radius: 4px; }
        .message-meta { font-size: 12px; color: #666; margin-top: 5px; display: flex; justify-content: space-between; }
        textarea, input[type="text"] { width: 100%; padding: 10px; margin-bottom: 10px; border: 1px solid #ccc; border-radius: 5px; box-sizing: border-box; }
        button { background: #007bff; color: white; border: none; padding: 10px 15px; border-radius: 5px; cursor: pointer; }
        button:hover { background: #0056b3; }
        .logout-btn { background: #dc3545; text-decoration: none; padding: 8px 12px; color: white; border-radius: 5px; font-size: 14px; }
        .logout-btn:hover { background: #a71d2a; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h2>Mehmonlar Kitobi</h2>
            <div>
                {% if 'user' in session %}
                    <span>Salom, <b>{{ session['user'] }}</b>!</span>
                    <a href="{{ url_for('logout') }}" class="logout-btn">Chiqish</a>
                {% else %}
                    <a href="{{ url_for('login') }}"><button>Kirish</button></a>
                {% endif %}
            </div>
        </div>

        <!-- Flash xabarlarni chiqarish -->
        {% with messages_flash = get_flashed_messages(with_categories=true) %}
            {% if messages_flash %}
                {% for category, msg in messages_flash %}
                    <div class="flash {{ category }}">{{ msg }}</div>
                {% endfor %}
            {% endif %}
        {% endwith %}

        <!-- Faqat login qilganlar xabar yoza oladi -->
        {% if 'user' in session %}
            <form action="{{ url_for('post_message') }}" method="POST">
                <textarea name="text" rows="3" placeholder="Xabaringizni yozing..." required></textarea>
                <button type="submit">Xabar qoldirish</button>
            </form>
        {% else %}
            <p><i>Xabar qoldirish uchun <a href="{{ url_for('login') }}">tizimga kiring</a>.</i></p>
        {% endif %}

        <h3>Oxirgi xabarlar:</h3>
        <div class="messages-list">
            {% if messages %}
                {% for msg in messages %}
                    <div class="message-box">
                        <div>{{ msg.text }}</div>
                        <div class="message-meta">
                            <span>Muallif: <b>{{ msg.author }}</b></span>
                            <span>{{ msg.time }}</span>
                        </div>
                    </div>
                {% endfor %}
            {% else %}
                <p>Hozircha xabarlar yo'q. Birinchi bo'lib yozing!</p>
            {% endif %}
        </div>
    </div>
</body>
</html>
3. Login sahifasi (templates/login.html)
Foydalanuvchi o'z nomini kiritib tizimga kirishi uchun kichik forma sahifasi.

HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Tizimga kirish</title>
    <style>
        body { font-family: Arial, sans-serif; background: #f4f7f6; margin: 0; padding: 100px; display: flex; justify-content: center; }
        .login-card { background: #fff; padding: 30px; border-radius: 10px; width: 350px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); text-align: center; }
        input { width: 100%; padding: 10px; margin-bottom: 15px; border: 1px solid #ccc; border-radius: 5px; box-sizing: border-box; }
        button { width: 100%; background: #28a745; color: white; border: none; padding: 10px; border-radius: 5px; cursor: pointer; }
        button:hover { background: #218838; }
        a { color: #007bff; text-decoration: none; font-size: 14px; display: inline-block; margin-top: 15px; }
    </style>
</head>
<body>
    <div class="login-card">
        <h2>Tizimga kirish</h2>
        <form action="{{ url_for('login') }}" method="POST">
            <input type="text" name="username" placeholder="Ismingizni kiriting..." required autocomplete="off">
            <button type="submit">Kirish</button>
        </form>
        <a href="{{ url_for('index') }}">← Bosh sahifaga qaytish</a>
    </div>
</body>
</html>
