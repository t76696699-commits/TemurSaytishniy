from flask import Flask, request, jsonify, abort
from flask_sqlalchemy import SQLAlchemy
from flask_cors import CORS

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///api.db'
db = SQLAlchemy(app)
CORS(app, resources={r"/api/*": {"origins": "*"}})

class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(120), nullable=False)
    body = db.Column(db.Text, default='')

    def to_dict(self):
        return {'id': self.id, 'title': self.title, 'body': self.body}

@app.route('/api/notes', methods=['GET'])
def list_notes():
    return jsonify([n.to_dict() for n in Note.query.all()])

@app.route('/api/notes/<int:id>', methods=['GET'])
def get_note(id):
    n = Note.query.get_or_404(id)
    return jsonify(n.to_dict())

@app.route('/api/notes', methods=['POST'])
def create_note():
    data = request.get_json()
    if not data or not data.get('title'):
        return jsonify({'error': 'title shart'}), 400
    n = Note(title=data['title'], body=data.get('body', ''))
    db.session.add(n)
    db.session.commit()
    return jsonify(n.to_dict()), 201

@app.route('/api/notes/<int:id>', methods=['DELETE'])
def delete_note(id):
    n = Note.query.get_or_404(id)
    db.session.delete(n)
    db.session.commit()
    return '', 204

@app.errorhandler(404)
def not_found(e):
    return jsonify({'error': 'Topilmadi'}), 404
