# Redis Caching with Flask

In-memory caching using Redis to improve Flask application performance.

## Table of Contents
- [Redis Setup](#redis-setup)
- [Flask-Caching](#flask-caching)
- [Basic Caching Patterns](#basic-caching-patterns)
- [Advanced Caching](#advanced-caching)
- [Session Storage](#session-storage)
- [Rate Limiting](#rate-limiting)

---

## Redis Setup

### Installation

```bash
# Install Redis server
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install redis-server

# macOS
brew install redis

# Start Redis
sudo systemctl start redis    # Linux
brew services start redis     # macOS

# Test Redis
redis-cli ping  # Should return PONG

# Python Redis client
pip install redis flask-caching
```

### Basic Redis Configuration

```python
# config.py
import os

class Config:
    REDIS_URL = os.environ.get('REDIS_URL') or 'redis://localhost:6379/0'
    CACHE_TYPE = 'redis'
    CACHE_REDIS_URL = REDIS_URL
    CACHE_DEFAULT_TIMEOUT = 300  # 5 minutes
```

---

## Flask-Caching

### Setup Flask-Caching

```python
# app.py
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)
app.config.from_object('config.Config')

cache = Cache(app)

# Alternative: Initialize separately
cache = Cache()

def create_app():
    app = Flask(__name__)
    cache.init_app(app, config={
        'CACHE_TYPE': 'redis',
        'CACHE_REDIS_URL': 'redis://localhost:6379/0'
    })
    return app
```

### Simple Caching Examples

```python
from app import app, cache
from flask import jsonify

# Cache a view function
@app.route('/expensive-operation')
@cache.cached(timeout=300)  # Cache for 5 minutes
def expensive_operation():
    # Simulate expensive computation
    import time
    time.sleep(2)
    result = perform_complex_calculation()
    return jsonify(result)

# Cache with query parameters
@app.route('/users/<int:user_id>')
@cache.cached(timeout=60, query_string=True)
def get_user(user_id):
    user = User.query.get_or_404(user_id)
    return jsonify(user.to_dict())

# Cache unless condition
@app.route('/posts')
@cache.cached(timeout=120, unless=lambda: 'refresh' in request.args)
def get_posts():
    posts = Post.query.all()
    return jsonify([p.to_dict() for p in posts])
```

---

## Basic Caching Patterns

### Cache Functions (Memoization)

```python
# Cache function results
@cache.memoize(timeout=600)
def get_user_statistics(user_id):
    """Expensive statistics calculation."""
    user = User.query.get(user_id)
    
    return {
        'total_posts': user.posts.count(),
        'total_views': sum(p.view_count for p in user.posts),
        'avg_views': user.posts.with_entities(
            db.func.avg(Post.view_count)
        ).scalar()
    }

# Usage - automatically cached
stats = get_user_statistics(42)

# Clear specific cache
cache.delete_memoized(get_user_statistics, 42)

# Clear all memoized results for function
cache.delete_memoized(get_user_statistics)
```

### Manual Cache Operations

```python
# Set cache manually
cache.set('key', 'value', timeout=300)

# Get from cache
value = cache.get('key')

# Delete from cache
cache.delete('key')

# Check if key exists
if cache.has('key'):
    value = cache.get('key')

# Set multiple values
cache.set_many({
    'key1': 'value1',
    'key2': 'value2',
    'key3': 'value3'
}, timeout=300)

# Get multiple values
values = cache.get_many('key1', 'key2', 'key3')

# Delete multiple keys
cache.delete_many('key1', 'key2', 'key3')

# Clear all cache
cache.clear()

# Add (only if not exists)
cache.add('key', 'value', timeout=300)
```

---

## Advanced Caching

### Cache with Dynamic Keys

```python
def get_cache_key(user_id, resource_type):
    """Generate dynamic cache key."""
    return f'user:{user_id}:{resource_type}'

def get_user_data(user_id):
    """Get user data with custom cache key."""
    cache_key = get_cache_key(user_id, 'profile')
    
    # Try to get from cache
    data = cache.get(cache_key)
    
    if data is None:
        # Cache miss - fetch from database
        user = User.query.get(user_id)
        data = user.to_dict()
        
        # Store in cache
        cache.set(cache_key, data, timeout=600)
    
    return data

def invalidate_user_cache(user_id):
    """Invalidate all cache entries for a user."""
    patterns = [
        f'user:{user_id}:profile',
        f'user:{user_id}:posts',
        f'user:{user_id}:stats'
    ]
    
    for pattern in patterns:
        cache.delete(pattern)
```

### Cache Decorator with Arguments

```python
from functools import wraps

def cached_by_user(timeout=300):
    """Custom cache decorator that includes user_id in key."""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            user_id = kwargs.get('user_id') or g.user.id
            cache_key = f'{f.__name__}:{user_id}'
            
            result = cache.get(cache_key)
            if result is None:
                result = f(*args, **kwargs)
                cache.set(cache_key, result, timeout=timeout)
            
            return result
        return decorated_function
    return decorator

# Usage
@app.route('/user/<int:user_id>/dashboard')
@cached_by_user(timeout=600)
def user_dashboard(user_id):
    # Expensive operations
    return render_template('dashboard.html', data=get_dashboard_data(user_id))
```

### Cache Invalidation Patterns

```python
def update_user_profile(user_id, **updates):
    """Update user and invalidate related caches."""
    user = User.query.get(user_id)
    
    for key, value in updates.items():
        setattr(user, key, value)
    
    db.session.commit()
    
    # Invalidate caches
    cache.delete(f'user:{user_id}:profile')
    cache.delete(f'user:{user_id}:stats')
    cache.delete_memoized(get_user_statistics, user_id)
    
    return user

def cache_aside_pattern(key, fetch_func, timeout=300):
    """Implement cache-aside (lazy loading) pattern."""
    data = cache.get(key)
    
    if data is None:
        data = fetch_func()
        cache.set(key, data, timeout=timeout)
    
    return data

# Usage
posts = cache_aside_pattern(
    'recent_posts',
    lambda: Post.query.order_by(Post.created_at.desc()).limit(10).all(),
    timeout=600
)
```

---

## Direct Redis Client

### Using Redis Client Directly

```python
# extensions.py
import redis
from flask import current_app

class RedisClient:
    def __init__(self):
        self.client = None
    
    def init_app(self, app):
        self.client = redis.from_url(
            app.config['REDIS_URL'],
            decode_responses=True
        )
    
    def get(self, key):
        return self.client.get(key)
    
    def set(self, key, value, ex=None):
        return self.client.set(key, value, ex=ex)
    
    def delete(self, key):
        return self.client.delete(key)
    
    def exists(self, key):
        return self.client.exists(key)
    
    def incr(self, key):
        return self.client.incr(key)
    
    def expire(self, key, seconds):
        return self.client.expire(key, seconds)

redis_client = RedisClient()

# app.py
from extensions import redis_client

app = Flask(__name__)
redis_client.init_app(app)
```

### Redis Data Structures

```python
# String operations
redis_client.client.set('user:1:name', 'John Doe')
name = redis_client.client.get('user:1:name')

# Set with expiration
redis_client.client.setex('session:abc123', 3600, 'user_data')

# Increment counter
redis_client.client.incr('post:5:views')
redis_client.client.incrby('post:5:views', 10)

# Hash operations (like dictionaries)
redis_client.client.hset('user:1', 'name', 'John')
redis_client.client.hset('user:1', 'email', 'john@example.com')
redis_client.client.hgetall('user:1')  # Get all fields

redis_client.client.hmset('user:2', {
    'name': 'Jane',
    'email': 'jane@example.com',
    'age': 28
})

# List operations (queues)
redis_client.client.lpush('notifications', 'New message')  # Add to left
redis_client.client.rpush('tasks', 'Process data')         # Add to right
redis_client.client.lpop('notifications')                  # Remove from left
redis_client.client.lrange('notifications', 0, -1)         # Get all

# Set operations (unique values)
redis_client.client.sadd('user:1:followers', 'user2', 'user3')
redis_client.client.smembers('user:1:followers')
redis_client.client.sismember('user:1:followers', 'user2')  # Check membership

# Sorted sets (leaderboards)
redis_client.client.zadd('leaderboard', {'user1': 100, 'user2': 150, 'user3': 120})
redis_client.client.zrange('leaderboard', 0, -1, withscores=True)  # Ascending
redis_client.client.zrevrange('leaderboard', 0, 9, withscores=True)  # Top 10

# Key management
redis_client.client.keys('user:*')  # Get all keys matching pattern
redis_client.client.delete('key1', 'key2', 'key3')
redis_client.client.expire('temp_data', 60)  # Expire in 60 seconds
redis_client.client.ttl('temp_data')  # Time to live
```

---

## Session Storage

### Use Redis for Session Storage

```python
# app.py
from flask import Flask, session
from flask_session import Session

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key'
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_REDIS'] = redis.from_url('redis://localhost:6379/0')
app.config['SESSION_PERMANENT'] = False
app.config['SESSION_USE_SIGNER'] = True

Session(app)

@app.route('/login', methods=['POST'])
def login():
    # Session data stored in Redis
    session['user_id'] = user.id
    session['username'] = user.username
    session['logged_in_at'] = datetime.utcnow().isoformat()
    return redirect(url_for('dashboard'))

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('index'))
```

---

## Rate Limiting

### Simple Rate Limiter

```python
from functools import wraps
from flask import request, jsonify

def rate_limit(max_requests=100, window=3600):
    """Rate limit decorator using Redis."""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            # Use IP address as identifier
            identifier = request.remote_addr
            key = f'rate_limit:{f.__name__}:{identifier}'
            
            # Get current count
            current = redis_client.client.get(key)
            
            if current is None:
                # First request in window
                redis_client.client.setex(key, window, 1)
            elif int(current) >= max_requests:
                # Rate limit exceeded
                return jsonify({
                    'error': 'Rate limit exceeded',
                    'retry_after': redis_client.client.ttl(key)
                }), 429
            else:
                # Increment counter
                redis_client.client.incr(key)
            
            return f(*args, **kwargs)
        return decorated_function
    return decorator

# Usage
@app.route('/api/search')
@rate_limit(max_requests=10, window=60)  # 10 requests per minute
def search():
    query = request.args.get('q')
    results = perform_search(query)
    return jsonify(results)
```

### Advanced Rate Limiting

```python
from datetime import datetime, timedelta

class RateLimiter:
    """Sliding window rate limiter."""
    
    def __init__(self, redis_client, max_requests, window_seconds):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window = window_seconds
    
    def is_allowed(self, identifier, resource):
        """Check if request is allowed."""
        key = f'rate_limit:{resource}:{identifier}'
        now = datetime.utcnow().timestamp()
        
        # Remove old entries
        self.redis.zremrangebyscore(key, 0, now - self.window)
        
        # Count requests in current window
        request_count = self.redis.zcard(key)
        
        if request_count < self.max_requests:
            # Add current request
            self.redis.zadd(key, {now: now})
            self.redis.expire(key, self.window)
            return True
        
        return False
    
    def get_remaining(self, identifier, resource):
        """Get remaining requests."""
        key = f'rate_limit:{resource}:{identifier}'
        now = datetime.utcnow().timestamp()
        
        self.redis.zremrangebyscore(key, 0, now - self.window)
        request_count = self.redis.zcard(key)
        
        return max(0, self.max_requests - request_count)

# Usage
rate_limiter = RateLimiter(redis_client.client, max_requests=100, window_seconds=3600)

@app.before_request
def check_rate_limit():
    if request.endpoint and request.endpoint.startswith('api.'):
        identifier = request.headers.get('X-API-Key') or request.remote_addr
        
        if not rate_limiter.is_allowed(identifier, request.endpoint):
            return jsonify({'error': 'Rate limit exceeded'}), 429
```

---

## Caching Strategies

### Cache-Aside (Lazy Loading)

```python
def get_popular_posts():
    """Get popular posts with cache-aside pattern."""
    cache_key = 'popular_posts'
    
    # Try cache first
    posts = cache.get(cache_key)
    
    if posts is None:
        # Cache miss - query database
        posts = Post.query.filter_by(published=True)\
            .order_by(Post.view_count.desc())\
            .limit(10).all()
        
        # Convert to dict for JSON serialization
        posts = [p.to_dict() for p in posts]
        
        # Store in cache
        cache.set(cache_key, posts, timeout=300)
    
    return posts
```

### Write-Through Cache

```python
def create_post_with_cache(user_id, title, content):
    """Create post and update cache immediately."""
    post = Post(
        user_id=user_id,
        title=title,
        content=content,
        slug=slugify(title)
    )
    
    db.session.add(post)
    db.session.commit()
    
    # Update cache
    cache_key = f'user:{user_id}:posts'
    cache.delete(cache_key)  # Invalidate list cache
    
    # Cache the new post
    cache.set(f'post:{post.id}', post.to_dict(), timeout=600)
    
    return post
```

### Cache Warming

```python
def warm_cache():
    """Pre-populate cache with frequently accessed data."""
    
    # Cache popular posts
    popular_posts = Post.query.order_by(Post.view_count.desc()).limit(20).all()
    cache.set('popular_posts', [p.to_dict() for p in popular_posts], timeout=3600)
    
    # Cache active users
    active_users = User.query.filter_by(is_active=True).limit(100).all()
    cache.set('active_users', [u.to_dict() for u in active_users], timeout=1800)
    
    # Cache individual user data
    for user in active_users:
        cache.set(f'user:{user.id}', user.to_dict(), timeout=3600)
    
    print("Cache warmed successfully")

# Run on application startup or scheduled task
@app.before_first_request
def initialize():
    warm_cache()
```

---

## Real-World Examples

### Caching API Responses

```python
import json

@app.route('/api/feed')
def api_feed():
    """Cached API feed."""
    page = request.args.get('page', 1, type=int)
    cache_key = f'api_feed:page:{page}'
    
    # Check cache
    cached_data = cache.get(cache_key)
    if cached_data:
        return jsonify(json.loads(cached_data))
    
    # Fetch from database
    posts = Post.query.filter_by(published=True)\
        .order_by(Post.published_at.desc())\
        .paginate(page=page, per_page=20)
    
    data = {
        'posts': [p.to_dict() for p in posts.items],
        'total': posts.total,
        'pages': posts.pages,
        'current_page': page
    }
    
    # Cache for 5 minutes
    cache.set(cache_key, json.dumps(data), timeout=300)
    
    return jsonify(data)
```

### Fragment Caching in Templates

```python
# Template fragment caching
from flask import render_template_string

@app.route('/dashboard')
def dashboard():
    user = g.user
    
    # Cache expensive template fragments
    sidebar_html = cache.get(f'sidebar:{user.id}')
    if sidebar_html is None:
        sidebar_html = render_template_string(
            '{% include "components/sidebar.html" %}',
            user=user
        )
        cache.set(f'sidebar:{user.id}', sidebar_html, timeout=600)
    
    return render_template('dashboard.html',
                         user=user,
                         sidebar_html=sidebar_html)
```

### Database Query Result Caching

```python
def cached_query(cache_key, query_func, timeout=300):
    """Generic cached query wrapper."""
    result = cache.get(cache_key)
    
    if result is None:
        result = query_func()
        
        # Convert to serializable format
        if isinstance(result, list):
            result = [item.to_dict() if hasattr(item, 'to_dict') else item 
                     for item in result]
        elif hasattr(result, 'to_dict'):
            result = result.to_dict()
        
        cache.set(cache_key, result, timeout=timeout)
    
    return result

# Usage
users = cached_query(
    'all_active_users',
    lambda: User.query.filter_by(is_active=True).all(),
    timeout=600
)
```

---

## Leaderboard with Redis

```python
def update_user_score(user_id, score):
    """Update user score in leaderboard."""
    redis_client.client.zadd('leaderboard', {f'user:{user_id}': score})

def get_leaderboard(top_n=10):
    """Get top users from leaderboard."""
    results = redis_client.client.zrevrange(
        'leaderboard', 
        0, 
        top_n - 1, 
        withscores=True
    )
    
    leaderboard = []
    for user_key, score in results:
        user_id = int(user_key.split(':')[1])
        user = User.query.get(user_id)
        leaderboard.append({
            'user': user.to_dict(),
            'score': int(score)
        })
    
    return leaderboard

def get_user_rank(user_id):
    """Get user's rank in leaderboard."""
    rank = redis_client.client.zrevrank('leaderboard', f'user:{user_id}')
    return rank + 1 if rank is not None else None

def increment_user_score(user_id, amount=1):
    """Increment user score."""
    return redis_client.client.zincrby('leaderboard', amount, f'user:{user_id}')
```

---

## Counters and Analytics

### Real-time Counters

```python
def track_page_view(page_name):
    """Track page views."""
    today = datetime.utcnow().strftime('%Y-%m-%d')
    
    # Increment daily counter
    redis_client.client.incr(f'pageviews:{page_name}:{today}')
    
    # Increment all-time counter
    redis_client.client.incr(f'pageviews:{page_name}:total')

def get_page_views(page_name, date=None):
    """Get page views for specific date."""
    if date is None:
        date = datetime.utcnow().strftime('%Y-%m-%d')
    
    daily = redis_client.client.get(f'pageviews:{page_name}:{date}') or 0
    total = redis_client.client.get(f'pageviews:{page_name}:total') or 0
    
    return {
        'daily': int(daily),
        'total': int(total)
    }

def track_user_activity(user_id, activity_type):
    """Track user activities."""
    timestamp = datetime.utcnow().isoformat()
    
    # Add to user's activity stream (keep last 100)
    redis_client.client.lpush(f'activity:{user_id}', f'{activity_type}:{timestamp}')
    redis_client.client.ltrim(f'activity:{user_id}', 0, 99)
    
    # Track activity type count
    redis_client.client.hincrby(f'activity_stats:{user_id}', activity_type, 1)

def get_user_activity(user_id, limit=20):
    """Get user's recent activity."""
    activities = redis_client.client.lrange(f'activity:{user_id}', 0, limit - 1)
    return [act.decode() if isinstance(act, bytes) else act for act in activities]
```

---

## Pub/Sub Pattern

### Real-time Notifications

```python
def publish_notification(channel, message):
    """Publish notification to channel."""
    redis_client.client.publish(channel, json.dumps(message))

def subscribe_to_notifications(channels):
    """Subscribe to notification channels."""
    pubsub = redis_client.client.pubsub()
    pubsub.subscribe(channels)
    
    for message in pubsub.listen():
        if message['type'] == 'message':
            data = json.loads(message['data'])
            handle_notification(data)

# Example: Notify when new post is created
@app.route('/posts', methods=['POST'])
def create_post():
    post = Post(**request.json)
    db.session.add(post)
    db.session.commit()
    
    # Publish notification
    publish_notification('new_posts', {
        'post_id': post.id,
        'title': post.title,
        'author': post.author.username
    })
    
    return jsonify(post.to_dict()), 201
```

---

## Cache Configuration Strategies

### Multiple Cache Backends

```python
# app.py
from flask_caching import Cache

# Short-term cache for API responses
api_cache = Cache(config={
    'CACHE_TYPE': 'redis',
    'CACHE_REDIS_URL': 'redis://localhost:6379/0',
    'CACHE_DEFAULT_TIMEOUT': 300
})

# Long-term cache for rarely changing data
static_cache = Cache(config={
    'CACHE_TYPE': 'redis',
    'CACHE_REDIS_URL': 'redis://localhost:6379/1',
    'CACHE_DEFAULT_TIMEOUT': 3600
})

api_cache.init_app(app)
static_cache.init_app(app)

# Usage
@app.route('/api/data')
@api_cache.cached(timeout=60)
def api_data():
    return jsonify(get_data())

@app.route('/static-content')
@static_cache.cached(timeout=3600)
def static_content():
    return render_template('static.html')
```

---

## Monitoring and Debugging

### Cache Statistics

```python
@app.route('/admin/cache-stats')
def cache_stats():
    """Display cache statistics."""
    info = redis_client.client.info('stats')
    
    stats = {
        'total_connections': info.get('total_connections_received'),
        'total_commands': info.get('total_commands_processed'),
        'used_memory': info.get('used_memory_human'),
        'connected_clients': info.get('connected_clients'),
        'uptime_days': info.get('uptime_in_days')
    }
    
    # Get key count
    stats['total_keys'] = redis_client.client.dbsize()
    
    # Sample keys
    stats['sample_keys'] = redis_client.client.keys('*')[:10]
    
    return jsonify(stats)

def get_cache_hit_rate():
    """Calculate cache hit rate."""
    info = redis_client.client.info('stats')
    hits = info.get('keyspace_hits', 0)
    misses = info.get('keyspace_misses', 0)
    
    total = hits + misses
    hit_rate = (hits / total * 100) if total > 0 else 0
    
    return {
        'hits': hits,
        'misses': misses,
        'hit_rate': f'{hit_rate:.2f}%'
    }
```

### Cache Debugging

```python
@app.route('/debug/cache/<key>')
def debug_cache(key):
    """Debug specific cache key."""
    value = cache.get(key)
    ttl = redis_client.client.ttl(key)
    
    return jsonify({
        'key': key,
        'exists': cache.has(key),
        'value': value,
        'ttl': ttl,
        'type': redis_client.client.type(key).decode()
    })

@app.route('/admin/clear-cache', methods=['POST'])
def clear_cache():
    """Clear all cache (admin only)."""
    cache.clear()
    return jsonify({'status': 'Cache cleared'})

@app.route('/admin/cache-keys')
def list_cache_keys():
    """List all cache keys."""
    pattern = request.args.get('pattern', '*')
    keys = redis_client.client.keys(pattern)
    
    return jsonify({
        'count': len(keys),
        'keys': [k.decode() if isinstance(k, bytes) else k for k in keys]
    })
```

---

## Complete Caching Example

```python
# models.py
from app import db, cache
from datetime import datetime

class Article(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200))
    content = db.Column(db.Text)
    view_count = db.Column(db.Integer, default=0)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    @staticmethod
    @cache.memoize(timeout=600)
    def get_cached(article_id):
        """Get article with caching."""
        return Article.query.get(article_id)
    
    @staticmethod
    def invalidate_cache(article_id):
        """Invalidate article cache."""
        cache.delete_memoized(Article.get_cached, article_id)
    
    def increment_views(self):
        """Increment view count with Redis counter."""
        # Use Redis for counter (faster)
        counter_key = f'article:{self.id}:views'
        views = redis_client.client.incr(counter_key)
        
        # Persist to database every 10 views
        if views % 10 == 0:
            self.view_count = views
            db.session.commit()
        
        return views

# routes.py
@app.route('/article/<int:article_id>')
def view_article(article_id):
    """View article with multiple caching layers."""
    
    # Try to get from cache
    cache_key = f'article:{article_id}:rendered'
    cached_html = cache.get(cache_key)
    
    if cached_html:
        return cached_html
    
    # Get article (uses memoize cache)
    article = Article.get_cached(article_id)
    
    if not article:
        abort(404)
    
    # Increment views (uses Redis counter)
    article.increment_views()
    
    # Render template
    html = render_template('article.html', article=article)
    
    # Cache rendered HTML
    cache.set(cache_key, html, timeout=300)
    
    return html
```

---

## Best Practices

### DO's

1. **Set appropriate TTLs** - Shorter for frequently changing data
2. **Use cache namespaces** - Prefix keys by feature/module
3. **Invalidate on updates** - Clear cache when data changes
4. **Monitor cache hit rates** - Aim for 80%+ hit rate
5. **Use connection pooling** - Reuse Redis connections
6. **Serialize complex objects** - Use JSON or pickle
7. **Handle cache failures gracefully** - Fall back to database
8. **Use Redis for session storage** - Better than file-based sessions

### DON'Ts

1. **Don't cache sensitive data** - Passwords, credit cards, etc.
2. **Don't set infinite TTLs** - Data can become stale
3. **Don't cache large objects** - Keep cache entries small
4. **Don't ignore cache misses** - Monitor and optimize
5. **Don't forget to handle None** - Cache might return None
6. **Don't use `keys *` in production** - Use SCAN instead

---

## Cache Patterns Summary

| Pattern | Use Case | Example |
|---------|----------|---------|
| **Cache-Aside** | General caching | Check cache → if miss, query DB → store in cache |
| **Write-Through** | Data consistency | Write to DB → immediately update cache |
| **Write-Behind** | High-write loads | Write to cache → async persist to DB |
| **Refresh-Ahead** | Predictable access | Pre-refresh cache before expiration |
| **Read-Through** | Transparent caching | Cache automatically loads from DB on miss |

---

## Redis Commands Reference

| Command | Description | Example |
|---------|-------------|---------|
| `GET key` | Get value | `redis_client.get('user:1')` |
| `SET key value` | Set value | `redis_client.set('key', 'value')` |
| `SETEX key seconds value` | Set with expiration | `redis_client.setex('key', 60, 'value')` |
| `DEL key` | Delete key | `redis_client.delete('key')` |
| `EXISTS key` | Check if exists | `redis_client.exists('key')` |
| `EXPIRE key seconds` | Set expiration | `redis_client.expire('key', 300)` |
| `TTL key` | Get time to live | `redis_client.ttl('key')` |
| `INCR key` | Increment | `redis_client.incr('counter')` |
| `KEYS pattern` | Find keys | `redis_client.keys('user:*')` |
| `FLUSHDB` | Clear database | `redis_client.flushdb()` |
| `HSET key field value` | Hash set | `redis_client.hset('user:1', 'name', 'John')` |
| `HGETALL key` | Get all hash fields | `redis_client.hgetall('user:1')` |
| `LPUSH key value` | List push left | `redis_client.lpush('queue', 'task')` |
| `RPOP key` | List pop right | `redis_client.rpop('queue')` |
| `SADD key member` | Set add | `redis_client.sadd('tags', 'python')` |
| `ZADD key score member` | Sorted set add | `redis_client.zadd('scores', {'user1': 100})` |

---

## Related Documentation

- [Redis Documentation](https://redis.io/documentation)
- [Flask-Caching Documentation](https://flask-caching.readthedocs.io/)
- [Redis Python Client](https://redis-py.readthedocs.io/)
