Ushbu talablarga to‘liq javob beradigan Flask dasturining to‘liq kodi va fayllar strukturasi quyidagicha.

📁 Loyiha strukturasi
Plaintext
my_flask_app/
├── app.py
├── static/
│   └── style.css
└── templates/
    └── index.html
1. Backend (app.py)
Python va Flask yordamida yozilgan asosiy fayl. Unda kamida 8 ta mahsulot ro'yxati va GET so'rovini qabul qilib, kichik/katta harf farqlamaydigan (case-insensitive) qidiruv logikasi yozilgan.

Python
from flask import Flask, render_template, request

app = Flask(__name__)

# Kamida 8 ta mahsulotdan iborat Python ro'yxati
products = [
    "Smartfon Samsung Galaxy S26 Ultra",
    "Noutbuk Apple MacBook Pro",
    "Quloqchin AirPods Pro",
    "Aqlli soat Smart Watch",
    "Planshet iPad Air",
    "Gaming Klaviatura RGB",
    "Sichqoncha Wireless Mouse",
    "Monitor 27 dyuym 4K",
    "Portativ Kolonka JBL",
    "Fleshka 128GB USB 3.0"
]

@app.route('/')
def index():
    # GET formadan 'q' parametrini olish
    query = request.args.get('q', '').strip()
    
    if query:
        # Katta va kichik harflarni farqlamasdan filtrlash
        filtered_products = [p for p in products if query.lower() in p.lower()]
    else:
        # Agar qidiruv bo'sh bo'lsa, barcha mahsulotlarni chiqarish (yoki bo'sh qoldirish mumkin)
        filtered_products = products
        
    return render_template('index.html', products=filtered_products, query=query)

if __name__ == '__main__':
    app.run(debug=True)
2. Frontend shabloni (templates/index.html)
GET metodiga ega forma va natijalarni ro'yxat ko'rinishida chiqaruvchi HTML fayl.

HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Mahsulot Qidiruvi</title>
    <!-- static/style.css ni url_for orqali ulash -->
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
    <div class="container">
        <h1>Mahsulot Qidirish</h1>
        
        <!-- GET metodi orqali ishlaydigan forma -->
        <form method="GET" action="/" class="search-form">
            <input type="text" name="q" value="{{ query }}" placeholder="Mahsulot nomini kiriting...">
            <button type="submit">Qidirish</button>
        </form>

        <div class="results">
            {% if products %}
                <h2>Qidiruv natijalari:</h2>
                <ul>
                    {% for product in products %}
                        <li>{{ product }}</li>
                    {% endfor %}
                </ul>
            {% else %}
                <!-- Bo'sh natija holati -->
                <p class="not-found">Hech narsa topilmadi</p>
            {% endif %}
        </div>
    </div>
</body>
</html>
3. Stil fayli (static/style.css)
Sahifani chiroyli ko'rinishga keltirish uchun CSS kodi.

CSS
body {
    font-family: Arial, sans-serif;
    background-color: #f4f7f6;
    margin: 0;
    padding: 50px;
    display: flex;
    justify-content: center;
}

.container {
    background: #ffffff;
    padding: 30px;
    border-radius: 10px;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
    width: 450px;
}

h1 {
    text-align: center;
    color: #333;
    margin-bottom: 25px;
}

.search-form {
    display: flex;
    gap: 10px;
    margin-bottom: 25px;
}

.search-form input {
    flex: 1;
    padding: 12px;
    border: 1px solid #ccc;
    border-radius: 6px;
    outline: none;
    font-size: 14px;
}

.search-form input:focus {
    border-color: #007bff;
}

.search-form button {
    padding: 12px 20px;
    background-color: #007bff;
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-size: 14px;
    transition: background 0.2s;
}

.search-form button:hover {
    background-color: #0056b3;
}

.results h2 {
    font-size: 16px;
    color: #555;
    margin-bottom: 10px;
}

.results ul {
    list-style-type: none;
    padding: 0;
    margin: 0;
}

.results li {
    background: #f9f9f9;
    margin-bottom: 8px;
    padding: 12px;
    border-left: 4px solid #007bff;
    border-radius: 4px;
    color: #333;
    font-size: 14px;
}

.not-found {
    color: #ff4d4d;
    text-align: center;
    font-weight: bold;
    font-size: 16px;
    margin-top: 20px;
}
🚀 Ishga tushirish uchun:
Kompyuteringizda Flask o'rnatilganligiga ishonch hosil qiling (pip install flask).

Yuqoridagi papka strukturasini tuzing va kodlarni tegishli fayllarga joylashtiring.

Terminalda quyidagi buyruqni bajaring:

Bash
python app.py
Brauzerni ochib [http://127.0.0.1:5000/](http://127.0.0.1:5000/) manziliga kiring.
