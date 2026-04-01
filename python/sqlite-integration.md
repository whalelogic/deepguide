# SQLite Database Integration with Flask

Complete guide to using SQLite with Flask, including Python functions, SQL queries, and CLI commands.

## Table of Contents
- [Flask-SQLAlchemy Setup](#flask-sqlalchemy-setup)
- [Database Models](#database-models)
- [Python Database Functions](#python-database-functions)
- [Raw SQL Queries](#raw-sql-queries)
- [SQLite CLI Commands](#sqlite-cli-commands)
- [Migration Patterns](#migration-patterns)

---

## Flask-SQLAlchemy Setup

### Installation and Configuration

```bash
pip install flask-sqlalchemy
```

```python
# app.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///instance/app.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['SQLALCHEMY_ECHO'] = True  # Log SQL queries

db = SQLAlchemy(app)
```

---

## Database Models

### Basic Model Definition

```python
# models.py
from datetime import datetime
from app import db

class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False, index=True)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(256), nullable=False)
    is_active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    posts = db.relationship('Post', backref='author', lazy='dynamic', cascade='all, delete-orphan')
    
    def __repr__(self):
        return f'<User {self.username}>'
    
    def to_dict(self):
        return {
            'id': self.id,
            'username': self.username,
            'email': self.email,
            'is_active': self.is_active,
            'created_at': self.created_at.isoformat()
        }

class Post(db.Model):
    __tablename__ = 'posts'
    
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    slug = db.Column(db.String(200), unique=True, nullable=False)
    published = db.Column(db.Boolean, default=False)
    view_count = db.Column(db.Integer, default=0)
    published_at = db.Column(db.DateTime)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    # Foreign keys
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    
    # Relationships
    comments = db.relationship('Comment', backref='post', lazy='dynamic', cascade='all, delete-orphan')
    tags = db.relationship('Tag', secondary='post_tags', backref='posts')
    
    def __repr__(self):
        return f'<Post {self.title}>'

class Comment(db.Model):
    __tablename__ = 'comments'
    
    id = db.Column(db.Integer, primary_key=True)
    content = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    post_id = db.Column(db.Integer, db.ForeignKey('posts.id'), nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)

class Tag(db.Model):
    __tablename__ = 'tags'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), unique=True, nullable=False)

# Many-to-many association table
post_tags = db.Table('post_tags',
    db.Column('post_id', db.Integer, db.ForeignKey('posts.id'), primary_key=True),
    db.Column('tag_id', db.Integer, db.ForeignKey('tags.id'), primary_key=True),
    db.Column('created_at', db.DateTime, default=datetime.utcnow)
)
```

---

## Python Database Functions

### Create Operations

```python
def create_user(username, email, password):
    """Create a new user."""
    from werkzeug.security import generate_password_hash
    
    user = User(
        username=username,
        email=email,
        password_hash=generate_password_hash(password)
    )
    
    db.session.add(user)
    
    try:
        db.session.commit()
        return user
    except Exception as e:
        db.session.rollback()
        raise e

def create_post(user_id, title, content, tags=None):
    """Create a new blog post with optional tags."""
    from slugify import slugify
    
    post = Post(
        user_id=user_id,
        title=title,
        content=content,
        slug=slugify(title),
        published_at=datetime.utcnow()
    )
    
    # Add tags
    if tags:
        for tag_name in tags:
            tag = Tag.query.filter_by(name=tag_name).first()
            if not tag:
                tag = Tag(name=tag_name)
            post.tags.append(tag)
    
    db.session.add(post)
    db.session.commit()
    
    return post

def bulk_create_users(users_data):
    """Bulk create users for efficiency."""
    users = [User(**data) for data in users_data]
    db.session.bulk_save_objects(users)
    db.session.commit()
```

### Read Operations

```python
def get_user_by_id(user_id):
    """Get user by ID."""
    return User.query.get(user_id)
    # Or use get_or_404 in route handlers
    # return User.query.get_or_404(user_id)

def get_user_by_username(username):
    """Get user by username."""
    return User.query.filter_by(username=username).first()

def get_active_users():
    """Get all active users."""
    return User.query.filter_by(is_active=True).all()

def search_users(query):
    """Search users by username or email."""
    search = f"%{query}%"
    return User.query.filter(
        db.or_(
            User.username.like(search),
            User.email.like(search)
        )
    ).all()

def get_recent_posts(limit=10):
    """Get recent published posts."""
    return Post.query.filter_by(published=True)\
        .order_by(Post.published_at.desc())\
        .limit(limit)\
        .all()

def get_user_posts(user_id, published_only=True):
    """Get all posts by a user."""
    query = Post.query.filter_by(user_id=user_id)
    
    if published_only:
        query = query.filter_by(published=True)
    
    return query.order_by(Post.created_at.desc()).all()

def get_posts_with_tag(tag_name):
    """Get all posts with a specific tag."""
    return Post.query.join(Post.tags).filter(Tag.name == tag_name).all()

def get_paginated_posts(page=1, per_page=20):
    """Get paginated posts."""
    return Post.query.filter_by(published=True)\
        .order_by(Post.published_at.desc())\
        .paginate(page=page, per_page=per_page, error_out=False)

def get_post_with_comments(post_id):
    """Get post with all comments eagerly loaded."""
    return Post.query.options(
        db.joinedload(Post.comments).joinedload(Comment.user)
    ).get(post_id)
```

### Update Operations

```python
def update_user_email(user_id, new_email):
    """Update user's email address."""
    user = User.query.get(user_id)
    
    if not user:
        return None
    
    user.email = new_email
    user.updated_at = datetime.utcnow()
    
    try:
        db.session.commit()
        return user
    except Exception as e:
        db.session.rollback()
        raise e

def publish_post(post_id):
    """Publish a post."""
    post = Post.query.get(post_id)
    
    if post and not post.published:
        post.published = True
        post.published_at = datetime.utcnow()
        db.session.commit()
    
    return post

def increment_view_count(post_id):
    """Increment post view count."""
    post = Post.query.get(post_id)
    if post:
        post.view_count += 1
        db.session.commit()

def bulk_update_users(user_ids, **updates):
    """Bulk update multiple users."""
    User.query.filter(User.id.in_(user_ids)).update(
        updates,
        synchronize_session=False
    )
    db.session.commit()
```

### Delete Operations

```python
def delete_user(user_id):
    """Delete a user and all associated data."""
    user = User.query.get(user_id)
    
    if not user:
        return False
    
    try:
        db.session.delete(user)
        db.session.commit()
        return True
    except Exception as e:
        db.session.rollback()
        raise e

def delete_old_posts(days=30):
    """Delete unpublished posts older than X days."""
    from datetime import timedelta
    
    cutoff_date = datetime.utcnow() - timedelta(days=days)
    
    deleted = Post.query.filter(
        Post.published == False,
        Post.created_at < cutoff_date
    ).delete()
    
    db.session.commit()
    return deleted

def soft_delete_user(user_id):
    """Soft delete - mark as inactive instead of deleting."""
    user = User.query.get(user_id)
    if user:
        user.is_active = False
        user.deleted_at = datetime.utcnow()
        db.session.commit()
    return user
```

### Complex Queries

```python
def get_user_statistics(user_id):
    """Get comprehensive user statistics."""
    from sqlalchemy import func
    
    user = User.query.get(user_id)
    
    stats = {
        'total_posts': user.posts.count(),
        'published_posts': user.posts.filter_by(published=True).count(),
        'total_views': db.session.query(func.sum(Post.view_count))\
            .filter_by(user_id=user_id).scalar() or 0,
        'total_comments': Comment.query.filter_by(user_id=user_id).count(),
        'most_viewed_post': user.posts.order_by(Post.view_count.desc()).first()
    }
    
    return stats

def get_popular_tags(limit=10):
    """Get most popular tags."""
    from sqlalchemy import func
    
    return db.session.query(
        Tag.name,
        func.count(post_tags.c.post_id).label('post_count')
    ).join(post_tags).group_by(Tag.id)\
     .order_by(db.desc('post_count'))\
     .limit(limit).all()

def search_posts_fulltext(search_term):
    """Full-text search in posts."""
    search = f"%{search_term}%"
    return Post.query.filter(
        db.or_(
            Post.title.ilike(search),
            Post.content.ilike(search)
        ),
        Post.published == True
    ).all()

def get_trending_posts(days=7, limit=10):
    """Get trending posts based on recent activity."""
    from datetime import timedelta
    from sqlalchemy import func
    
    cutoff_date = datetime.utcnow() - timedelta(days=days)
    
    return db.session.query(Post)\
        .outerjoin(Comment)\
        .filter(Post.published_at >= cutoff_date)\
        .group_by(Post.id)\
        .order_by(
            (func.count(Comment.id) + Post.view_count).desc()
        )\
        .limit(limit).all()
```

---

## Raw SQL Queries

### Executing Raw SQL

```python
def execute_raw_query(query, params=None):
    """Execute raw SQL query safely."""
    from sqlalchemy import text
    
    result = db.session.execute(text(query), params or {})
    db.session.commit()
    return result

def get_users_raw():
    """Get users using raw SQL."""
    result = db.session.execute(text('SELECT * FROM users WHERE is_active = :active'),
                                {'active': True})
    return result.fetchall()

def complex_report():
    """Generate complex report using raw SQL."""
    query = text("""
        SELECT 
            u.username,
            COUNT(DISTINCT p.id) as post_count,
            COUNT(DISTINCT c.id) as comment_count,
            SUM(p.view_count) as total_views
        FROM users u
        LEFT JOIN posts p ON u.id = p.user_id
        LEFT JOIN comments c ON u.id = c.user_id
        WHERE u.is_active = 1
        GROUP BY u.id, u.username
        HAVING post_count > 0
        ORDER BY total_views DESC
        LIMIT 10
    """)
    
    result = db.session.execute(query)
    return [dict(row) for row in result]

def update_with_raw_sql(user_id, new_username):
    """Update using raw SQL with parameters."""
    query = text("UPDATE users SET username = :username WHERE id = :id")
    db.session.execute(query, {'username': new_username, 'id': user_id})
    db.session.commit()
```

---

## SQLite CLI Commands

### Basic SQLite Commands

```bash
# Open database
sqlite3 instance/app.db

# Common commands (inside sqlite3 shell)
.help                    # Show help
.databases              # List databases
.tables                 # List all tables
.schema users           # Show table schema
.schema                 # Show all schemas
.indexes users          # Show indexes for table
.mode column            # Set output mode to columns
.headers on             # Show column headers
.quit                   # Exit
```

### Table Operations

```sql
-- Show table structure
.schema users

-- Describe table
PRAGMA table_info(users);

-- Show all indexes
.indexes

-- Show foreign keys
PRAGMA foreign_key_list(posts);

-- Check database integrity
PRAGMA integrity_check;

-- Show database stats
.dbconfig
```

### Query Commands

```bash
# Execute SQL from command line
sqlite3 instance/app.db "SELECT * FROM users LIMIT 5;"

# Execute from file
sqlite3 instance/app.db < queries.sql

# Export data
sqlite3 instance/app.db -csv "SELECT * FROM users;" > users.csv
sqlite3 instance/app.db -json "SELECT * FROM posts;"

# Import CSV
sqlite3 instance/app.db
.mode csv
.import data.csv users
```

### Common SQL Queries Reference

| Operation | Command |
|-----------|---------|
| **Select all** | `SELECT * FROM users;` |
| **Select specific columns** | `SELECT username, email FROM users;` |
| **Where clause** | `SELECT * FROM users WHERE is_active = 1;` |
| **Order by** | `SELECT * FROM posts ORDER BY created_at DESC;` |
| **Limit** | `SELECT * FROM users LIMIT 10;` |
| **Count** | `SELECT COUNT(*) FROM users;` |
| **Distinct** | `SELECT DISTINCT role FROM users;` |
| **Like pattern** | `SELECT * FROM users WHERE username LIKE 'john%';` |
| **Join** | `SELECT u.username, p.title FROM users u JOIN posts p ON u.id = p.user_id;` |
| **Group by** | `SELECT user_id, COUNT(*) FROM posts GROUP BY user_id;` |
| **Insert** | `INSERT INTO users (username, email) VALUES ('john', 'john@example.com');` |
| **Update** | `UPDATE users SET is_active = 1 WHERE id = 5;` |
| **Delete** | `DELETE FROM users WHERE id = 5;` |

---

## Migration Patterns

### Initialize Database

```python
# init_db.py
from app import app, db
from models import User, Post, Comment, Tag

def init_database():
    """Initialize database with tables."""
    with app.app_context():
        # Drop all tables (careful in production!)
        db.drop_all()
        
        # Create all tables
        db.create_all()
        
        print("Database initialized successfully!")

def seed_database():
    """Seed database with sample data."""
    with app.app_context():
        # Create admin user
        admin = User(
            username='admin',
            email='admin@example.com',
            password_hash=generate_password_hash('admin123'),
            is_active=True
        )
        db.session.add(admin)
        
        # Create sample tags
        tags = [
            Tag(name='Python'),
            Tag(name='Flask'),
            Tag(name='Tutorial'),
            Tag(name='Guide')
        ]
        db.session.add_all(tags)
        
        db.session.commit()
        
        # Create sample posts
        post = Post(
            user_id=admin.id,
            title='Getting Started with Flask',
            content='This is a comprehensive guide...',
            slug='getting-started-with-flask',
            published=True,
            published_at=datetime.utcnow()
        )
        post.tags.append(tags[0])
        post.tags.append(tags[1])
        
        db.session.add(post)
        db.session.commit()
        
        print("Database seeded successfully!")

if __name__ == '__main__':
    init_database()
    seed_database()
```

### Database Backup and Restore

```python
def backup_database(backup_path='backup.db'):
    """Backup SQLite database."""
    import shutil
    from pathlib import Path
    
    db_path = 'instance/app.db'
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    backup_file = f'backups/backup_{timestamp}.db'
    
    Path('backups').mkdir(exist_ok=True)
    shutil.copy2(db_path, backup_file)
    
    return backup_file

def restore_database(backup_path):
    """Restore database from backup."""
    import shutil
    
    db_path = 'instance/app.db'
    shutil.copy2(backup_path, db_path)
    
    return True
```

---

## Advanced Query Patterns

### Filtering and Chaining

```python
def advanced_user_search(username=None, email=None, is_active=None, 
                        created_after=None, sort_by='created_at'):
    """Advanced user search with multiple filters."""
    query = User.query
    
    if username:
        query = query.filter(User.username.ilike(f'%{username}%'))
    
    if email:
        query = query.filter(User.email.ilike(f'%{email}%'))
    
    if is_active is not None:
        query = query.filter_by(is_active=is_active)
    
    if created_after:
        query = query.filter(User.created_at >= created_after)
    
    # Dynamic sorting
    if sort_by == 'username':
        query = query.order_by(User.username)
    elif sort_by == 'email':
        query = query.order_by(User.email)
    else:
        query = query.order_by(User.created_at.desc())
    
    return query.all()
```

### Aggregation Functions

```python
from sqlalchemy import func

def get_post_statistics():
    """Get aggregated post statistics."""
    stats = db.session.query(
        func.count(Post.id).label('total_posts'),
        func.sum(Post.view_count).label('total_views'),
        func.avg(Post.view_count).label('avg_views'),
        func.max(Post.view_count).label('max_views')
    ).filter_by(published=True).first()
    
    return {
        'total_posts': stats.total_posts or 0,
        'total_views': stats.total_views or 0,
        'avg_views': round(stats.avg_views or 0, 2),
        'max_views': stats.max_views or 0
    }

def get_posts_by_month():
    """Group posts by month."""
    results = db.session.query(
        func.strftime('%Y-%m', Post.created_at).label('month'),
        func.count(Post.id).label('count')
    ).group_by('month')\
     .order_by('month').all()
    
    return [{'month': r.month, 'count': r.count} for r in results]

def get_top_authors(limit=10):
    """Get authors with most posts."""
    return db.session.query(
        User.username,
        func.count(Post.id).label('post_count'),
        func.sum(Post.view_count).label('total_views')
    ).join(Post)\
     .group_by(User.id)\
     .order_by(db.desc('post_count'))\
     .limit(limit).all()
```

### Relationships and Joins

```python
def get_post_details(post_id):
    """Get post with author and comment count."""
    from sqlalchemy.orm import joinedload
    
    post = Post.query.options(
        joinedload(Post.author),
        joinedload(Post.tags)
    ).get(post_id)
    
    if post:
        comment_count = post.comments.count()
        
        return {
            'post': post,
            'author': post.author,
            'tags': post.tags,
            'comment_count': comment_count
        }
    
    return None

def get_users_with_post_count():
    """Get all users with their post counts."""
    return db.session.query(
        User,
        func.count(Post.id).label('post_count')
    ).outerjoin(Post)\
     .group_by(User.id)\
     .all()
```

---

## Transaction Management

```python
def transfer_post_ownership(post_id, new_user_id):
    """Transfer post to another user with transaction."""
    try:
        post = Post.query.get(post_id)
        new_user = User.query.get(new_user_id)
        
        if not post or not new_user:
            raise ValueError("Post or user not found")
        
        old_user_id = post.user_id
        post.user_id = new_user_id
        
        # Log the transfer
        log = ActivityLog(
            action='post_transfer',
            details=f'Post {post_id} transferred from {old_user_id} to {new_user_id}'
        )
        db.session.add(log)
        
        db.session.commit()
        return True
        
    except Exception as e:
        db.session.rollback()
        print(f"Error: {e}")
        return False

def batch_operation_with_transaction():
    """Perform multiple operations in a single transaction."""
    try:
        # Multiple operations
        user1 = User(username='user1', email='user1@example.com')
        user2 = User(username='user2', email='user2@example.com')
        
        db.session.add_all([user1, user2])
        db.session.flush()  # Get IDs without committing
        
        post = Post(user_id=user1.id, title='First Post', content='...')
        db.session.add(post)
        
        # All or nothing
        db.session.commit()
        return True
        
    except Exception as e:
        db.session.rollback()
        print(f"Transaction failed: {e}")
        return False
```

---

## Database Utilities

### Connection and Session Management

```python
def get_db_stats():
    """Get database statistics."""
    from sqlalchemy import inspect
    
    inspector = inspect(db.engine)
    
    stats = {
        'tables': inspector.get_table_names(),
        'total_tables': len(inspector.get_table_names())
    }
    
    for table in stats['tables']:
        count = db.session.execute(
            text(f'SELECT COUNT(*) FROM {table}')
        ).scalar()
        stats[f'{table}_count'] = count
    
    return stats

def vacuum_database():
    """Optimize database (VACUUM)."""
    db.session.execute(text('VACUUM'))
    db.session.commit()

def analyze_database():
    """Update query planner statistics."""
    db.session.execute(text('ANALYZE'))
    db.session.commit()
```

### Custom Query Helper

```python
class DatabaseHelper:
    """Helper class for common database operations."""
    
    @staticmethod
    def exists(model, **filters):
        """Check if record exists."""
        return db.session.query(
            model.query.filter_by(**filters).exists()
        ).scalar()
    
    @staticmethod
    def get_or_create(model, defaults=None, **filters):
        """Get existing record or create new one."""
        instance = model.query.filter_by(**filters).first()
        
        if instance:
            return instance, False
        else:
            params = dict(filters)
            params.update(defaults or {})
            instance = model(**params)
            db.session.add(instance)
            db.session.commit()
            return instance, True
    
    @staticmethod
    def bulk_get_or_create(model, items, key_field):
        """Bulk get or create records."""
        existing_keys = {getattr(obj, key_field): obj 
                        for obj in model.query.all()}
        
        new_items = []
        for item_data in items:
            key = item_data.get(key_field)
            if key not in existing_keys:
                new_items.append(model(**item_data))
        
        if new_items:
            db.session.bulk_save_objects(new_items)
            db.session.commit()
        
        return len(new_items)

# Usage
user, created = DatabaseHelper.get_or_create(
    User,
    defaults={'email': 'new@example.com'},
    username='newuser'
)
```

---

## SQLite-Specific Features

### JSON Support (SQLite 3.38+)

```python
# Store JSON data
class Settings(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'))
    preferences = db.Column(db.JSON)

# Usage
settings = Settings(
    user_id=1,
    preferences={
        'theme': 'dark',
        'notifications': True,
        'language': 'en'
    }
)
db.session.add(settings)
db.session.commit()

# Query JSON
def get_users_with_dark_theme():
    """Query JSON field."""
    return Settings.query.filter(
        Settings.preferences['theme'].astext == 'dark'
    ).all()
```

### Full-Text Search

```sql
-- Create FTS5 table in SQLite
CREATE VIRTUAL TABLE posts_fts USING fts5(title, content, content=posts, content_rowid=id);

-- Populate FTS table
INSERT INTO posts_fts(rowid, title, content) SELECT id, title, content FROM posts;

-- Search
SELECT * FROM posts_fts WHERE posts_fts MATCH 'flask tutorial';
```

```python
def fulltext_search(query):
    """Full-text search using FTS5."""
    sql = text("""
        SELECT p.* FROM posts p
        JOIN posts_fts fts ON p.id = fts.rowid
        WHERE posts_fts MATCH :query
        ORDER BY rank
    """)
    
    result = db.session.execute(sql, {'query': query})
    return result.fetchall()
```

---

## Performance Optimization

### Indexing

```python
# Add indexes in models
class User(db.Model):
    email = db.Column(db.String(120), unique=True, nullable=False, index=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow, index=True)

# Composite index
class Post(db.Model):
    __table_args__ = (
        db.Index('idx_user_published', 'user_id', 'published'),
    )
```

### Eager Loading

```python
def get_posts_optimized(page=1):
    """Get posts with eager loading to avoid N+1 queries."""
    from sqlalchemy.orm import joinedload, subqueryload
    
    return Post.query.options(
        joinedload(Post.author),           # Use JOIN
        subqueryload(Post.comments),       # Use separate query
        joinedload(Post.tags)
    ).paginate(page=page, per_page=20)
```

### Query Optimization

```python
# Bad - N+1 query problem
posts = Post.query.all()
for post in posts:
    print(post.author.username)  # Each iteration queries database

# Good - Eager load relationships
posts = Post.query.options(joinedload(Post.author)).all()
for post in posts:
    print(post.author.username)  # No additional queries
```

---

## Common Patterns Table

| Task | SQLAlchemy | Raw SQL |
|------|-----------|---------|
| **Get by ID** | `User.query.get(5)` | `SELECT * FROM users WHERE id = 5` |
| **Get by filter** | `User.query.filter_by(username='john').first()` | `SELECT * FROM users WHERE username = 'john' LIMIT 1` |
| **Get all** | `User.query.all()` | `SELECT * FROM users` |
| **Count** | `User.query.count()` | `SELECT COUNT(*) FROM users` |
| **Exists** | `db.session.query(User.query.filter_by(email='x').exists()).scalar()` | `SELECT EXISTS(SELECT 1 FROM users WHERE email = 'x')` |
| **Order** | `User.query.order_by(User.created_at.desc())` | `SELECT * FROM users ORDER BY created_at DESC` |
| **Filter multiple** | `User.query.filter(User.age > 18, User.is_active == True)` | `SELECT * FROM users WHERE age > 18 AND is_active = 1` |
| **OR condition** | `User.query.filter(db.or_(User.role == 'admin', User.role == 'mod'))` | `SELECT * FROM users WHERE role = 'admin' OR role = 'mod'` |
| **IN clause** | `User.query.filter(User.id.in_([1,2,3]))` | `SELECT * FROM users WHERE id IN (1, 2, 3)` |
| **LIKE** | `User.query.filter(User.username.like('%john%'))` | `SELECT * FROM users WHERE username LIKE '%john%'` |

---

## Best Practices

1. **Use SQLAlchemy ORM** - Safer and more maintainable than raw SQL
2. **Always use parameterized queries** - Prevent SQL injection
3. **Handle transactions properly** - Use try/except with rollback
4. **Index frequently queried columns** - Improves performance
5. **Use eager loading** - Avoid N+1 query problems
6. **Implement soft deletes** - Mark as inactive instead of deleting
7. **Back up regularly** - Automate database backups
8. **Use migrations** - Track schema changes with Flask-Migrate
9. **Close sessions properly** - SQLAlchemy handles this automatically
10. **Test queries** - Use EXPLAIN QUERY PLAN in SQLite

---

## Related Documentation

- [Flask-SQLAlchemy Documentation](https://flask-sqlalchemy.palletsprojects.com/)
- [SQLAlchemy ORM Tutorial](https://docs.sqlalchemy.org/en/latest/orm/tutorial.html)
- [SQLite Documentation](https://www.sqlite.org/docs.html)
