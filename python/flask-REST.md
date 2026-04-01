# RESTful API Development with Flask

Complete guide to building RESTful APIs with Flask, including best practices and examples.

## Table of Contents
- [API Basics](#api-basics)
- [RESTful Routes](#restful-routes)
- [Request Handling](#request-handling)
- [Response Formatting](#response-formatting)
- [Error Handling](#error-handling)
- [Authentication](#authentication)
- [Pagination](#pagination)
- [Rate Limiting](#rate-limiting)
- [API Documentation](#api-documentation)

---

## API Basics

### Minimal API Setup

```python
from flask import Flask, jsonify, request
from flask_cors import CORS

app = Flask(__name__)
CORS(app)  # Enable CORS for API

@app.route('/api/health')
def health_check():
    """API health check endpoint."""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.utcnow().isoformat()
    })

@app.route('/api/v1/users', methods=['GET'])
def get_users():
    """Get all users."""
    users = User.query.all()
    return jsonify([user.to_dict() for user in users])

@app.route('/api/v1/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    """Get specific user."""
    user = User.query.get_or_404(user_id)
    return jsonify(user.to_dict())
```

---

## RESTful Routes

### Complete CRUD API

```python
from flask import Blueprint, jsonify, request, abort
from models import User, db

api_bp = Blueprint('api', __name__, url_prefix='/api/v1')

# CREATE
@api_bp.route('/users', methods=['POST'])
def create_user():
    """Create new user."""
    data = request.get_json()
    
    # Validation
    if not data or not data.get('username') or not data.get('email'):
        return jsonify({'error': 'Missing required fields'}), 400
    
    # Check if user exists
    if User.query.filter_by(username=data['username']).first():
        return jsonify({'error': 'Username already exists'}), 409
    
    # Create user
    user = User(
        username=data['username'],
        email=data['email']
    )
    
    if 'password' in data:
        user.set_password(data['password'])
    
    db.session.add(user)
    db.session.commit()
    
    return jsonify(user.to_dict()), 201

# READ (List)
@api_bp.route('/users', methods=['GET'])
def list_users():
    """List all users with filtering and pagination."""
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 20, type=int)
    
    # Apply filters
    query = User.query
    
    if 'active' in request.args:
        is_active = request.args.get('active').lower() == 'true'
        query = query.filter_by(is_active=is_active)
    
    if 'search' in request.args:
        search = f"%{request.args.get('search')}%"
        query = query.filter(
            db.or_(
                User.username.like(search),
                User.email.like(search)
            )
        )
    
    # Paginate
    pagination = query.paginate(page=page, per_page=per_page, error_out=False)
    
    return jsonify({
        'users': [user.to_dict() for user in pagination.items],
        'total': pagination.total,
        'pages': pagination.pages,
        'current_page': page,
        'per_page': per_page
    })

# READ (Single)
@api_bp.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    """Get single user by ID."""
    user = User.query.get_or_404(user_id)
    
    # Include related data
    include_posts = request.args.get('include_posts', 'false').lower() == 'true'
    
    user_data = user.to_dict()
    
    if include_posts:
        user_data['posts'] = [post.to_dict() for post in user.posts.all()]
    
    return jsonify(user_data)

# UPDATE
@api_bp.route('/users/<int:user_id>', methods=['PUT', 'PATCH'])
def update_user(user_id):
    """Update user."""
    user = User.query.get_or_404(user_id)
    data = request.get_json()
    
    # Authorization check
    if not current_user_can_edit(user):
        return jsonify({'error': 'Unauthorized'}), 403
    
    # Update fields
    updatable_fields = ['username', 'email', 'bio']
    
    for field in updatable_fields:
        if field in data:
            setattr(user, field, data[field])
    
    user.updated_at = datetime.utcnow()
    
    try:
        db.session.commit()
        return jsonify(user.to_dict())
    except Exception as e:
        db.session.rollback()
        return jsonify({'error': str(e)}), 400

# DELETE
@api_bp.route('/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    """Delete user."""
    user = User.query.get_or_404(user_id)
    
    # Authorization check
    if not current_user_is_admin():
        return jsonify({'error': 'Unauthorized'}), 403
    
    db.session.delete(user)
    db.session.commit()
    
    return '', 204
```

---

## Request Handling

### Parsing Request Data

```python
# JSON data
@api_bp.route('/posts', methods=['POST'])
def create_post():
    data = request.get_json()
    
    # Handle missing JSON
    if not data:
        return jsonify({'error': 'No JSON data provided'}), 400
    
    title = data.get('title')
    content = data.get('content')
    tags = data.get('tags', [])
    
    post = Post(title=title, content=content)
    db.session.add(post)
    db.session.commit()
    
    return jsonify(post.to_dict()), 201

# Form data
@api_bp.route('/upload', methods=['POST'])
def upload_file():
    if 'file' not in request.files:
        return jsonify({'error': 'No file provided'}), 400
    
    file = request.files['file']
    description = request.form.get('description', '')
    
    # Save file
    filename = secure_filename(file.filename)
    file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
    
    return jsonify({'filename': filename}), 201

# Query parameters
@api_bp.route('/search')
def search():
    query = request.args.get('q', '')
    page = request.args.get('page', 1, type=int)
    sort = request.args.get('sort', 'relevance')
    
    results = perform_search(query, page, sort)
    
    return jsonify(results)

# Headers
@api_bp.route('/api/data')
def get_data():
    api_key = request.headers.get('X-API-Key')
    content_type = request.headers.get('Content-Type')
    
    if not api_key:
        return jsonify({'error': 'API key required'}), 401
    
    # Validate API key
    if not validate_api_key(api_key):
        return jsonify({'error': 'Invalid API key'}), 401
    
    return jsonify({'data': 'secret information'})
```

### Input Validation

```python
from marshmallow import Schema, fields, ValidationError, validates

class UserSchema(Schema):
    username = fields.Str(required=True, validate=lambda x: len(x) >= 3)
    email = fields.Email(required=True)
    age = fields.Int(validate=lambda x: 0 < x < 150)
    bio = fields.Str(validate=lambda x: len(x) <= 500)
    
    @validates('username')
    def validate_username(self, value):
        if User.query.filter_by(username=value).first():
            raise ValidationError('Username already exists')

user_schema = UserSchema()
users_schema = UserSchema(many=True)

@api_bp.route('/users', methods=['POST'])
def create_user():
    """Create user with validation."""
    try:
        # Validate and deserialize input
        data = user_schema.load(request.get_json())
    except ValidationError as err:
        return jsonify({'errors': err.messages}), 400
    
    user = User(**data)
    db.session.add(user)
    db.session.commit()
    
    return jsonify(user_schema.dump(user)), 201
```

---

## Response Formatting

### Standard Response Format

```python
def success_response(data=None, message='Success', status=200):
    """Standard success response."""
    response = {
        'status': 'success',
        'message': message
    }
    
    if data is not None:
        response['data'] = data
    
    return jsonify(response), status

def error_response(message, errors=None, status=400):
    """Standard error response."""
    response = {
        'status': 'error',
        'message': message
    }
    
    if errors:
        response['errors'] = errors
    
    return jsonify(response), status

# Usage
@api_bp.route('/users', methods=['POST'])
def create_user():
    data = request.get_json()
    
    try:
        user = User(**data)
        db.session.add(user)
        db.session.commit()
        
        return success_response(
            data=user.to_dict(),
            message='User created successfully',
            status=201
        )
    except ValueError as e:
        return error_response(str(e), status=400)
```

### HATEOAS (Hypermedia)

```python
def add_links(resource_dict, resource_type, resource_id):
    """Add HATEOAS links to resource."""
    resource_dict['_links'] = {
        'self': url_for(f'api.get_{resource_type}', id=resource_id, _external=True),
        'collection': url_for(f'api.list_{resource_type}s', _external=True)
    }
    
    return resource_dict

class User(db.Model):
    # ... fields ...
    
    def to_dict(self, include_links=False):
        data = {
            'id': self.id,
            'username': self.username,
            'email': self.email,
            'created_at': self.created_at.isoformat()
        }
        
        if include_links:
            data['_links'] = {
                'self': url_for('api.get_user', user_id=self.id, _external=True),
                'posts': url_for('api.list_user_posts', user_id=self.id, _external=True),
                'avatar': url_for('static', filename=f'avatars/{self.id}.jpg', _external=True)
            }
        
        return data
```

---

## Error Handling

### Global Error Handlers

```python
@app.errorhandler(404)
def not_found(error):
    """Handle 404 errors."""
    return jsonify({
        'status': 'error',
        'message': 'Resource not found',
        'code': 404
    }), 404

@app.errorhandler(500)
def internal_error(error):
    """Handle 500 errors."""
    db.session.rollback()
    return jsonify({
        'status': 'error',
        'message': 'Internal server error',
        'code': 500
    }), 500

@app.errorhandler(400)
def bad_request(error):
    """Handle 400 errors."""
    return jsonify({
        'status': 'error',
        'message': 'Bad request',
        'code': 400
    }), 400

@app.errorhandler(403)
def forbidden(error):
    """Handle 403 errors."""
    return jsonify({
        'status': 'error',
        'message': 'Forbidden',
        'code': 403
    }), 403

@app.errorhandler(ValidationError)
def handle_validation_error(error):
    """Handle validation errors."""
    return jsonify({
        'status': 'error',
        'message': 'Validation failed',
        'errors': error.messages
    }), 400
```

### Custom Exceptions

```python
class APIException(Exception):
    """Base API exception."""
    status_code = 400
    
    def __init__(self, message, status_code=None, payload=None):
        super().__init__()
        self.message = message
        if status_code is not None:
            self.status_code = status_code
        self.payload = payload
    
    def to_dict(self):
        rv = dict(self.payload or ())
        rv['message'] = self.message
        rv['status'] = 'error'
        return rv

class ResourceNotFound(APIException):
    status_code = 404

class InvalidAPIUsage(APIException):
    status_code = 400

class Unauthorized(APIException):
    status_code = 401

# Register exception handler
@app.errorhandler(APIException)
def handle_api_exception(error):
    response = jsonify(error.to_dict())
    response.status_code = error.status_code
    return response

# Usage
@api_bp.route('/users/<int:user_id>')
def get_user(user_id):
    user = User.query.get(user_id)
    
    if not user:
        raise ResourceNotFound(f'User {user_id} not found')
    
    return jsonify(user.to_dict())
```

---

## Authentication

### Token-Based Authentication

```python
import jwt
from datetime import datetime, timedelta
from functools import wraps

def generate_token(user_id, expires_in=3600):
    """Generate JWT token."""
    payload = {
        'user_id': user_id,
        'exp': datetime.utcnow() + timedelta(seconds=expires_in),
        'iat': datetime.utcnow()
    }
    
    token = jwt.encode(payload, app.config['SECRET_KEY'], algorithm='HS256')
    return token

def verify_token(token):
    """Verify JWT token."""
    try:
        payload = jwt.decode(token, app.config['SECRET_KEY'], algorithms=['HS256'])
        return payload['user_id']
    except jwt.ExpiredSignatureError:
        return None
    except jwt.InvalidTokenError:
        return None

def token_required(f):
    """Decorator for token authentication."""
    @wraps(f)
    def decorated(*args, **kwargs):
        token = None
        
        # Get token from header
        if 'Authorization' in request.headers:
            auth_header = request.headers['Authorization']
            try:
                token = auth_header.split(' ')[1]  # Bearer TOKEN
            except IndexError:
                return jsonify({'error': 'Invalid token format'}), 401
        
        if not token:
            return jsonify({'error': 'Token is missing'}), 401
        
        # Verify token
        user_id = verify_token(token)
        
        if user_id is None:
            return jsonify({'error': 'Token is invalid or expired'}), 401
        
        # Get user
        user = User.query.get(user_id)
        if not user:
            return jsonify({'error': 'User not found'}), 401
        
        # Make user available in route
        g.current_user = user
        
        return f(*args, **kwargs)
    
    return decorated

# Usage
@api_bp.route('/auth/login', methods=['POST'])
def login():
    """Login and get token."""
    data = request.get_json()
    
    user = User.query.filter_by(username=data.get('username')).first()
    
    if user and user.check_password(data.get('password')):
        token = generate_token(user.id)
        
        return jsonify({
            'token': token,
            'expires_in': 3600,
            'user': user.to_dict()
        })
    
    return jsonify({'error': 'Invalid credentials'}), 401

@api_bp.route('/profile', methods=['GET'])
@token_required
def get_profile():
    """Get authenticated user profile."""
    return jsonify(g.current_user.to_dict())
```

---

## Pagination

### Cursor-Based Pagination

```python
@api_bp.route('/posts')
def list_posts():
    """List posts with cursor-based pagination."""
    cursor = request.args.get('cursor', type=int)
    limit = request.args.get('limit', 20, type=int)
    
    query = Post.query.filter_by(published=True).order_by(Post.id.desc())
    
    if cursor:
        query = query.filter(Post.id < cursor)
    
    posts = query.limit(limit + 1).all()
    
    has_more = len(posts) > limit
    posts = posts[:limit]
    
    next_cursor = posts[-1].id if has_more and posts else None
    
    return jsonify({
        'posts': [post.to_dict() for post in posts],
        'next_cursor': next_cursor,
        'has_more': has_more
    })
```

### Offset-Based Pagination

```python
def paginate_query(query, page=1, per_page=20):
    """Helper function for pagination."""
    pagination = query.paginate(page=page, per_page=per_page, error_out=False)
    
    return {
        'items': [item.to_dict() for item in pagination.items],
        'total': pagination.total,
        'pages': pagination.pages,
        'current_page': page,
        'per_page': per_page,
        'has_prev': pagination.has_prev,
        'has_next': pagination.has_next,
        'prev_page': pagination.prev_num,
        'next_page': pagination.next_num
    }

@api_bp.route('/users')
def list_users():
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 20, type=int)
    
    query = User.query.order_by(User.created_at.desc())
    result = paginate_query(query, page, per_page)
    
    return jsonify(result)
```

---

## Rate Limiting

### Flask-Limiter

```bash
pip install flask-limiter
```

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app=app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"],
    storage_uri="redis://localhost:6379"
)

# Apply to specific routes
@api_bp.route('/search')
@limiter.limit("10 per minute")
def search():
    query = request.args.get('q')
    results = perform_search(query)
    return jsonify(results)

# Dynamic limits based on user tier
@api_bp.route('/api/data')
@limiter.limit(lambda: get_user_rate_limit())
def get_data():
    return jsonify({'data': 'some data'})

def get_user_rate_limit():
    """Get rate limit based on user subscription."""
    if g.current_user.subscription_type == 'premium':
        return "1000 per hour"
    elif g.current_user.subscription_type == 'basic':
        return "100 per hour"
    else:
        return "10 per hour"

# Exempt from rate limiting
@api_bp.route('/public')
@limiter.exempt
def public_endpoint():
    return jsonify({'message': 'No rate limit'})
```

---

## Filtering and Sorting

### Advanced Filtering

```python
from sqlalchemy import and_, or_, not_

@api_bp.route('/posts')
def list_posts():
    """List posts with advanced filtering."""
    query = Post.query
    
    # Filter by author
    if 'author' in request.args:
        query = query.filter_by(author_id=request.args.get('author', type=int))
    
    # Filter by date range
    if 'start_date' in request.args:
        start_date = datetime.fromisoformat(request.args.get('start_date'))
        query = query.filter(Post.created_at >= start_date)
    
    if 'end_date' in request.args:
        end_date = datetime.fromisoformat(request.args.get('end_date'))
        query = query.filter(Post.created_at <= end_date)
    
    # Filter by tags
    if 'tags' in request.args:
        tags = request.args.get('tags').split(',')
        query = query.join(Post.tags).filter(Tag.name.in_(tags))
    
    # Sort
    sort_by = request.args.get('sort', 'created_at')
    order = request.args.get('order', 'desc')
    
    if sort_by == 'title':
        sort_column = Post.title
    elif sort_by == 'views':
        sort_column = Post.view_count
    else:
        sort_column = Post.created_at
    
    if order == 'asc':
        query = query.order_by(sort_column.asc())
    else:
        query = query.order_by(sort_column.desc())
    
    # Paginate
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 20, type=int)
    
    pagination = query.paginate(page=page, per_page=per_page)
    
    return jsonify({
        'posts': [post.to_dict() for post in pagination.items],
        'total': pagination.total,
        'page': page,
        'pages': pagination.pages
    })
```

---

## Versioning

### URL Versioning

```python
# API v1
api_v1 = Blueprint('api_v1', __name__, url_prefix='/api/v1')

@api_v1.route('/users')
def list_users_v1():
    users = User.query.all()
    return jsonify([user.to_dict() for user in users])

# API v2 with additional fields
api_v2 = Blueprint('api_v2', __name__, url_prefix='/api/v2')

@api_v2.route('/users')
def list_users_v2():
    users = User.query.all()
    return jsonify([user.to_dict_v2() for user in users])

# Register blueprints
app.register_blueprint(api_v1)
app.register_blueprint(api_v2)
```

### Header Versioning

```python
def get_api_version():
    """Get API version from header."""
    return request.headers.get('API-Version', 'v1')

@api_bp.route('/users')
def list_users():
    version = get_api_version()
    
    users = User.query.all()
    
    if version == 'v2':
        return jsonify([user.to_dict_v2() for user in users])
    else:
        return jsonify([user.to_dict() for user in users])
```

---

## Webhooks

### Webhook Endpoint

```python
@api_bp.route('/webhooks/github', methods=['POST'])
def github_webhook():
    """Handle GitHub webhook."""
    signature = request.headers.get('X-Hub-Signature-256')
    
    # Verify signature
    if not verify_github_signature(request.data, signature):
        return jsonify({'error': 'Invalid signature'}), 401
    
    event_type = request.headers.get('X-GitHub-Event')
    payload = request.get_json()
    
    if event_type == 'push':
        handle_push_event(payload)
    elif event_type == 'pull_request':
        handle_pr_event(payload)
    
    return jsonify({'status': 'processed'}), 200

def verify_github_signature(payload, signature):
    """Verify GitHub webhook signature."""
    import hmac
    import hashlib
    
    secret = app.config['GITHUB_WEBHOOK_SECRET'].encode()
    computed_signature = 'sha256=' + hmac.new(
        secret,
        payload,
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(computed_signature, signature)
```

---

## File Upload API

```python
from werkzeug.utils import secure_filename
import os

ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif', 'pdf'}

def allowed_file(filename):
    """Check if file extension is allowed."""
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

@api_bp.route('/upload', methods=['POST'])
@token_required
def upload_file():
    """Upload file endpoint."""
    if 'file' not in request.files:
        return jsonify({'error': 'No file part'}), 400
    
    file = request.files['file']
    
    if file.filename == '':
        return jsonify({'error': 'No selected file'}), 400
    
    if not allowed_file(file.filename):
        return jsonify({'error': 'File type not allowed'}), 400
    
    # Check file size (5MB max)
    file.seek(0, os.SEEK_END)
    file_length = file.tell()
    if file_length > 5 * 1024 * 1024:
        return jsonify({'error': 'File too large'}), 400
    file.seek(0)
    
    # Save file
    filename = secure_filename(file.filename)
    timestamp = datetime.utcnow().strftime('%Y%m%d_%H%M%S')
    unique_filename = f"{timestamp}_{filename}"
    
    upload_folder = app.config['UPLOAD_FOLDER']
    os.makedirs(upload_folder, exist_ok=True)
    
    file_path = os.path.join(upload_folder, unique_filename)
    file.save(file_path)
    
    # Create file record
    file_record = FileUpload(
        user_id=g.current_user.id,
        filename=unique_filename,
        original_filename=filename,
        file_path=file_path,
        file_size=file_length
    )
    db.session.add(file_record)
    db.session.commit()
    
    return jsonify({
        'id': file_record.id,
        'filename': unique_filename,
        'url': url_for('api.download_file', file_id=file_record.id, _external=True)
    }), 201

@api_bp.route('/files/<int:file_id>')
@token_required
def download_file(file_id):
    """Download file."""
    file_record = FileUpload.query.get_or_404(file_id)
    
    # Authorization check
    if file_record.user_id != g.current_user.id and not g.current_user.is_admin:
        return jsonify({'error': 'Unauthorized'}), 403
    
    return send_file(
        file_record.file_path,
        as_attachment=True,
        download_name=file_record.original_filename
    )
```

---

## Batch Operations

### Bulk Create

```python
@api_bp.route('/users/batch', methods=['POST'])
@token_required
def batch_create_users():
    """Create multiple users at once."""
    data = request.get_json()
    
    if not isinstance(data, list):
        return jsonify({'error': 'Expected list of users'}), 400
    
    created_users = []
    errors = []
    
    for index, user_data in enumerate(data):
        try:
            user = User(**user_data)
            db.session.add(user)
            created_users.append(user_data)
        except Exception as e:
            errors.append({
                'index': index,
                'data': user_data,
                'error': str(e)
            })
    
    try:
        db.session.commit()
        
        return jsonify({
            'created': len(created_users),
            'failed': len(errors),
            'errors': errors
        }), 201
    except Exception as e:
        db.session.rollback()
        return jsonify({'error': str(e)}), 400
```

---

## API Documentation

### Swagger/OpenAPI Setup

```bash
pip install flask-swagger-ui flasgger
```

```python
from flasgger import Swagger

swagger_config = {
    "headers": [],
    "specs": [
        {
            "endpoint": 'apispec',
            "route": '/apispec.json',
            "rule_filter": lambda rule: True,
            "model_filter": lambda tag: True,
        }
    ],
    "static_url_path": "/flasgger_static",
    "swagger_ui": True,
    "specs_route": "/docs/"
}

swagger = Swagger(app, config=swagger_config)

@api_bp.route('/users/<int:user_id>')
def get_user(user_id):
    """
    Get user by ID
    ---
    tags:
      - Users
    parameters:
      - name: user_id
        in: path
        type: integer
        required: true
        description: The user ID
    responses:
      200:
        description: User found
        schema:
          properties:
            id:
              type: integer
            username:
              type: string
            email:
              type: string
      404:
        description: User not found
    """
    user = User.query.get_or_404(user_id)
    return jsonify(user.to_dict())
```

---

## API Testing

### Example API Tests

```python
# tests/test_api.py
import unittest
import json
from app import app, db
from models import User

class APITestCase(unittest.TestCase):
    def setUp(self):
        app.config['TESTING'] = True
        app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///:memory:'
        self.client = app.test_client()
        
        with app.app_context():
            db.create_all()
    
    def tearDown(self):
        with app.app_context():
            db.session.remove()
            db.drop_all()
    
    def test_create_user(self):
        """Test POST /api/v1/users."""
        response = self.client.post(
            '/api/v1/users',
            data=json.dumps({
                'username': 'testuser',
                'email': 'test@example.com',
                'password': 'password123'
            }),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, 201)
        data = json.loads(response.data)
        self.assertEqual(data['username'], 'testuser')
    
    def test_get_users(self):
        """Test GET /api/v1/users."""
        response = self.client.get('/api/v1/users')
        
        self.assertEqual(response.status_code, 200)
        data = json.loads(response.data)
        self.assertIn('users', data)
    
    def test_authentication(self):
        """Test token authentication."""
        # Create user
        with app.app_context():
            user = User(username='test', email='test@example.com')
            user.set_password('password123')
            db.session.add(user)
            db.session.commit()
        
        # Login
        response = self.client.post(
            '/api/v1/auth/login',
            data=json.dumps({
                'username': 'test',
                'password': 'password123'
            }),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, 200)
        data = json.loads(response.data)
        token = data['token']
        
        # Access protected endpoint
        response = self.client.get(
            '/api/v1/profile',
            headers={'Authorization': f'Bearer {token}'}
        )
        
        self.assertEqual(response.status_code, 200)
```

---

## Complete API Example

```python
# api/__init__.py
from flask import Blueprint
from .users import users_bp
from .posts import posts_bp
from .auth import auth_bp

api = Blueprint('api', __name__, url_prefix='/api/v1')

# Register sub-blueprints
api.register_blueprint(users_bp)
api.register_blueprint(posts_bp)
api.register_blueprint(auth_bp)

# api/users.py
from flask import Blueprint, jsonify, request
from models import User, db

users_bp = Blueprint('users', __name__, url_prefix='/users')

@users_bp.route('', methods=['GET'])
def list_users():
    page = request.args.get('page', 1, type=int)
    users = User.query.paginate(page=page, per_page=20)
    
    return jsonify({
        'users': [user.to_dict() for user in users.items],
        'total': users.total,
        'pages': users.pages
    })

@users_bp.route('/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = User.query.get_or_404(user_id)
    return jsonify(user.to_dict())

@users_bp.route('', methods=['POST'])
@token_required
def create_user():
    data = request.get_json()
    
    user = User(**data)
    db.session.add(user)
    db.session.commit()
    
    return jsonify(user.to_dict()), 201

@users_bp.route('/<int:user_id>', methods=['PUT'])
@token_required
def update_user(user_id):
    user = User.query.get_or_404(user_id)
    
    if user.id != g.current_user.id and not g.current_user.is_admin:
        return jsonify({'error': 'Unauthorized'}), 403
    
    data = request.get_json()
    
    for key, value in data.items():
        if hasattr(user, key):
            setattr(user, key, value)
    
    db.session.commit()
    
    return jsonify(user.to_dict())

@users_bp.route('/<int:user_id>', methods=['DELETE'])
@token_required
def delete_user(user_id):
    if not g.current_user.is_admin:
        return jsonify({'error': 'Unauthorized'}), 403
    
    user = User.query.get_or_404(user_id)
    db.session.delete(user)
    db.session.commit()
    
    return '', 204
```

---

## Best Practices

### API Design Principles

1. **Use HTTP methods correctly**
   - GET: Retrieve data
   - POST: Create new resource
   - PUT: Update entire resource
   - PATCH: Partial update
   - DELETE: Remove resource

2. **Use proper status codes**
   - 200: Success
   - 201: Created
   - 204: No content
   - 400: Bad request
   - 401: Unauthorized
   - 403: Forbidden
   - 404: Not found
   - 500: Server error

3. **Version your API** - Use /api/v1, /api/v2

4. **Use JSON** - Standard format for APIs

5. **Implement pagination** - Don't return all records

6. **Add rate limiting** - Prevent abuse

7. **Document your API** - Use Swagger/OpenAPI

8. **Use HTTPS** - Encrypt data in transit

9. **Validate input** - Never trust client data

10. **Handle errors gracefully** - Return meaningful error messages

---

## Sample curl Commands

```bash
# GET request
curl -X GET http://localhost:5000/api/v1/users

# POST request with JSON
curl -X POST http://localhost:5000/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"username":"john","email":"john@example.com"}'

# With authentication token
curl -X GET http://localhost:5000/api/v1/profile \
  -H "Authorization: Bearer YOUR_TOKEN"

# PUT request
curl -X PUT http://localhost:5000/api/v1/users/1 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"email":"newemail@example.com"}'

# DELETE request
curl -X DELETE http://localhost:5000/api/v1/users/1 \
  -H "Authorization: Bearer YOUR_TOKEN"

# File upload
curl -X POST http://localhost:5000/api/v1/upload \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -F "file=@/path/to/file.jpg"

# Query parameters
curl -X GET "http://localhost:5000/api/v1/posts?page=2&per_page=10&sort=views"
```

---

## Related Documentation

- [Flask RESTful](https://flask-restful.readthedocs.io/)
- [Flask-RESTX](https://flask-restx.readthedocs.io/)
- [Marshmallow](https://marshmallow.readthedocs.io/)
- [Flask-Limiter](https://flask-limiter.readthedocs.io/)
