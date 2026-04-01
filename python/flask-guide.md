# Flask Web Framework Reference Guide (Python + SQLite)

## Table of Contents
1. [Core Flask Functions & Methods](#1-core-flask-functions--methods)
2. [\_\_init\_\_.py — Application Factory](#2-__init__py--application-factory)
3. [main.py / app.py — Entry Point](#3-mainpy--apppy--entry-point)
4. [Templates — Jinja2 Directives](#4-templates--jinja2-directives)
5. [Routes & URL Building](#5-routes--url-building)
6. [Forms — Flask-WTF & WTForms](#6-forms--flask-wtf--wtforms)
7. [Authentication — Flask-Login](#7-authentication--flask-login)
8. [CRUD Operations with SQLite](#8-crud-operations-with-sqlite)

---

## 1. Core Flask Functions & Methods

| # | Function / Method | Description | Example | Notes |
| :---: | :--- | :--- | :--- | :--- |
| 1 | `render_template()` | Renders a Jinja2 HTML template from the `templates/` folder and returns it as a response | `return render_template('index.html', title='Home')` | Pass keyword args to expose variables in the template. |
| 2 | `redirect()` | Returns a redirect response to another URL | `return redirect(url_for('home'))` | Always use with `url_for()` to avoid hard-coded paths. |
| 3 | `url_for()` | Builds a URL for a named view function | `url_for('profile', username='alice')` | Avoids hard-coding URLs; handles query strings automatically. |
| 4 | `flash()` | Queues a one-time message to display on the next request | `flash('Saved!', 'success')` | Read in templates with `get_flashed_messages()`. Categories: `'success'`, `'error'`, `'info'`. |
| 5 | `request.form` | Dictionary of POST form data | `name = request.form.get('name')` | Available only during a POST/PUT request. |
| 6 | `request.args` | Dictionary of URL query-string parameters | `page = request.args.get('page', 1)` | Available on any request method. |
| 7 | `request.method` | HTTP method of the current request as a string | `if request.method == 'POST':` | Common values: `'GET'`, `'POST'`, `'PUT'`, `'DELETE'`. |
| 8 | `jsonify()` | Converts a dict or list to a JSON `Response` object | `return jsonify({'status': 'ok', 'data': rows})` | Sets `Content-Type: application/json` automatically. |
| 9 | `abort()` | Immediately raises an HTTP error response | `abort(404)` | Use `abort(403)` for forbidden, `abort(500)` for server errors. |
| 10 | `make_response()` | Creates a `Response` object to set headers or cookies | `resp = make_response(render_template('page.html')); resp.set_cookie('token', val)` | Needed when you must modify the response before returning it. |
| 11 | `g` | Application-context global for storing per-request data | `g.db = sqlite3.connect(DATABASE)` | Cleared automatically after each request; ideal for DB connections. |
| 12 | `current_app` | Proxy to the active Flask application instance | `current_app.config['SECRET_KEY']` | Use inside helpers/blueprints where `app` is not in scope. |
| 13 | `app.config[]` | Stores application configuration values | `app.config['DEBUG'] = True` | Load from a `.env` file or config object for production use. |
| 14 | `app.route()` | Decorator that maps a URL rule to a view function | `@app.route('/users', methods=['GET','POST'])` | `methods` defaults to `['GET']`. |
| 15 | `app.app_context()` | Context manager that pushes an application context | `with app.app_context(): db.init_db()` | Required when calling Flask code outside a request (e.g., CLI). |
| 16 | `app.teardown_appcontext()` | Registers a function to run when the app context tears down | `app.teardown_appcontext(close_db)` | Use to reliably close DB connections after every request. |
| 17 | `Blueprint()` | Creates a reusable group of routes, templates, and static files | `auth = Blueprint('auth', __name__)` | Register on the app with `app.register_blueprint(auth, url_prefix='/auth')`. |
| 18 | `session[]` | Cookie-based dictionary for storing per-user data across requests | `session['user_id'] = user.id` | Requires `SECRET_KEY` to sign the cookie. |
| 19 | `send_from_directory()` | Safely sends a file from a given directory | `return send_from_directory('uploads', filename)` | Prevents path-traversal attacks. |
| 20 | `before_request()` | Registers a function to run before every request | `@app.before_request def check_auth(): ...` | Use to enforce login checks globally. |
| 21 | `after_request()` | Registers a function to run after every request (receives response) | `@app.after_request def add_header(resp): ...` | Must return the response object. |
| 22 | `errorhandler()` | Registers a custom handler for an HTTP error code | `@app.errorhandler(404) def not_found(e): return render_template('404.html'), 404` | Catches the error before Flask returns a default response. |
| 23 | `get_flashed_messages()` | Retrieves and clears queued flash messages | `{% for msg in get_flashed_messages(with_categories=True) %}` | Call inside a Jinja2 template; `with_categories=True` returns `(category, msg)` tuples. |
| 24 | `request.json` | Parses the request body as JSON | `data = request.json` | Returns `None` if body is not valid JSON; use for REST APIs. |

---

## 2. `__init__.py` — Application Factory

The `__init__.py` inside the `app/` package is where the Flask application is **created and configured**.  
Using the *Application Factory* pattern (`create_app()`) makes the app testable and avoids circular imports.

```python
# app/__init__.py
from flask import Flask
from dotenv import load_dotenv
import os

load_dotenv()  # Load environment variables from .env / .flaskenv

def create_app():
    app = Flask(__name__)

    # ── Configuration ──────────────────────────────────────────────────────────
    app.config['SECRET_KEY'] = os.getenv('SECRET_KEY', 'change-me')
    app.config['DATABASE']   = os.path.join(app.instance_path, 'app.db')

    # ── Database ───────────────────────────────────────────────────────────────
    from app import database
    with app.app_context():
        database.init_db()
    app.teardown_appcontext(database.close_connection)

    # ── Blueprints / Routes ────────────────────────────────────────────────────
    from app import routes
    routes.init_app(app)

    # Optional: register an auth blueprint
    # from app.auth import auth_bp
    # app.register_blueprint(auth_bp, url_prefix='/auth')

    return app
```

| Key line | Purpose |
| :--- | :--- |
| `Flask(__name__)` | Creates the Flask app; `__name__` sets the root path for templates/static. |
| `app.config['SECRET_KEY']` | Required for sessions and CSRF protection. |
| `with app.app_context()` | Pushes a context so database helpers can use `g` and `current_app`. |
| `app.teardown_appcontext()` | Ensures the SQLite connection is closed after every request. |

---

## 3. `main.py` / `app.py` — Entry Point

`app.py` (or `main.py`) is the **thin entry point** that calls `create_app()` and starts the development server.

```python
# app.py  (or main.py)
from app import create_app
import os

app = create_app()

if __name__ == '__main__':
    port = int(os.getenv('PORT', 5000))
    app.run(host='0.0.0.0', port=port, debug=True)
```

**`.flaskenv` file** — environment variables loaded by `python-dotenv`:

```
FLASK_APP=app.py
FLASK_ENV=development
FLASK_DEBUG=1
SECRET_KEY=my-secret-key
```

**Running the app:**

```bash
flask run              # Uses FLASK_APP from .flaskenv
flask run --port 8080  # Custom port
python app.py          # Direct execution
```

---

## 4. Templates — Jinja2 Directives

Flask uses **Jinja2** as its templating engine. Templates live in the `app/templates/` directory.

### Common Jinja2 Syntax

| # | Directive | Description | Example |
| :---: | :--- | :--- | :--- |
| 1 | `{{ variable }}` | Renders a Python variable as text | `<h1>Hello, {{ username }}!</h1>` |
| 2 | `{% if %} … {% endif %}` | Conditional block | `{% if user %} Logged in {% endif %}` |
| 3 | `{% for item in list %} … {% endfor %}` | Loops over an iterable | `{% for city in cities %} <li>{{ city.name }}</li> {% endfor %}` |
| 4 | `{% extends "base.html" %}` | Inherits layout from another template | Must be the **first** tag in the child template. |
| 5 | `{% block content %} … {% endblock %}` | Defines/overrides a named content block | Used together with `extends`. |
| 6 | `{% include "nav.html" %}` | Inserts another template inline | Good for headers, footers, nav bars. |
| 7 | `{{ url_for('route_name') }}` | Generates a URL for a view function | `<a href="{{ url_for('home') }}">Home</a>` |
| 8 | `{{ form.csrf_token }}` | Renders the hidden CSRF token field | Required in every Flask-WTF form. |
| 9 | `{% with messages = get_flashed_messages(with_categories=True) %}` | Reads flash messages | Wrap in `{% if messages %}` check. |
| 10 | `{{ value \| filter }}` | Applies a Jinja2 filter | `{{ name \| upper }}`, `{{ price \| round(2) }}` |

### Minimal `base.html`

```html
<!-- app/templates/base.html -->
<!DOCTYPE html>
<html>
<head>
  <title>{% block title %}My App{% endblock %}</title>
</head>
<body>
  <!-- Flash messages -->
  {% with messages = get_flashed_messages(with_categories=True) %}
    {% if messages %}
      {% for category, message in messages %}
        <div class="alert alert-{{ category }}">{{ message }}</div>
      {% endfor %}
    {% endif %}
  {% endwith %}

  {% block content %}{% endblock %}
</body>
</html>
```

### Child template example

```html
<!-- app/templates/index.html -->
{% extends "base.html" %}

{% block title %}Home{% endblock %}

{% block content %}
  <h1>Welcome!</h1>
  <ul>
    {% for city in cities %}
      <li>{{ city['city'] }} — {{ city['population'] | int }}</li>
    {% endfor %}
  </ul>
{% endblock %}
```

---

## 5. Routes & URL Building

Routes map **URL patterns** to Python view functions using the `@app.route()` decorator.

### Route Basics

```python
# app/routes.py
from flask import render_template, redirect, url_for, request, flash, abort

def init_app(app):

    # ── Simple GET route ───────────────────────────────────────────────────────
    @app.route('/')
    def home():
        return render_template('index.html', title='Home')

    # ── Route with URL variable ────────────────────────────────────────────────
    @app.route('/user/<int:user_id>')
    def user_profile(user_id):
        user = database.get_user(user_id)
        if not user:
            abort(404)
        return render_template('profile.html', user=user)

    # ── Route accepting GET and POST ───────────────────────────────────────────
    @app.route('/submit', methods=['GET', 'POST'])
    def submit():
        if request.method == 'POST':
            name = request.form.get('name')
            flash(f'Hello {name}!', 'success')
            return redirect(url_for('home'))
        return render_template('submit.html')

    # ── JSON / API route ───────────────────────────────────────────────────────
    @app.route('/api/cities')
    def api_cities():
        cities = database.get_all_cities()
        return jsonify([dict(c) for c in cities])
```

### URL Variable Converters

| Converter | Description | Example |
| :--- | :--- | :--- |
| `<string:name>` | Default; accepts any text without a slash | `/user/<string:username>` |
| `<int:id>` | Accepts positive integers | `/post/<int:post_id>` |
| `<float:value>` | Accepts floating-point numbers | `/price/<float:amount>` |
| `<path:subpath>` | Like string but also accepts slashes | `/files/<path:filepath>` |

---

## 6. Forms — Flask-WTF & WTForms

**Flask-WTF** wraps WTForms and adds CSRF protection automatically.

### Installation

```bash
pip install flask-wtf
```

### Defining a Form

```python
# app/forms.py
from flask_wtf import FlaskForm
from wtforms import StringField, IntegerField, PasswordField, SubmitField
from wtforms.validators import DataRequired, Email, Length, EqualTo

class CityForm(FlaskForm):
    city       = StringField('City',       validators=[DataRequired()])
    country    = StringField('Country',    validators=[DataRequired()])
    population = IntegerField('Population', validators=[DataRequired()])
    submit     = SubmitField('Save')

class RegisterForm(FlaskForm):
    email    = StringField('Email',    validators=[DataRequired(), Email()])
    password = PasswordField('Password', validators=[DataRequired(), Length(min=8)])
    confirm  = PasswordField('Confirm', validators=[EqualTo('password')])
    submit   = SubmitField('Register')
```

### Using the Form in a Route

```python
@app.route('/add', methods=['GET', 'POST'])
def add_city():
    form = CityForm()
    if form.validate_on_submit():          # True only on valid POST
        database.insert_city(
            form.city.data,
            form.population.data,
            form.country.data
        )
        flash('City added!', 'success')
        return redirect(url_for('cities'))
    return render_template('add_city.html', form=form)
```

### Rendering the Form in a Template

```html
<form method="POST">
  {{ form.csrf_token }}
  <div>
    {{ form.city.label }}
    {{ form.city(class="form-control") }}
    {% for error in form.city.errors %}
      <span class="error">{{ error }}</span>
    {% endfor %}
  </div>
  {{ form.submit(class="btn btn-primary") }}
</form>
```

### Common WTForms Validators

| Validator | Description |
| :--- | :--- |
| `DataRequired()` | Field must not be empty |
| `Email()` | Must be a valid email format |
| `Length(min, max)` | String length must be within bounds |
| `EqualTo('field')` | Must match the value of another field |
| `NumberRange(min, max)` | Numeric value must be within bounds |
| `Optional()` | Field is allowed to be empty |

---

## 7. Authentication — Flask-Login

**Flask-Login** manages user sessions (login, logout, "remember me", protected routes).

### Installation

```bash
pip install flask-login
```

### Setup in `__init__.py`

```python
from flask_login import LoginManager

login_manager = LoginManager()

def create_app():
    app = Flask(__name__)
    app.config['SECRET_KEY'] = os.getenv('SECRET_KEY')

    login_manager.init_app(app)
    login_manager.login_view = 'auth.login'  # redirect unauthenticated users here

    return app
```

### User Model

```python
# app/models.py
from flask_login import UserMixin

class User(UserMixin):
    """Simple user class backed by SQLite."""
    def __init__(self, id, email, password_hash):
        self.id            = id
        self.email         = email
        self.password_hash = password_hash

@login_manager.user_loader
def load_user(user_id):
    row = database.get_user_by_id(int(user_id))
    if row:
        return User(row['id'], row['email'], row['password_hash'])
    return None
```

### Auth Routes

```python
from flask_login import login_user, logout_user, login_required, current_user
from werkzeug.security import generate_password_hash, check_password_hash

@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        user = database.get_user_by_email(form.email.data)
        if user and check_password_hash(user['password_hash'], form.password.data):
            login_user(User(user['id'], user['email'], user['password_hash']))
            return redirect(url_for('dashboard'))
        flash('Invalid credentials.', 'error')
    return render_template('login.html', form=form)

@app.route('/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('home'))

@app.route('/dashboard')
@login_required            # Redirects to login_view if not authenticated
def dashboard():
    return render_template('dashboard.html', user=current_user)
```

| Function | Description |
| :--- | :--- |
| `login_user(user)` | Logs in the given `UserMixin` object and stores the session. |
| `logout_user()` | Clears the current user's session. |
| `login_required` | Decorator that blocks unauthenticated access to a route. |
| `current_user` | Proxy to the currently logged-in user (or `AnonymousUser`). |
| `generate_password_hash()` | Hashes a plain-text password using Werkzeug (bcrypt/pbkdf2). |
| `check_password_hash()` | Verifies a plain-text password against a stored hash. |

---

## 8. CRUD Operations with SQLite

Flask integrates with SQLite via Python's built-in `sqlite3` module.  
The standard pattern stores the connection on Flask's `g` object so it lives for one request.

### Database Helper (`database.py`)

```python
# app/database.py
import sqlite3
from flask import g
import os

DATABASE = os.path.join(os.path.dirname(os.path.abspath(__file__)), '..', 'app.db')

# ── Connection management ──────────────────────────────────────────────────────
def get_db():
    if '_database' not in g:
        g._database = sqlite3.connect(DATABASE)
        g._database.row_factory = sqlite3.Row   # rows act like dicts
    return g._database

def close_connection(exception=None):
    db = g.pop('_database', None)
    if db is not None:
        db.close()

def init_db():
    with sqlite3.connect(DATABASE) as conn:
        conn.execute('''
            CREATE TABLE IF NOT EXISTS cities (
                id         INTEGER PRIMARY KEY AUTOINCREMENT,
                city       TEXT    NOT NULL UNIQUE,
                country    TEXT    NOT NULL,
                population INTEGER NOT NULL
            )
        ''')
        conn.commit()

# ── Generic query helper ───────────────────────────────────────────────────────
def query_db(query, args=(), one=False):
    cur = get_db().execute(query, args)
    rv  = cur.fetchall()
    cur.close()
    return (rv[0] if rv else None) if one else rv
```

### CREATE — Insert a Row

```python
def insert_city(city, population, country):
    db = get_db()
    try:
        db.execute(
            'INSERT INTO cities (city, country, population) VALUES (?, ?, ?)',
            (city, country, population)
        )
        db.commit()
        return True
    except sqlite3.IntegrityError:
        return False   # duplicate entry
```

### READ — Query Rows

```python
def get_all_cities():
    return query_db('SELECT id, city, country, population FROM cities ORDER BY city')

def get_city_by_id(city_id):
    return query_db('SELECT * FROM cities WHERE id = ?', (city_id,), one=True)
```

### UPDATE — Modify a Row

```python
def update_city(city_id, city, population, country):
    db = get_db()
    db.execute(
        'UPDATE cities SET city = ?, country = ?, population = ? WHERE id = ?',
        (city, country, population, city_id)
    )
    db.commit()
```

### DELETE — Remove a Row

```python
def delete_city(city_id):
    db = get_db()
    db.execute('DELETE FROM cities WHERE id = ?', (city_id,))
    db.commit()
```

### CRUD in Routes

```python
@app.route('/cities')
def list_cities():
    cities = database.get_all_cities()
    return render_template('cities.html', cities=cities)

@app.route('/cities/add', methods=['GET', 'POST'])
def add_city():
    form = CityForm()
    if form.validate_on_submit():
        database.insert_city(form.city.data, form.population.data, form.country.data)
        flash('City added!', 'success')
        return redirect(url_for('list_cities'))
    return render_template('city_form.html', form=form)

@app.route('/cities/edit/<int:city_id>', methods=['GET', 'POST'])
def edit_city(city_id):
    city = database.get_city_by_id(city_id)
    if not city:
        abort(404)
    form = CityForm(data=dict(city))
    if form.validate_on_submit():
        database.update_city(city_id, form.city.data, form.population.data, form.country.data)
        flash('City updated!', 'success')
        return redirect(url_for('list_cities'))
    return render_template('city_form.html', form=form, editing=True)

@app.route('/cities/delete/<int:city_id>', methods=['POST'])
def delete_city(city_id):
    database.delete_city(city_id)
    flash('City deleted.', 'success')
    return redirect(url_for('list_cities'))
```

---

## Quick Reference — Putting It All Together

### Minimal Flask + SQLite Project Layout

```
my_app/
├── app.py               ← entry point (create_app + run)
├── .flaskenv            ← FLASK_APP, FLASK_ENV, SECRET_KEY
├── requirements.txt
└── app/
    ├── __init__.py      ← create_app() factory
    ├── routes.py        ← view functions / blueprints
    ├── forms.py         ← WTForms form classes
    ├── database.py      ← get_db, init_db, CRUD helpers
    ├── models.py        ← UserMixin + user_loader (Flask-Login)
    └── templates/
        ├── base.html    ← shared layout with flash messages
        ├── index.html
        ├── login.html
        └── city_form.html
```

### Typical Request Lifecycle

```
Browser → Flask router (@app.route)
        → Before-request hooks (@app.before_request)
        → View function
            → get_db()          (open SQLite connection on g)
            → query / mutate DB
            → render_template() (Jinja2 renders HTML)
        → After-request hooks (@app.after_request)
        → teardown_appcontext  (close_connection called)
        → Response sent to browser
```

### `requirements.txt` (typical)

```
flask
flask-wtf
flask-login
python-dotenv
werkzeug
requests
```

---

**Last Updated:** 2026-02-20  
**Flask Version:** 3.x compatible  
**Total Functions/Methods Covered:** 24
