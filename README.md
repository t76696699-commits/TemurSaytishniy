📁 Loyiha strukturasi
Plaintext
notes_project/
├── app/
│   ├── __init__.py
│   ├── models.py
│   ├── main/
│   │   └── routes.py
│   ├── notes/
│   │   └── routes.py
│   └── templates/
│       ├── base.html
│       ├── index.html
│       ├── about.html
│       ├── note_detail.html
│       └── note_form.html
├── instance/
│   └── notes.db
└── app.py
1. Asosiy ishga tushiruvchi fayl (app.py)
Talabga ko‘ra, app.py juda qisqa bo‘lib, faqat factory funksiyasini chaqiradi (3-5 qator).

Python
from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run(debug=True)
2. Ilova paketi va Factory (app/__init__.py)
Bu yerda create_app() funksiyasi ishga tushadi, ma'lumotlar bazasi (SQLAlchemy) ulanadi va Blueprint'lar ro'yxatdan o'tkaziladi.

Python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

def create_app():
    app = Flask(__name__)
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///notes.db'
    app.config['SECRET_KEY'] = 'super-secret-key-blueprint'

    db.init_app(app)

    # Blueprint'larni ro'yxatdan o'tkazish
    from app.main.routes import main_bp
    from app.notes.routes import notes_bp

    app.register_blueprint(main_bp)
    app.register_blueprint(notes_bp, url_prefix='/notes')

    # Bazani yaratish
    with app.app_context():
        db.create_all()

    return app
3. Ma'lumotlar bazasi modeli (app/models.py)
Python
from app import db

class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    content = db.Column(db.Text, nullable=False)
4. Asosiy Blueprint (app/main/routes.py)
Bosh sahifa va about sahifalari uchun.

Python
from flask import Blueprint, render_template
from app.models import Note

main_bp = Blueprint('main', __name__)

@main_bp.route('/')
def index():
    notes = Note.query.all()
    return render_template('index.html', notes=notes)

@main_bp.route('/about')
def about():
    return render_template('about.html')
5. Notlar Blueprint (app/notes/routes.py)
CRUD (Create, Read, Update, Delete) amallari uchun.

Python
from flask import Blueprint, render_template, request, redirect, url_for, flash
from app import db
from app.models import Note

notes_bp = Blueprint('notes', __name__)

@notes_bp.route('/new', methods=['GET', 'POST'])
def create_note():
    if request.method == 'POST':
        title = request.form['title']
        content = request.form['content']
        new_note = Note(title=title, content=content)
        db.session.add(new_note)
        db.session.commit()
        flash('Not muvaffaqiyatli qo\'shildi!', 'success')
        return redirect(url_for('main.index'))
    return render_template('note_form.html', note=None)

@notes_bp.route('/<int:id>')
def detail_note(id):
    note = Note.query.get_or_404(id)
    return render_template('note_detail.html', note=note)

@notes_bp.route('/<int:id>/edit', methods=['GET', 'POST'])
def edit_note(id):
    note = Note.query.get_or_404(id)
    if request.method == 'POST':
        note.title = request.form['title']
        note.content = request.form['content']
        db.session.commit()
        flash('Not tahrirlandi!', 'success')
        return redirect(url_for('notes.detail_note', id=note.id))
    return render_template('note_form.html', note=note)

@notes_bp.route('/<int:id>/delete', methods=['POST'])
def delete_note(id):
    note = Note.query.get_or_404(id)
    db.session.delete(note)
    db.session.commit()
    flash('Not o\'chirildi!', 'danger')
    return redirect(url_for('main.index'))
6. HTML Shablonlar (app/templates/)
A. Baza shablon (app/templates/base.html)
HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Notlar Ilovasi{% endblock %}</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f4f7f6; }
        .container { background: #fff; padding: 30px; border-radius: 8px; max-width: 600px; margin: auto; box-shadow: 0 4px 10px rgba(0,0,0,0.1); }
        .alert { padding: 10px; margin-bottom: 15px; border-radius: 4px; background: #d4edda; color: #155724; }
        .alert-danger { background: #f8d7da; color: #721c24; }
        nav a { margin-right: 15px; text-decoration: none; color: #007bff; }
    </style>
</head>
<body>
    <div class="container">
        <nav>
            <a href="{{ url_for('main.index') }}">Bosh sahifa</a>
            <a href="{{ url_for('notes.create_note') }}">Yangi not</a>
            <a href="{{ url_for('main.about') }}">Haqida</a>
        </nav>
        <hr>

        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category }}">{{ message }}</div>
                {% endfor %}
            {% endif %}
        {% endwith %}

        {% block content %}{% endblock %}
    </div>
</body>
</html>
B. Bosh sahifa (app/templates/index.html)
HTML
{% extends "base.html" %}
{% block content %}
<h1>Notlarim</h1>
<ul>
    {% for note in notes %}
        <li>
            <a href="{{ url_for('notes.detail_note', id=note.id) }}">{{ note.title }}</a>
        </li>
    {% else %}
        <p>Hozircha notlar mavjud emas.</p>
    {% endfor %}
</ul>
{% endblock %}
C. Forma shabloni (app/templates/note_form.html)
HTML
{% extends "base.html" %}
{% block content %}
<h2>{{ 'Notni tahrirlash' if note else 'Yangi not qo\'shish' }}</h2>
<form method="POST">
    <input type="text" name="title" value="{{ note.title if note else '' }}" placeholder="Sarlavha" required style="width:100%; padding:8px; margin-bottom:10px;"><br>
    <textarea name="content" placeholder="Matn" required style="width:100%; height:100px; padding:8px; margin-bottom:10px;">{{ note.content if note else '' }}</textarea><br>
    <button type="submit" style="padding:10px 15px; background:#007bff; color:#fff; border:none; border-radius:4px; cursor:pointer;">Saqlash</button>
</form>
{% endblock %}
D. Detal sahifasi (app/templates/note_detail.html)
HTML
{% extends "base.html" %}
{% block content %}
<h1>{{ note.title }}</h1>
<p>{{ note.content }}</p>
<hr>
<a href="{{ url_for('notes.edit_note', id=note.id) }}">Tahrirlash</a>
<form action="{{ url_for('notes.delete_note', id=note.id) }}" method="POST" style="display:inline;">
    <button type="submit" onclick="return confirm('O\'chirishni xohlaysizmi?')" style="background:red; color:white; border:none; padding:5px 10px; cursor:pointer;">O'chirish</button>
</form>
<br><br>
<a href="{{ url_for('main.index') }}">← Orqaga</a>
{% endblock %}
E. Haqida sahifasi (app/templates/about.html)
HTML
{% extends "base.html" %}
{% block content %}
<h2>Loyiha haqida</h2>
<p>Bu ilova Flask Blueprint va App Factory pattern asosida yaratilgan zamonaviy Notlar ilovasidir.</p>
{% endblock %}
