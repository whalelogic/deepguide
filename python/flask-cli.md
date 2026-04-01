# Flask CLI Commands

The Flask Command Line Interface provides powerful tools for development, database management, and custom commands.

## Table of Contents
- [Common Flask CLI Commands](#common-flask-cli-commands)
- [Custom Commands](#custom-commands)
- [Database Commands](#database-commands)
- [Shell Context](#shell-context)

---

## Common Flask CLI Commands

| Command | Description | Example |
|---------|-------------|---------|
| `flask run` | Start development server | `flask run --host=0.0.0.0 --port=5000` |
| `flask run --debug` | Run with debug mode enabled | `flask run --debug` |
| `flask shell` | Start Python shell with app context | `flask shell` |
| `flask routes` | Display all registered routes | `flask routes` |
| `flask db init` | Initialize migration repository | `flask db init` |
| `flask db migrate` | Generate migration script | `flask db migrate -m "Add user table"` |
| `flask db upgrade` | Apply migrations to database | `flask db upgrade` |
| `flask db downgrade` | Revert last migration | `flask db downgrade` |
| `flask db current` | Show current migration | `flask db current` |
| `flask db history` | Show migration history | `flask db history` |
| `flask init-db` | Custom: Initialize database | `flask init-db` |
| `flask seed-db` | Custom: Seed database with data | `flask seed-db` |

---

## Environment Setup

```bash
# Set Flask app entry point
export FLASK_APP=app.py
export FLASK_ENV=development  # or 'production'
export FLASK_DEBUG=1

# Alternative: use .flaskenv file
# Flask automatically loads .flaskenv
echo "FLASK_APP=app.py" > .flaskenv
echo "FLASK_ENV=development" >> .flaskenv
```

---

## Custom Commands

### Creating Custom CLI Commands

```python
# app.py or cli.py
import click
from flask import Flask
from flask.cli import with_appcontext

app = Flask(__name__)

@app.cli.command('init-db')
@with_appcontext
def init_db_command():
    """Initialize the database."""
    from models import db
    db.create_all()
    click.echo('Initialized the database.')

@app.cli.command('seed-db')
@click.option('--count', default=10, help='Number of records to create')
@with_appcontext
def seed_db_command(count):
    """Seed the database with sample data."""
    from models import db, User
    
    for i in range(count):
        user = User(
            username=f'user{i}',
            email=f'user{i}@example.com'
        )
        db.session.add(user)
    
    db.session.commit()
    click.echo(f'Added {count} users to the database.')

@app.cli.command('create-admin')
@click.argument('username')
@click.argument('email')
@click.password_option()
@with_appcontext
def create_admin(username, email, password):
    """Create an admin user."""
    from models import db, User
    from werkzeug.security import generate_password_hash
    
    user = User(
        username=username,
        email=email,
        password_hash=generate_password_hash(password),
        is_admin=True
    )
    db.session.add(user)
    db.session.commit()
    click.echo(f'Admin user {username} created successfully.')

# Command groups for organization
@app.cli.group()
def user():
    """User management commands."""
    pass

@user.command('list')
@with_appcontext
def list_users():
    """List all users."""
    from models import User
    users = User.query.all()
    for u in users:
        click.echo(f'{u.id}: {u.username} ({u.email})')

@user.command('delete')
@click.argument('user_id', type=int)
@with_appcontext
def delete_user(user_id):
    """Delete a user by ID."""
    from models import db, User
    user = User.query.get(user_id)
    if user:
        db.session.delete(user)
        db.session.commit()
        click.echo(f'User {user.username} deleted.')
    else:
        click.echo(f'User {user_id} not found.')
```

### Usage Examples

```bash
# Initialize database
flask init-db

# Seed with 50 records
flask seed-db --count 50

# Create admin user (will prompt for password)
flask create-admin john john@example.com

# User management commands
flask user list
flask user delete 5
```

---

## Database Commands (Flask-Migrate)

### Setup Flask-Migrate

```python
# app.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///app.db'

db = SQLAlchemy(app)
migrate = Migrate(app, db)
```

### Migration Workflow

```bash
# 1. Initialize migrations (only once)
flask db init

# 2. Create initial migration
flask db migrate -m "Initial migration"

# 3. Apply migration to database
flask db upgrade

# 4. Make changes to models.py, then create migration
flask db migrate -m "Add email column to users"

# 5. Apply new migration
flask db upgrade

# 6. Rollback if needed
flask db downgrade

# 7. View migration history
flask db history

# 8. Show current revision
flask db current
```

---

## Shell Context

### Configuring Shell Context

```python
# app.py
from flask import Flask
from models import db, User, Post, Comment

app = Flask(__name__)

@app.shell_context_processor
def make_shell_context():
    """Add database instance and models to shell context."""
    return {
        'db': db,
        'User': User,
        'Post': Post,
        'Comment': Comment,
        'datetime': datetime
    }
```

### Using Flask Shell

```bash
$ flask shell
>>> # All models are automatically imported
>>> User.query.all()
[<User 'john'>, <User 'jane'>]

>>> # Database operations
>>> user = User(username='alice', email='alice@example.com')
>>> db.session.add(user)
>>> db.session.commit()

>>> # Query examples
>>> User.query.filter_by(username='alice').first()
<User 'alice'>

>>> User.query.count()
3
```

---

## Running the Application

### Development Server

```bash
# Basic run
flask run

# Custom host and port
flask run --host=0.0.0.0 --port=8080

# Enable debug mode
flask run --debug

# Disable reloader
flask run --no-reload

# With extra files to watch
flask run --extra-files file1.yaml:file2.json
```

### Production Deployment

```bash
# Don't use flask run in production!
# Use a production WSGI server instead:

# Gunicorn
pip install gunicorn
gunicorn -w 4 -b 0.0.0.0:8000 app:app

# uWSGI
pip install uwsgi
uwsgi --http :8000 --wsgi-file app.py --callable app --processes 4
```

---

## Environment Variables

### Setting Variables

```bash
# Linux/Mac
export FLASK_APP=app.py
export FLASK_ENV=development
export FLASK_DEBUG=1
export DATABASE_URL=sqlite:///instance/app.db
export SECRET_KEY=your-secret-key

# Windows CMD
set FLASK_APP=app.py
set FLASK_ENV=development

# Windows PowerShell
$env:FLASK_APP="app.py"
$env:FLASK_ENV="development"
```

### Using .flaskenv File

```bash
# .flaskenv (automatically loaded by Flask)
FLASK_APP=app.py
FLASK_ENV=development
FLASK_RUN_HOST=0.0.0.0
FLASK_RUN_PORT=5000

# Install python-dotenv
pip install python-dotenv
```

---

## Advanced CLI Features

### Click Options and Arguments

```python
import click
from flask.cli import with_appcontext

@app.cli.command('export-data')
@click.option('--format', type=click.Choice(['json', 'csv', 'xml']), default='json')
@click.option('--output', type=click.File('w'), default='-')
@click.argument('table_name')
@with_appcontext
def export_data(format, output, table_name):
    """Export table data in specified format."""
    import json
    from models import db
    
    # Dynamic table lookup
    model = db.Model._decl_class_registry.get(table_name)
    if not model:
        click.echo(f'Table {table_name} not found', err=True)
        return
    
    data = [item.to_dict() for item in model.query.all()]
    
    if format == 'json':
        json.dump(data, output, indent=2)
    elif format == 'csv':
        # CSV export logic
        pass
    
    click.echo(f'Exported {len(data)} records')
```

### Progress Bars

```python
@app.cli.command('process-data')
@with_appcontext
def process_data():
    """Process data with progress bar."""
    from models import User
    
    users = User.query.all()
    
    with click.progressbar(users, label='Processing users') as bar:
        for user in bar:
            # Process user
            user.process()
            db.session.commit()
```

### Colored Output

```python
@app.cli.command('check-health')
@with_appcontext
def check_health():
    """Check application health."""
    try:
        db.session.execute('SELECT 1')
        click.secho('✓ Database: OK', fg='green')
    except:
        click.secho('✗ Database: FAILED', fg='red')
    
    try:
        # Check Redis
        from extensions import redis_client
        redis_client.ping()
        click.secho('✓ Redis: OK', fg='green')
    except:
        click.secho('✗ Redis: FAILED', fg='red')
```

---

## Debugging Commands

```bash
# Show all routes with methods
flask routes

# Show routes for specific blueprint
flask routes | grep api

# Get configuration
flask shell
>>> app.config

# Test database connection
flask shell
>>> db.engine.execute('SELECT 1').scalar()

# Check installed extensions
flask shell
>>> app.extensions
```

---

## Tips & Best Practices

1. **Use .flaskenv for development settings** - Keep environment-specific configs out of code
2. **Create custom commands for common tasks** - Database seeding, cache clearing, backups
3. **Use command groups** - Organize related commands together
4. **Add helpful descriptions** - Make your CLI self-documenting
5. **Leverage Click features** - Options, arguments, prompts, confirmations
6. **Use @with_appcontext** - Ensures database and app context is available
7. **Never use `flask run` in production** - Use proper WSGI servers

---

## Related Documentation

- [Flask Official CLI Documentation](https://flask.palletsprojects.com/en/latest/cli/)
- [Click Documentation](https://click.palletsprojects.com/)
- [Flask-Migrate Documentation](https://flask-migrate.readthedocs.io/)
