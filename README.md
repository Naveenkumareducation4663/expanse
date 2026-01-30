# expanse
from flask import Flask, render_template, request, redirect, url_for
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, login_required, logout_user, current_user
import hashlib

app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret123'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///database.db'

db = SQLAlchemy(app)
login_manager = LoginManager(app)
login_manager.login_view = 'login'

# ---------- DATABASE MODELS ----------

class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(100), unique=True)
    password = db.Column(db.String(200))

class Expense(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100))
    amount = db.Column(db.Integer)
    user_id = db.Column(db.Integer)

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

# ---------- ROUTES ----------

@app.route('/', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        email = request.form['email']
        pwd = hashlib.sha256(request.form['password'].encode()).hexdigest()

        user = User.query.filter_by(email=email, password=pwd).first()
        if user:
            login_user(user)
            return redirect('/dashboard')
    return render_template('login.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        email = request.form['email']
        pwd = hashlib.sha256(request.form['password'].encode()).hexdigest()

        user = User(email=email, password=pwd)
        db.session.add(user)
        db.session.commit()
        return redirect('/')
    return render_template('register.html')

@app.route('/dashboard', methods=['GET', 'POST'])
@login_required
def dashboard():
    if request.method == 'POST':
        title = request.form['title']
        amount = int(request.form['amount'])

        expense = Expense(title=title, amount=amount, user_id=current_user.id)
        db.session.add(expense)
        db.session.commit()

    expenses = Expense.query.filter_by(user_id=current_user.id).all()
    return render_template('dashboard.html', expenses=expenses)

@app.route('/delete/<int:id>')
@login_required
def delete(id):
    expense = Expense.query.get(id)
    db.session.delete(expense)
    db.session.commit()
    return redirect('/dashboard')

@app.route('/logout')
def logout():
    logout_user()
    return redirect('/')

# ---------- RUN ----------
if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(host='0.0.0.0', port=5000, debug=True)
