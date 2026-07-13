# Flask-10
from flask import Flask, render_template_string, request, redirect, url_for, flash
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

app = Flask(__name__)
app.secret_key = 'crud-ilovasi-maxfiy-kaliti'

app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///notes_v2.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)


class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    body = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)


@app.route('/')
@app.route('/notes')
def list_notes():
    notes = Note.query.order_by(Note.created_at.desc()).all()
    return render_template_string(LIST_TEMPLATE, notes=notes)


@app.route('/notes/new', methods=['GET', 'POST'])
def create_note():
    if request.method == 'POST':
        title = request.form.get('title', '').strip()
        body = request.form.get('body', '').strip()

        if not title or not body:
            flash("Sarlavha va matn bo'sh bo'lishi mumkin emas!", "error")
            return redirect(url_for('create_note'))

        note = Note(title=title, body=body)
        db.session.add(note)
        db.session.commit()

        flash("Yangi qayd muvaffaqiyatli qo'shildi!", "success")
        return redirect(url_for('list_notes'))

    return render_template_string(FORM_TEMPLATE, action="Yangi qayd yaratish", note=None)


@app.route('/notes/<int:id>')
def show_note(id):
    note = Note.query.get_or_404(id)
    return render_template_string(DETAIL_TEMPLATE, note=note)


@app.route('/notes/<int:id>/edit', methods=['GET', 'POST'])
def edit_note(id):
    note = Note.query.get_or_404(id)

    if request.method == 'POST':
        note.title = request.form.get('title', '').strip()
        note.body = request.form.get('body', '').strip()

        if not note.title or not note.body:
            flash("Maydonlarni bo'sh qoldirmang!", "error")
            return redirect(url_for('edit_note', id=note.id))

        db.session.commit()
        flash("Qayd muvaffaqiyatli yangilandi!", "success")
        return redirect(url_for('show_note', id=note.id))

    return render_template_string(FORM_TEMPLATE, action="Qaydni tahrirlash", note=note)


@app.route('/notes/<int:id>/delete', methods=['POST'])
def delete_note(id):
    note = Note.query.get_or_404(id)
    db.session.delete(note)
    db.session.commit()

    flash("Qayd muvaffaqiyatli o'chirildi!", "success")
    return redirect(url_for('list_notes'))



BASE_CSS = '''
body { font-family: Arial, sans-serif; max-width: 600px; margin: 40px auto; padding: 0 20px; background-color: #f4f6f9; color: #333; }
.flash { padding: 10px; margin-bottom: 15px; border-radius: 4px; font-weight: bold; }
.success { background-color: #d4edda; color: #155724; border-left: 5px solid #28a745; }
.error { background-color: #f8d7da; color: #721c24; border-left: 5px solid #dc3545; }
.card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); margin-bottom: 15px; }
.btn { display: inline-block; padding: 8px 12px; border: none; border-radius: 4px; cursor: pointer; text-decoration: none; font-size: 0.9em; }
.btn-primary { background: #007bff; color: white; }
.btn-danger { background: #dc3545; color: white; }
.btn-secondary { background: #6c757d; color: white; }
input[type="text"], textarea { width: 100%; padding: 8px; margin: 8px 0 15px 0; box-sizing: border-box; border: 1px solid #ccc; border-radius: 4px; }
textarea { height: 120px; resize: vertical; }
.meta { font-size: 0.8em; color: #777; margin-top: 10px; }
'''

FLASH_BLOCK = '''
{% with messages = get_flashed_messages(with_categories=true) %}
  {% if messages %}
    {% for category, message in messages %}
      <div class="flash {{ category }}">{{ message }}</div>
    {% endfor %}
  {% endif %}
{% endwith %}
'''

LIST_TEMPLATE = f'''
<!DOCTYPE html>
<html>
<head><title>Notlar CRUD</title><style>{BASE_CSS}</style></head>
<body>
    <h1>📝 Notlar CRUD Ilovasi</h1>
    {FLASH_BLOCK}
    <p><a href="{{{{ url_for('create_note') }}}}" class="btn btn-primary">+ Yangi qayd qo'shish</a></p>
    <hr>
    {{% if notes %}}
        {{% for note in notes %}}
            <div class="card">
                <h3><a href="{{{{ url_for('show_note', id=note.id) }}}}" style="color: #007bff; text-decoration: none;">{{{{ note.title }}}}</a></h3>
                <p>{{{{ note.body[:100] }}}}{{% if note.body|length > 100 %}}...{{% endif %}}</p>
                <div class="meta">🕒 {{{{ note.created_at.strftime('%H:%M | %d-%b-%Y') }}}}</div>
            </div>
        {{% endfor %}}
    {{% else %}}
        <p style="color: #777; text-align: center;">Hozircha hech qanday qayd yo'q.</p>
    {{% endif %}}
</body>
</html>
'''


DETAIL_TEMPLATE = f'''
<!DOCTYPE html>
<html>
<head><title>Not detali</title><style>{BASE_CSS}</style></head>
<body>
    {FLASH_BLOCK}
    <div class="card">
        <h2>{{{{ note.title }}}}</h2>
        <p style="line-height: 1.6; white-space: pre-line;">{{{{ note.body }}}}</p>
        <div class="meta">Yaratilgan vaqti: {{{{ note.created_at.strftime('%H:%M | %d-%b-%Y') }}}}</div>
        <hr>
        <div style="display: flex; gap: 10px;">
            <a href="{{{{ url_for('edit_note', id=note.id) }}}}" class="btn btn-primary">Tahrirlash</a>

            <form action="{{{{ url_for('delete_note', id=note.id) }}}}" method="POST" onsubmit="return confirm('Haqiqatan ham ocha olasizmi?');">
                <button type="submit" class="btn btn-danger">O'chirish</button>
            </form>

            <a href="{{{{ url_for('list_notes') }}}}" class="btn btn-secondary">Orqaga</a>
        </div>
    </div>
</body>
</html>
'''


FORM_TEMPLATE = f'''
<!DOCTYPE html>
<html>
<head><title>Forma</title><style>{BASE_CSS}</style></head>
<body>
    <h2>{{{{ action }}}}</h2>
    {FLASH_BLOCK}
    <div class="card">
        <form method="POST">
            <label>Sarlavha:</label>
            <input type="text" name="title" value="{{{{ note.title if note else '' }}}}" required>

            <label>Matn:</label>
            <textarea name="body" required>{{{{ note.body if note else '' }}}}</textarea>

            <button type="submit" class="btn btn-primary">Saqlash</button>
            <a href="{{{{ url_for('list_notes') }}}}" class="btn btn-secondary">Bekor qilish</a>
        </form>
    </div>
</body>
</html>
'''

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=True)
