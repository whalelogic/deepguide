# Flask Quick Reference Cheat Sheet

Essential commands and patterns for Flask development.

## Setup & Run

```bash
# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install Flask
pip install flask flask-sqlalchemy flask-login

# Set Flask app
export FLASK_APP=app.py
export FLASK_ENV=development

# Run application
flask run
flask run --debug --host=0.0.0.0 --port=5000
```

## Common Flask CLI Commands

| Command | Description |
|---------|-------------|
| `flask run` | Start development server |
| `flask shell` | Open Python shell with app context |
| `flask routes` | List all routes |
| `flask db init` | Initialize migrations |
| `flask db migrate -m "msg"` | Create migration |
| `flask db upgrade` | Apply migrations |

## Basic Routes

```python
# GET route
@app.route('/')
def index():
    return render_template('index.html')

# POST route
@app.route('/submit', methods=['POST'])
def submit():
    data = request.get_json()
    return jsonify({'status': 'success'})

# Dynamic route
@app.route('/user/<username>')
def user_profile(username):
    return f'Profile: {username}'

# Multiple methods
@app.route('/data', methods=['GET', 'POST', 'DELETE'])
def data():
    if request.method == 'POST':
        return 'Creating data'
    return 'Getting data'
```

## Jinja2 Quick Syntax

```jinja2
{# Variables #}
{{ variable }}
{{ user.name }}
{{ items[0] }}

{# Control structures #}
{% if condition %}
{% elif other %}
{% else %}
{% endif %}

{% for item in items %}
    {{ loop.index }}. {{ item }}
{% endfor %}

{# Filters #}
{{ name | upper }}
{{ price | round(2) }}
{{ text | truncate(100) }}
{{ list | length }}
{{ value | default('N/A') }}

{# Template inheritance #}
{% extends "base.html" %}
{% block content %}{% endblock %}

{# Include #}
{% include 'header.html' %}
```

## Database Quick Reference

```python
# Create
user = User(username='john', email='john@example.com')
db.session.add(user)
db.session.commit()

# Read
User.query.all()
User.query.get(1)
User.query.filter_by(username='john').first()
User.query.filter(User.age > 18).all()

# Update
user = User.query.get(1)
user.email = 'newemail@example.com'
db.session.commit()

# Delete
db.session.delete(user)
db.session.commit()

# Complex queries
User.query.order_by(User.created_at.desc()).limit(10).all()
User.query.filter(User.email.like('%@gmail.com')).count()
```

## SQLite CLI Commands

```bash
sqlite3 database.db                    # Open database
.tables                                # List tables
.schema users                          # Show table schema
SELECT * FROM users;                   # Query data
.quit                                  # Exit
```

## Redis Caching

```python
from flask_caching import Cache

cache = Cache(config={'CACHE_TYPE': 'redis'})
cache.init_app(app)

# Cache route
@app.route('/data')
@cache.cached(timeout=300)
def get_data():
    return jsonify(expensive_operation())

# Manual cache
cache.set('key', 'value', timeout=300)
value = cache.get('key')
cache.delete('key')
```

## Authentication Quick Start

```python
from flask_login import LoginManager, login_user, logout_user, login_required

login_manager = LoginManager(app)
login_manager.login_view = 'login'

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

@app.route('/login', methods=['POST'])
def login():
    user = User.query.filter_by(username=username).first()
    if user and user.check_password(password):
        login_user(user)
        return redirect(url_for('index'))

@app.route('/protected')
@login_required
def protected():
    return 'Protected content'
```

## API Endpoints

```python
# JSON response
@app.route('/api/users')
def api_users():
    users = User.query.all()
    return jsonify([u.to_dict() for u in users])

# Handle JSON request
@app.route('/api/users', methods=['POST'])
def create_user_api():
    data = request.get_json()
    user = User(**data)
    db.session.add(user)
    db.session.commit()
    return jsonify(user.to_dict()), 201

# Error response
@app.errorhandler(404)
def not_found(error):
    return jsonify({'error': 'Not found'}), 404
```

## Environment Variables

```bash
# .env file
FLASK_APP=app.py
FLASK_ENV=development
SECRET_KEY=your-secret-key
DATABASE_URL=sqlite:///app.db
REDIS_URL=redis://localhost:6379/0

# Load in Python
from dotenv import load_dotenv
load_dotenv()
```

## Common Patterns

```python
# Blueprints
from flask import Blueprint

auth_bp = Blueprint('auth', __name__, url_prefix='/auth')

@auth_bp.route('/login')
def login():
    return render_template('login.html')

app.register_blueprint(auth_bp)

# Request hooks
@app.before_request
def before_request():
    g.user = current_user

@app.after_request
def after_request(response):
    response.headers['X-Custom-Header'] = 'value'
    return response

# Flash messages
flash('Message', 'category')
# In template: get_flashed_messages()

# URL generation
url_for('index')
url_for('user_profile', username='john')
url_for('static', filename='style.css')
```

## GCloud Commands

```bash
# SSH into VM
gcloud compute ssh INSTANCE_NAME --zone=ZONE

# Copy files to VM
gcloud compute scp file.py INSTANCE:~/

# Start/stop instance
gcloud compute instances start INSTANCE
gcloud compute instances stop INSTANCE

# View logs
gcloud compute ssh INSTANCE --command="tail -f /var/log/app.log"
```

## Testing

```python
import unittest

class TestCase(unittest.TestCase):
    def setUp(self):
        self.app = app.test_client()
        app.config['TESTING'] = True
    
    def test_index(self):
        response = self.app.get('/')
        self.assertEqual(response.status_code, 200)
    
    def test_login(self):
        response = self.app.post('/login', data={
            'username': 'test',
            'password': 'test123'
        })
        self.assertEqual(response.status_code, 200)
```

## Production Deployment

```bash
# Use Gunicorn
pip install gunicorn
gunicorn -w 4 -b 0.0.0.0:8000 app:app

# With Nginx reverse proxy
# nginx.conf location block:
location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}

# Systemd service
sudo systemctl start flask-app
sudo systemctl enable flask-app
sudo systemctl status flask-app
```

## Debugging

```python
# Enable debug mode
app.run(debug=True)

# Print to console
print(f"Value: {variable}")
app.logger.info('Info message')
app.logger.error('Error message')

# Interactive debugger
import pdb; pdb.set_trace()

# Flask shell
flask shell
>>> User.query.all()
>>> db.session.execute('SELECT 1')
```

## Common HTTP Status Codes

| Code | Meaning | Use Case |
|------|---------|----------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Not authorized |
| 404 | Not Found | Resource doesn't exist |
| 500 | Internal Server Error | Server error |

## Security Essentials

```python
# Password hashing
from werkzeug.security import generate_password_hash, check_password_hash

hash = generate_password_hash('password')
check_password_hash(hash, 'password')  # True

# CSRF protection
from flask_wtf.csrf import CSRFProtect
csrf = CSRFProtect(app)

# CORS
from flask_cors import CORS
CORS(app)

# Secure sessions
app.config['SESSION_COOKIE_SECURE'] = True
app.config['SESSION_COOKIE_HTTPONLY'] = True
```

---

For detailed guides, see the comprehensive documentation in `/documents/`
