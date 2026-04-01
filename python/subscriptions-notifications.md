# Subscriptions and Notifications in Flask

Complete guide to implementing subscription systems and notification logic in Flask applications.

## Table of Contents
- [Email Notifications](#email-notifications)
- [Subscription System](#subscription-system)
- [Real-time Notifications](#real-time-notifications)
- [Background Tasks](#background-tasks)
- [Webhook Integration](#webhook-integration)
- [Push Notifications](#push-notifications)

---

## Email Notifications

### Flask-Mail Setup

```bash
pip install flask-mail
```

```python
# app.py
from flask import Flask
from flask_mail import Mail, Message

app = Flask(__name__)

# Email configuration
app.config['MAIL_SERVER'] = 'smtp.gmail.com'
app.config['MAIL_PORT'] = 587
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USERNAME'] = 'your-email@gmail.com'
app.config['MAIL_PASSWORD'] = 'your-app-password'
app.config['MAIL_DEFAULT_SENDER'] = 'noreply@yourapp.com'

mail = Mail(app)
```

### Sending Basic Emails

```python
from flask_mail import Message
from flask import render_template

def send_email(subject, recipients, text_body, html_body):
    """Send email with text and HTML versions."""
    msg = Message(
        subject=subject,
        recipients=recipients,
        body=text_body,
        html=html_body
    )
    mail.send(msg)

def send_welcome_email(user):
    """Send welcome email to new user."""
    subject = 'Welcome to Our App!'
    
    text_body = f"""
    Hi {user.username},
    
    Welcome to our application! We're excited to have you here.
    
    Best regards,
    The Team
    """
    
    html_body = render_template(
        'email/welcome.html',
        user=user
    )
    
    send_email(subject, [user.email], text_body, html_body)

def send_password_reset_email(user, reset_url):
    """Send password reset email."""
    subject = 'Reset Your Password'
    
    text_body = f"""
    Hi {user.username},
    
    To reset your password, click the following link:
    {reset_url}
    
    This link expires in 1 hour.
    
    If you didn't request this, please ignore this email.
    """
    
    html_body = render_template(
        'email/reset_password.html',
        user=user,
        reset_url=reset_url
    )
    
    send_email(subject, [user.email], text_body, html_body)
```

### Email Templates

```jinja2
{# templates/email/base.html #}
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background: #007bff; color: white; padding: 20px; text-align: center; }
        .content { padding: 20px; background: #f8f9fa; }
        .button { background: #007bff; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px; display: inline-block; }
        .footer { text-align: center; padding: 20px; color: #666; font-size: 12px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>{% block header %}My App{% endblock %}</h1>
        </div>
        <div class="content">
            {% block content %}{% endblock %}
        </div>
        <div class="footer">
            {% block footer %}
                <p>&copy; 2026 My App. All rights reserved.</p>
                <p><a href="{{ url_for('unsubscribe', _external=True) }}">Unsubscribe</a></p>
            {% endblock %}
        </div>
    </div>
</body>
</html>
```

```jinja2
{# templates/email/notification.html #}
{% extends "email/base.html" %}

{% block header %}New Notification{% endblock %}

{% block content %}
    <h2>Hi {{ user.username }},</h2>
    
    <p>{{ notification.message }}</p>
    
    {% if notification.action_url %}
        <p style="text-align: center; margin: 30px 0;">
            <a href="{{ notification.action_url }}" class="button">View Details</a>
        </p>
    {% endif %}
    
    <p>Best regards,<br>The Team</p>
{% endblock %}
```

---

## Subscription System

### Database Models

```python
# models.py
from datetime import datetime, timedelta
from app import db

class Subscription(db.Model):
    __tablename__ = 'subscriptions'
    
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    subscription_type = db.Column(db.String(50), nullable=False)  # 'free', 'premium', 'enterprise'
    status = db.Column(db.String(20), default='active')  # 'active', 'cancelled', 'expired'
    
    # Dates
    started_at = db.Column(db.DateTime, default=datetime.utcnow)
    expires_at = db.Column(db.DateTime)
    cancelled_at = db.Column(db.DateTime)
    
    # Payment info
    stripe_subscription_id = db.Column(db.String(100))
    
    user = db.relationship('User', backref='subscription')
    
    def is_active(self):
        """Check if subscription is active."""
        return (self.status == 'active' and 
                (not self.expires_at or self.expires_at > datetime.utcnow()))
    
    def days_remaining(self):
        """Get days remaining in subscription."""
        if not self.expires_at:
            return None
        delta = self.expires_at - datetime.utcnow()
        return max(0, delta.days)
    
    def cancel(self):
        """Cancel subscription."""
        self.status = 'cancelled'
        self.cancelled_at = datetime.utcnow()
        db.session.commit()
    
    def renew(self, duration_days=30):
        """Renew subscription."""
        self.status = 'active'
        self.expires_at = datetime.utcnow() + timedelta(days=duration_days)
        db.session.commit()

class NotificationPreference(db.Model):
    __tablename__ = 'notification_preferences'
    
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    
    # Email preferences
    email_notifications = db.Column(db.Boolean, default=True)
    email_marketing = db.Column(db.Boolean, default=True)
    email_digest = db.Column(db.String(20), default='daily')  # 'never', 'daily', 'weekly'
    
    # In-app notifications
    push_notifications = db.Column(db.Boolean, default=True)
    
    # Specific notification types
    notify_new_follower = db.Column(db.Boolean, default=True)
    notify_comment = db.Column(db.Boolean, default=True)
    notify_mention = db.Column(db.Boolean, default=True)
    notify_subscription_expiry = db.Column(db.Boolean, default=True)
    
    user = db.relationship('User', backref='notification_prefs', uselist=False)
```

### Subscription Management

```python
# subscription.py
from models import Subscription, db
from datetime import datetime, timedelta

def create_subscription(user_id, subscription_type='free', duration_days=30):
    """Create new subscription."""
    subscription = Subscription(
        user_id=user_id,
        subscription_type=subscription_type,
        status='active',
        started_at=datetime.utcnow(),
        expires_at=datetime.utcnow() + timedelta(days=duration_days) if subscription_type != 'free' else None
    )
    
    db.session.add(subscription)
    db.session.commit()
    
    return subscription

def upgrade_subscription(user_id, new_type):
    """Upgrade user subscription."""
    subscription = Subscription.query.filter_by(user_id=user_id).first()
    
    if not subscription:
        return create_subscription(user_id, new_type)
    
    subscription.subscription_type = new_type
    subscription.status = 'active'
    
    if new_type == 'premium':
        subscription.expires_at = datetime.utcnow() + timedelta(days=30)
    elif new_type == 'enterprise':
        subscription.expires_at = datetime.utcnow() + timedelta(days=365)
    
    db.session.commit()
    
    # Send upgrade notification
    send_subscription_email(user_id, 'upgraded', new_type)
    
    return subscription

def check_subscription_expiry():
    """Check for expiring subscriptions and send notifications."""
    # Find subscriptions expiring in 3 days
    three_days_from_now = datetime.utcnow() + timedelta(days=3)
    
    expiring_subscriptions = Subscription.query.filter(
        Subscription.status == 'active',
        Subscription.expires_at <= three_days_from_now,
        Subscription.expires_at > datetime.utcnow()
    ).all()
    
    for sub in expiring_subscriptions:
        send_expiry_notification(sub.user, sub.days_remaining())

def cancel_subscription(user_id):
    """Cancel subscription."""
    subscription = Subscription.query.filter_by(user_id=user_id).first()
    
    if subscription:
        subscription.cancel()
        send_cancellation_email(subscription.user)
        
        return True
    
    return False
```

---

## Notification System

### Notification Model

```python
# models.py
class Notification(db.Model):
    __tablename__ = 'notifications'
    
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    
    type = db.Column(db.String(50), nullable=False)  # 'comment', 'follow', 'mention', etc.
    title = db.Column(db.String(200), nullable=False)
    message = db.Column(db.Text, nullable=False)
    
    # Related object
    related_type = db.Column(db.String(50))  # 'post', 'comment', etc.
    related_id = db.Column(db.Integer)
    
    # Status
    is_read = db.Column(db.Boolean, default=False)
    read_at = db.Column(db.DateTime)
    
    # Link
    action_url = db.Column(db.String(500))
    
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    user = db.relationship('User', backref='notifications')
    
    def mark_as_read(self):
        """Mark notification as read."""
        self.is_read = True
        self.read_at = datetime.utcnow()
        db.session.commit()
```

### Notification Functions

```python
# notifications.py
from models import Notification, NotificationPreference, User, db
from flask import url_for

def create_notification(user_id, notification_type, title, message, 
                       related_type=None, related_id=None, action_url=None):
    """Create in-app notification."""
    notification = Notification(
        user_id=user_id,
        type=notification_type,
        title=title,
        message=message,
        related_type=related_type,
        related_id=related_id,
        action_url=action_url
    )
    
    db.session.add(notification)
    db.session.commit()
    
    # Check if user wants email notification
    prefs = NotificationPreference.query.filter_by(user_id=user_id).first()
    
    if prefs and should_send_email(prefs, notification_type):
        send_notification_email(user_id, notification)
    
    return notification

def should_send_email(prefs, notification_type):
    """Check if email should be sent based on preferences."""
    if not prefs.email_notifications:
        return False
    
    type_mapping = {
        'comment': prefs.notify_comment,
        'mention': prefs.notify_mention,
        'follow': prefs.notify_new_follower
    }
    
    return type_mapping.get(notification_type, False)

def notify_new_comment(post_id, commenter_username):
    """Notify post author of new comment."""
    from models import Post
    
    post = Post.query.get(post_id)
    if not post:
        return
    
    # Don't notify if commenting on own post
    if post.user_id == commenter_username:
        return
    
    create_notification(
        user_id=post.user_id,
        notification_type='comment',
        title='New Comment',
        message=f'{commenter_username} commented on your post "{post.title}"',
        related_type='post',
        related_id=post_id,
        action_url=url_for('post_detail', post_id=post_id)
    )

def notify_new_follower(user_id, follower_id):
    """Notify user of new follower."""
    follower = User.query.get(follower_id)
    
    create_notification(
        user_id=user_id,
        notification_type='follow',
        title='New Follower',
        message=f'{follower.username} started following you',
        related_type='user',
        related_id=follower_id,
        action_url=url_for('user_profile', user_id=follower_id)
    )

def notify_mention(mentioned_user_id, post_id, mentioner_username):
    """Notify user when mentioned in post."""
    from models import Post
    
    post = Post.query.get(post_id)
    
    create_notification(
        user_id=mentioned_user_id,
        notification_type='mention',
        title='You were mentioned',
        message=f'{mentioner_username} mentioned you in "{post.title}"',
        related_type='post',
        related_id=post_id,
        action_url=url_for('post_detail', post_id=post_id)
    )

def get_unread_notifications(user_id, limit=10):
    """Get unread notifications for user."""
    return Notification.query.filter_by(
        user_id=user_id,
        is_read=False
    ).order_by(Notification.created_at.desc()).limit(limit).all()

def mark_all_as_read(user_id):
    """Mark all notifications as read."""
    Notification.query.filter_by(user_id=user_id, is_read=False).update({
        'is_read': True,
        'read_at': datetime.utcnow()
    })
    db.session.commit()
```

---

## Background Tasks with Celery

### Celery Setup

```bash
pip install celery redis
```

```python
# celery_app.py
from celery import Celery
from app import app

def make_celery(app):
    celery = Celery(
        app.import_name,
        backend=app.config['CELERY_RESULT_BACKEND'],
        broker=app.config['CELERY_BROKER_URL']
    )
    celery.conf.update(app.config)
    
    class ContextTask(celery.Task):
        def __call__(self, *args, **kwargs):
            with app.app_context():
                return self.run(*args, **kwargs)
    
    celery.Task = ContextTask
    return celery

# Configuration
app.config['CELERY_BROKER_URL'] = 'redis://localhost:6379/0'
app.config['CELERY_RESULT_BACKEND'] = 'redis://localhost:6379/0'

celery = make_celery(app)
```

### Async Email Tasks

```python
# tasks.py
from celery_app import celery
from flask_mail import Message
from app import mail

@celery.task
def send_async_email(subject, recipients, text_body, html_body):
    """Send email asynchronously."""
    msg = Message(
        subject=subject,
        recipients=recipients,
        body=text_body,
        html=html_body
    )
    mail.send(msg)

@celery.task
def send_bulk_emails(user_ids, subject, template):
    """Send bulk emails to multiple users."""
    from models import User
    
    users = User.query.filter(User.id.in_(user_ids)).all()
    
    for user in users:
        html_body = render_template(template, user=user)
        send_async_email.delay(
            subject=subject,
            recipients=[user.email],
            text_body='',
            html_body=html_body
        )

@celery.task
def send_daily_digest():
    """Send daily notification digest."""
    from models import User, Notification
    
    users = User.query.filter_by(is_active=True).all()
    
    for user in users:
        prefs = user.notification_prefs
        
        if not prefs or prefs.email_digest != 'daily':
            continue
        
        # Get unread notifications
        notifications = Notification.query.filter_by(
            user_id=user.id,
            is_read=False
        ).filter(
            Notification.created_at >= datetime.utcnow() - timedelta(days=1)
        ).all()
        
        if notifications:
            html_body = render_template(
                'email/daily_digest.html',
                user=user,
                notifications=notifications
            )
            
            send_async_email.delay(
                subject='Your Daily Digest',
                recipients=[user.email],
                text_body='',
                html_body=html_body
            )

# Usage in routes
@app.route('/contact', methods=['POST'])
def contact():
    data = request.get_json()
    
    # Send email asynchronously
    send_async_email.delay(
        subject='New Contact Form Submission',
        recipients=['admin@example.com'],
        text_body=data['message'],
        html_body=render_template('email/contact.html', data=data)
    )
    
    return jsonify({'status': 'Message sent'})
```

### Running Celery Worker

```bash
# Start Celery worker
celery -A celery_app.celery worker --loglevel=info

# Start Celery beat for scheduled tasks
celery -A celery_app.celery beat --loglevel=info

# Combined
celery -A celery_app.celery worker --beat --loglevel=info
```

### Scheduled Tasks

```python
# celery_app.py
from celery.schedules import crontab

celery.conf.beat_schedule = {
    'send-daily-digest': {
        'task': 'tasks.send_daily_digest',
        'schedule': crontab(hour=8, minute=0),  # Every day at 8 AM
    },
    'check-subscription-expiry': {
        'task': 'tasks.check_subscription_expiry',
        'schedule': crontab(hour=9, minute=0),  # Every day at 9 AM
    },
    'cleanup-old-notifications': {
        'task': 'tasks.cleanup_old_notifications',
        'schedule': crontab(hour=2, minute=0, day_of_week=1),  # Every Monday at 2 AM
    }
}

# tasks.py
@celery.task
def check_subscription_expiry():
    """Check and notify expiring subscriptions."""
    from models import Subscription
    from notifications import send_expiry_notification
    
    # Find subscriptions expiring in 3 days
    three_days = datetime.utcnow() + timedelta(days=3)
    
    expiring = Subscription.query.filter(
        Subscription.status == 'active',
        Subscription.expires_at <= three_days,
        Subscription.expires_at > datetime.utcnow()
    ).all()
    
    for subscription in expiring:
        send_expiry_notification(subscription.user, subscription.days_remaining())

@celery.task
def cleanup_old_notifications():
    """Delete read notifications older than 30 days."""
    from models import Notification
    
    cutoff = datetime.utcnow() - timedelta(days=30)
    
    deleted = Notification.query.filter(
        Notification.is_read == True,
        Notification.read_at < cutoff
    ).delete()
    
    db.session.commit()
    return deleted
```

---

## Real-time Notifications with WebSockets

### Flask-SocketIO Setup

```bash
pip install flask-socketio python-socketio
```

```python
# app.py
from flask import Flask
from flask_socketio import SocketIO, emit, join_room, leave_room

app = Flask(__name__)
socketio = SocketIO(app, cors_allowed_origins="*")

@socketio.on('connect')
def handle_connect():
    """Handle client connection."""
    if current_user.is_authenticated:
        join_room(f'user_{current_user.id}')
        emit('connected', {'user_id': current_user.id})

@socketio.on('disconnect')
def handle_disconnect():
    """Handle client disconnection."""
    if current_user.is_authenticated:
        leave_room(f'user_{current_user.id}')

def send_realtime_notification(user_id, notification_data):
    """Send real-time notification to specific user."""
    socketio.emit('notification', notification_data, room=f'user_{user_id}')

def broadcast_notification(notification_data):
    """Broadcast notification to all connected users."""
    socketio.emit('notification', notification_data, broadcast=True)

# Usage in other parts of application
@app.route('/api/comment', methods=['POST'])
@login_required
def create_comment():
    data = request.get_json()
    
    comment = Comment(**data, user_id=current_user.id)
    db.session.add(comment)
    db.session.commit()
    
    # Send real-time notification to post author
    post = Post.query.get(data['post_id'])
    if post.user_id != current_user.id:
        send_realtime_notification(post.user_id, {
            'type': 'new_comment',
            'message': f'{current_user.username} commented on your post',
            'url': url_for('post_detail', post_id=post.id)
        })
    
    return jsonify(comment.to_dict())
```

### Client-Side WebSocket

```html
<!-- templates/base.html -->
<script src="https://cdn.socket.io/4.5.4/socket.io.min.js"></script>
<script>
    const socket = io();
    
    socket.on('connect', function() {
        console.log('Connected to notification service');
    });
    
    socket.on('notification', function(data) {
        showNotification(data.type, data.message, data.url);
    });
    
    function showNotification(type, message, url) {
        // Show browser notification
        if (Notification.permission === 'granted') {
            new Notification('New Notification', {
                body: message,
                icon: '/static/icon.png'
            });
        }
        
        // Show in-app notification
        const notifDiv = document.createElement('div');
        notifDiv.className = `notification notification-${type}`;
        notifDiv.innerHTML = `
            <p>${message}</p>
            ${url ? `<a href="${url}">View</a>` : ''}
        `;
        document.getElementById('notifications').appendChild(notifDiv);
        
        // Update notification badge
        updateNotificationBadge();
    }
    
    function updateNotificationBadge() {
        fetch('/api/notifications/unread-count')
            .then(r => r.json())
            .then(data => {
                const badge = document.getElementById('notification-badge');
                badge.textContent = data.count;
                badge.style.display = data.count > 0 ? 'block' : 'none';
            });
    }
</script>
```

---

## Notification Routes

```python
# routes/notifications.py
from flask import Blueprint, jsonify, render_template
from flask_login import login_required, current_user
from models import Notification, db

notifications_bp = Blueprint('notifications', __name__, url_prefix='/notifications')

@notifications_bp.route('/')
@login_required
def list_notifications():
    """List all user notifications."""
    page = request.args.get('page', 1, type=int)
    
    notifications = Notification.query.filter_by(user_id=current_user.id)\
        .order_by(Notification.created_at.desc())\
        .paginate(page=page, per_page=20)
    
    return render_template('notifications/list.html', notifications=notifications)

@notifications_bp.route('/api/unread-count')
@login_required
def unread_count():
    """Get count of unread notifications."""
    count = Notification.query.filter_by(
        user_id=current_user.id,
        is_read=False
    ).count()
    
    return jsonify({'count': count})

@notifications_bp.route('/<int:notification_id>/read', methods=['POST'])
@login_required
def mark_as_read(notification_id):
    """Mark notification as read."""
    notification = Notification.query.get_or_404(notification_id)
    
    if notification.user_id != current_user.id:
        abort(403)
    
    notification.mark_as_read()
    
    return jsonify({'status': 'success'})

@notifications_bp.route('/mark-all-read', methods=['POST'])
@login_required
def mark_all_read():
    """Mark all notifications as read."""
    Notification.query.filter_by(
        user_id=current_user.id,
        is_read=False
    ).update({
        'is_read': True,
        'read_at': datetime.utcnow()
    })
    
    db.session.commit()
    
    return jsonify({'status': 'success'})

@notifications_bp.route('/<int:notification_id>/delete', methods=['DELETE'])
@login_required
def delete_notification(notification_id):
    """Delete a notification."""
    notification = Notification.query.get_or_404(notification_id)
    
    if notification.user_id != current_user.id:
        abort(403)
    
    db.session.delete(notification)
    db.session.commit()
    
    return jsonify({'status': 'deleted'})
```

---

## Webhook Integration

### Webhook Handler

```python
# webhooks.py
from flask import Blueprint, request, jsonify
import hmac
import hashlib

webhooks_bp = Blueprint('webhooks', __name__, url_prefix='/webhooks')

def verify_webhook_signature(payload, signature, secret):
    """Verify webhook signature."""
    expected_signature = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(signature, expected_signature)

@webhooks_bp.route('/stripe', methods=['POST'])
def stripe_webhook():
    """Handle Stripe webhook events."""
    payload = request.get_data()
    signature = request.headers.get('Stripe-Signature')
    
    # Verify signature
    if not verify_webhook_signature(payload, signature, app.config['STRIPE_WEBHOOK_SECRET']):
        return jsonify({'error': 'Invalid signature'}), 400
    
    event = request.get_json()
    
    # Handle different event types
    if event['type'] == 'customer.subscription.created':
        handle_subscription_created(event['data']['object'])
    
    elif event['type'] == 'customer.subscription.updated':
        handle_subscription_updated(event['data']['object'])
    
    elif event['type'] == 'customer.subscription.deleted':
        handle_subscription_cancelled(event['data']['object'])
    
    elif event['type'] == 'invoice.payment_succeeded':
        handle_payment_success(event['data']['object'])
    
    elif event['type'] == 'invoice.payment_failed':
        handle_payment_failed(event['data']['object'])
    
    return jsonify({'status': 'success'})

def handle_subscription_created(subscription_data):
    """Handle new subscription creation."""
    stripe_id = subscription_data['id']
    customer_id = subscription_data['customer']
    
    # Find user by Stripe customer ID
    user = User.query.filter_by(stripe_customer_id=customer_id).first()
    
    if user:
        sub = Subscription(
            user_id=user.id,
            subscription_type='premium',
            status='active',
            stripe_subscription_id=stripe_id,
            started_at=datetime.utcnow(),
            expires_at=datetime.fromtimestamp(subscription_data['current_period_end'])
        )
        db.session.add(sub)
        db.session.commit()
        
        # Send welcome email
        send_subscription_welcome_email(user)

def handle_payment_failed(invoice_data):
    """Handle failed payment."""
    subscription_id = invoice_data['subscription']
    
    subscription = Subscription.query.filter_by(
        stripe_subscription_id=subscription_id
    ).first()
    
    if subscription:
        # Notify user
        send_payment_failed_email(subscription.user, invoice_data['amount_due'])
```

---

## Subscription Routes

```python
# routes/subscriptions.py
from flask import Blueprint, render_template, request, jsonify
from flask_login import login_required, current_user
from models import Subscription, db

subscriptions_bp = Blueprint('subscriptions', __name__, url_prefix='/subscriptions')

@subscriptions_bp.route('/')
@login_required
def index():
    """Subscription management page."""
    subscription = Subscription.query.filter_by(user_id=current_user.id).first()
    
    return render_template('subscriptions/index.html', subscription=subscription)

@subscriptions_bp.route('/upgrade', methods=['POST'])
@login_required
def upgrade():
    """Upgrade subscription."""
    data = request.get_json()
    plan = data.get('plan')  # 'premium' or 'enterprise'
    
    subscription = upgrade_subscription(current_user.id, plan)
    
    return jsonify({
        'status': 'success',
        'subscription': subscription.to_dict()
    })

@subscriptions_bp.route('/cancel', methods=['POST'])
@login_required
def cancel():
    """Cancel subscription."""
    success = cancel_subscription(current_user.id)
    
    if success:
        return jsonify({'status': 'Subscription cancelled'})
    
    return jsonify({'error': 'No active subscription'}), 400

@subscriptions_bp.route('/preferences', methods=['GET', 'POST'])
@login_required
def preferences():
    """Manage notification preferences."""
    prefs = current_user.notification_prefs
    
    if not prefs:
        prefs = NotificationPreference(user_id=current_user.id)
        db.session.add(prefs)
    
    if request.method == 'POST':
        data = request.get_json()
        
        prefs.email_notifications = data.get('email_notifications', True)
        prefs.email_marketing = data.get('email_marketing', False)
        prefs.email_digest = data.get('email_digest', 'daily')
        prefs.notify_comment = data.get('notify_comment', True)
        prefs.notify_mention = data.get('notify_mention', True)
        prefs.notify_new_follower = data.get('notify_new_follower', True)
        
        db.session.commit()
        
        return jsonify({'status': 'Preferences updated'})
    
    return render_template('subscriptions/preferences.html', prefs=prefs)
```

---

## Event-Driven Notifications

### Event System

```python
# events.py
from blinker import Namespace

# Create namespace for app signals
signals = Namespace()

# Define signals
user_registered = signals.signal('user-registered')
post_created = signals.signal('post-created')
comment_created = signals.signal('comment-created')
user_followed = signals.signal('user-followed')

# Connect signal handlers
@user_registered.connect
def on_user_registered(sender, user):
    """Handle user registration event."""
    send_welcome_email(user)
    create_notification(
        user_id=user.id,
        notification_type='welcome',
        title='Welcome!',
        message='Welcome to our platform. Get started by exploring features.'
    )

@post_created.connect
def on_post_created(sender, post):
    """Handle post creation event."""
    # Notify followers
    followers = post.author.followers.all()
    
    for follower in followers:
        create_notification(
            user_id=follower.id,
            notification_type='new_post',
            title='New Post',
            message=f'{post.author.username} published "{post.title}"',
            related_type='post',
            related_id=post.id,
            action_url=url_for('post_detail', post_id=post.id)
        )

@comment_created.connect
def on_comment_created(sender, comment):
    """Handle comment creation event."""
    notify_new_comment(comment.post_id, comment.author.username)

# Usage in routes
@app.route('/register', methods=['POST'])
def register():
    user = User(**data)
    db.session.add(user)
    db.session.commit()
    
    # Trigger event
    user_registered.send(app, user=user)
    
    return jsonify({'status': 'registered'})
```

---

## Notification Batching

### Batch Notifications

```python
from collections import defaultdict

def batch_notifications(user_ids, notification_type, title_template, message_template):
    """Create notifications in batch."""
    notifications = []
    
    for user_id in user_ids:
        notif = Notification(
            user_id=user_id,
            type=notification_type,
            title=title_template.format(user_id=user_id),
            message=message_template.format(user_id=user_id)
        )
        notifications.append(notif)
    
    db.session.bulk_save_objects(notifications)
    db.session.commit()

def aggregate_notifications(user_id, hours=24):
    """Aggregate similar notifications."""
    cutoff = datetime.utcnow() - timedelta(hours=hours)
    
    notifications = Notification.query.filter(
        Notification.user_id == user_id,
        Notification.is_read == False,
        Notification.created_at >= cutoff
    ).all()
    
    # Group by type
    grouped = defaultdict(list)
    for notif in notifications:
        grouped[notif.type].append(notif)
    
    aggregated = []
    for notif_type, notifs in grouped.items():
        if len(notifs) > 1:
            # Create summary notification
            aggregated.append({
                'type': notif_type,
                'count': len(notifs),
                'message': f'You have {len(notifs)} new {notif_type} notifications',
                'items': notifs
            })
        else:
            aggregated.append(notifs[0])
    
    return aggregated
```

---

## Email Digest System

```python
# digest.py
from datetime import datetime, timedelta
from flask import render_template

def generate_daily_digest(user_id):
    """Generate daily digest for user."""
    from models import User, Notification, Post
    
    user = User.query.get(user_id)
    yesterday = datetime.utcnow() - timedelta(days=1)
    
    # Get recent notifications
    notifications = Notification.query.filter(
        Notification.user_id == user_id,
        Notification.created_at >= yesterday
    ).all()
    
    # Get popular posts
    popular_posts = Post.query.filter(
        Post.published_at >= yesterday
    ).order_by(Post.view_count.desc()).limit(5).all()
    
    # Get user's stats
    stats = {
        'new_followers': user.followers.filter(
            Follow.created_at >= yesterday
        ).count(),
        'new_comments': Comment.query.filter(
            Comment.post_id.in_([p.id for p in user.posts]),
            Comment.created_at >= yesterday
        ).count()
    }
    
    digest_data = {
        'user': user,
        'notifications': notifications,
        'popular_posts': popular_posts,
        'stats': stats,
        'date': datetime.utcnow().strftime('%B %d, %Y')
    }
    
    return digest_data

@celery.task
def send_digests(frequency='daily'):
    """Send digest emails to users."""
    from models import User, NotificationPreference
    
    # Get users who want this frequency
    users = User.query.join(NotificationPreference).filter(
        NotificationPreference.email_digest == frequency,
        User.is_active == True
    ).all()
    
    for user in users:
        digest_data = generate_daily_digest(user.id)
        
        html_body = render_template(
            'email/digest.html',
            **digest_data
        )
        
        send_async_email.delay(
            subject=f'Your {frequency.capitalize()} Digest',
            recipients=[user.email],
            text_body='',
            html_body=html_body
        )
```

---

## SMS Notifications (Twilio)

```bash
pip install twilio
```

```python
# sms.py
from twilio.rest import Client

class SMSNotifier:
    def __init__(self, account_sid, auth_token, from_number):
        self.client = Client(account_sid, auth_token)
        self.from_number = from_number
    
    def send_sms(self, to_number, message):
        """Send SMS notification."""
        try:
            message = self.client.messages.create(
                body=message,
                from_=self.from_number,
                to=to_number
            )
            return message.sid
        except Exception as e:
            print(f"SMS send failed: {e}")
            return None

# Usage
sms_notifier = SMSNotifier(
    account_sid=app.config['TWILIO_ACCOUNT_SID'],
    auth_token=app.config['TWILIO_AUTH_TOKEN'],
    from_number=app.config['TWILIO_PHONE_NUMBER']
)

def send_verification_code(user):
    """Send SMS verification code."""
    import random
    
    code = f"{random.randint(100000, 999999)}"
    
    # Store code in cache
    cache.set(f'sms_code:{user.id}', code, timeout=300)
    
    # Send SMS
    sms_notifier.send_sms(
        user.phone_number,
        f'Your verification code is: {code}'
    )
```

---

## Best Practices

### Notification Best Practices

1. **Rate limiting** - Don't spam users with notifications
2. **Batching** - Group similar notifications together
3. **User preferences** - Let users control what they receive
4. **Unsubscribe option** - Always provide easy unsubscribe
5. **Clear CTAs** - Include clear action buttons/links
6. **Mobile-friendly** - Ensure emails look good on mobile
7. **Test emails** - Always test before sending to users
8. **Monitor delivery** - Track open rates and bounces
9. **Async processing** - Use background tasks for email
10. **Fallback options** - Have backup notification methods

### Email Template Best Practices

```jinja2
{# Good email template structure #}
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ subject }}</title>
    <style>
        /* Inline CSS for email compatibility */
        body { margin: 0; padding: 0; font-family: Arial, sans-serif; }
        .container { max-width: 600px; margin: 0 auto; }
        /* Mobile responsive */
        @media only screen and (max-width: 600px) {
            .container { width: 100% !important; }
        }
    </style>
</head>
<body>
    <div class="container">
        {% block content %}{% endblock %}
    </div>
</body>
</html>
```

---

## Complete Example: Newsletter System

```python
# newsletter.py
from models import User, Newsletter, db
from tasks import send_async_email

class NewsletterManager:
    """Manage newsletter subscriptions and sending."""
    
    @staticmethod
    def subscribe(email, name=None):
        """Subscribe to newsletter."""
        existing = Newsletter.query.filter_by(email=email).first()
        
        if existing:
            if not existing.is_active:
                existing.is_active = True
                existing.subscribed_at = datetime.utcnow()
                db.session.commit()
            return existing, False
        
        subscriber = Newsletter(email=email, name=name)
        db.session.add(subscriber)
        db.session.commit()
        
        # Send confirmation email
        send_subscription_confirmation(subscriber)
        
        return subscriber, True
    
    @staticmethod
    def unsubscribe(token):
        """Unsubscribe from newsletter."""
        subscriber = Newsletter.query.filter_by(unsubscribe_token=token).first()
        
        if subscriber:
            subscriber.is_active = False
            subscriber.unsubscribed_at = datetime.utcnow()
            db.session.commit()
            return True
        
        return False
    
    @staticmethod
    def send_newsletter(subject, content):
        """Send newsletter to all active subscribers."""
        subscribers = Newsletter.query.filter_by(is_active=True).all()
        
        for subscriber in subscribers:
            html_body = render_template(
                'email/newsletter.html',
                subscriber=subscriber,
                content=content,
                unsubscribe_url=url_for('unsubscribe', 
                                       token=subscriber.unsubscribe_token,
                                       _external=True)
            )
            
            send_async_email.delay(
                subject=subject,
                recipients=[subscriber.email],
                text_body='',
                html_body=html_body
            )
        
        return len(subscribers)

# Routes
@app.route('/newsletter/subscribe', methods=['POST'])
def newsletter_subscribe():
    data = request.get_json()
    subscriber, created = NewsletterManager.subscribe(
        email=data['email'],
        name=data.get('name')
    )
    
    return jsonify({
        'status': 'subscribed' if created else 'already_subscribed',
        'email': subscriber.email
    })

@app.route('/newsletter/unsubscribe/<token>')
def unsubscribe(token):
    success = NewsletterManager.unsubscribe(token)
    
    if success:
        flash('You have been unsubscribed from our newsletter.', 'info')
    else:
        flash('Invalid unsubscribe link.', 'error')
    
    return render_template('unsubscribed.html')
```

---

## Related Documentation

- [Flask-Mail Documentation](https://flask-mail.readthedocs.io/)
- [Celery Documentation](https://docs.celeryproject.org/)
- [Flask-SocketIO Documentation](https://flask-socketio.readthedocs.io/)
- [Twilio Python SDK](https://www.twilio.com/docs/libraries/python)
