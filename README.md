Ushbu loyiha uchun SQLAlchemy orqali ma'lumotlar bazasini boshqarish va CRUD (Create, Read, Update, Delete) operatsiyalarini amalga oshirish logikasini ko'rib chiqamiz.

📁 Loyiha strukturasi
Plaintext
notes_app/
├── app.py
├── templates/
│   ├── index.html
│   ├── note_detail.html
│   ├── note_form.html
└── ...
1. Backend (app.py)
Model yaratish va har bir route uchun mantiq.

Python
from flask import Flask, render_template, request, redirect, url_for, flash, abort
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///notes.db'
app.config['SECRET_KEY'] = 'super-secret-key'
db = SQLAlchemy(app)

class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    content = db.Column(db.Text, nullable=False)

# Bazani yaratish (terminalda bir marta bajarish kerak)
with app.app_context():
    db.create_all()

@app.route('/')
def index():
    notes = Note.query.all()
    return render_template('index.html', notes=notes)

@app.route('/notes/new', methods=['GET', 'POST'])
def create_note():
    if request.method == 'POST':
        new_note = Note(title=request.form['title'], content=request.form['content'])
        db.session.add(new_note)
        db.session.commit()
        flash('Not muvaffaqiyatli qo\'shildi!', 'success')
        return redirect(url_for('index'))
    return render_template('note_form.html', note=None)

@app.route('/notes/<int:id>')
def detail_note(id):
    note = Note.query.get_or_404(id)
    return render_template('note_detail.html', note=note)

@app.route('/notes/<int:id>/edit', methods=['GET', 'POST'])
def edit_note(id):
    note = Note.query.get_or_404(id)
    if request.method == 'POST':
        note.title = request.form['title']
        note.content = request.form['content']
        db.session.commit()
        flash('Not tahrirlandi!', 'success')
        return redirect(url_for('detail_note', id=note.id))
    return render_template('note_form.html', note=note)

@app.route('/notes/<int:id>/delete', methods=['POST'])
def delete_note(id):
    note = Note.query.get_or_404(id)
    db.session.delete(note)
    db.session.commit()
    flash('Not o\'chirildi!', 'danger')
    return redirect(url_for('index'))
2. Frontend (Forma shabloni: note_form.html)
Bu shablon ham yangi not yaratish, ham tahrirlash uchun ishlatiladi.

HTML
<form method="POST">
    <input type="text" name="title" value="{{ note.title if note else '' }}" placeholder="Sarlavha" required>
    <textarea name="content" placeholder="Matn" required>{{ note.content if note else '' }}</textarea>
    <button type="submit">Saqlash</button>
</form>
3. Bosh sahifa (index.html)
HTML
{% with messages = get_flashed_messages(with_categories=true) %}
  {% for category, message in messages %}
    <div class="alert alert-{{ category }}">{{ message }}</div>
  {% endfor %}
{% endwith %}

<h1>Notlarim</h1>
<a href="{{ url_for('create_note') }}">Yangi not</a>
<ul>
    {% for note in notes %}
    <li>
        <a href="{{ url_for('detail_note', id=note.id) }}">{{ note.title }}</a>
    </li>
    {% endfor %}
</ul>
4. Detal va O'chirish (note_detail.html)
HTML
<h1>{{ note.title }}</h1>
<p>{{ note.content }}</p>

<a href="{{ url_for('edit_note', id=note.id) }}">Tahrirlash</a>

<!-- Delete uchun POST form -->
<form action="{{ url_for('delete_note', id=note.id) }}" method="POST" style="display:inline;">
    <button type="submit" onclick="return confirm('Ishonchingiz komilmi?')">O'chirish</button>
</form>

<a href="{{ url_for('index') }}">Orqaga</a>
