# User Authentication in Flask

Complete guide to implementing secure user authentication in Flask applications.

## Table of Contents
- [Flask-Login Setup](#flask-login-setup)
- [User Model](#user-model)
- [Password Hashing](#password-hashing)
- [Login/Logout/Registration](#login-logout-registration)
- [Session Management](#session-management)
- [Protected Routes](#protected-routes)
- [Remember Me](#remember-me)
- [Password Reset](#password-reset)

---

## Flask-Login Setup

### Installation

```bash
pip install flask-login werkzeug
```

### Basic Configuration

```python
# app.py
from flask import Flask
from flask_login import LoginManager

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key-change-in-production'

login_manager = LoginManager()
login_manager.init_app(app)
login_manager.login_view = 'login'  # Redirect unauthorized users
login_manager.login_message = 'Please log in to access this page.'
login_manager.login_message_category = 'info'

@login_manager.user_loader
def load_user(user_id):
    """Load user by ID for Flask-Login."""
    from models import User
    return User.query.get(int(user_id))

@login_manager.unauthorized_handler
def unauthorized():
    """Handle unauthorized access."""
    flash('You must be logged in to view this page.', 'warning')
    return redirect(url_for('login', next=request.url))
```

---

## User Model

### User Model with Authentication

```python
# models.py
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash
from datetime import datetime
from app import db

class User(UserMixin, db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False, index=True)
    email = db.Column(db.String(120), unique=True, nullable=False, index=True)
    password_hash = db.Column(db.String(256), nullable=False)
    
    # Authentication fields
    is_active = db.Column(db.Boolean, default=True)
    is_admin = db.Column(db.Boolean, default=False)
    confirmed = db.Column(db.Boolean, default=False)
    confirmed_on = db.Column(db.DateTime)
    
    # Timestamps
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    last_login = db.Column(db.DateTime)
    last_seen = db.Column(db.DateTime)
    
    # Password reset
    reset_token = db.Column(db.String(100), unique=True)
    reset_token_expires = db.Column(db.DateTime)
    
    def set_password(self, password):
        """Hash and set password."""
        self.password_hash = generate_password_hash(password)
    
    def check_password(self, password):
        """Check if password matches hash."""
        return check_password_hash(self.password_hash, password)
    
    def get_id(self):
        """Return user ID as string (required by Flask-Login)."""
        return str(self.id)
    
    @property
    def is_authenticated(self):
        """Return True if user is authenticated."""
        return True
    
    @property
    def is_anonymous(self):
        """Return False (not an anonymous user)."""
        return False
    
    def __repr__(self):
        return f'<User {self.username}>'
```

---

## Password Hashing

### Secure Password Management

```python
from werkzeug.security import generate_password_hash, check_password_hash

def create_user_secure(username, email, password):
    """Create user with hashed password."""
    
    # Validate password strength
    if len(password) < 8:
        raise ValueError("Password must be at least 8 characters")
    
    user = User(
        username=username,
        email=email,
        password_hash=generate_password_hash(
            password,
            method='pbkdf2:sha256',
            salt_length=16
        )
    )
    
    db.session.add(user)
    db.session.commit()
    
    return user

def verify_password(user, password):
    """Verify user password."""
    return check_password_hash(user.password_hash, password)

def update_password(user, old_password, new_password):
    """Update user password after verification."""
    
    # Verify old password
    if not user.check_password(old_password):
        raise ValueError("Current password is incorrect")
    
    # Validate new password
    if len(new_password) < 8:
        raise ValueError("New password must be at least 8 characters")
    
    # Update password
    user.set_password(new_password)
    db.session.commit()
    
    return True
```

---

## Login/Logout/Registration

### Registration

```python
# routes/auth.py
from flask import Blueprint, render_template, request, flash, redirect, url_for
from flask_login import login_user, logout_user, login_required, current_user
from models import User, db
from forms import RegistrationForm, LoginForm

auth_bp = Blueprint('auth', __name__)

@auth_bp.route('/register', methods=['GET', 'POST'])
def register():
    """User registration."""
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    
    form = RegistrationForm()
    
    if form.validate_on_submit():
        # Check if user exists
        if User.query.filter_by(username=form.username.data).first():
            flash('Username already taken.', 'danger')
            return render_template('auth/register.html', form=form)
        
        if User.query.filter_by(email=form.email.data).first():
            flash('Email already registered.', 'danger')
            return render_template('auth/register.html', form=form)
        
        # Create user
        user = User(
            username=form.username.data,
            email=form.email.data
        )
        user.set_password(form.password.data)
        
        db.session.add(user)
        db.session.commit()
        
        flash('Registration successful! Please log in.', 'success')
        return redirect(url_for('auth.login'))
    
    return render_template('auth/register.html', form=form)
```

### Login

```python
@auth_bp.route('/login', methods=['GET', 'POST'])
def login():
    """User login."""
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    
    form = LoginForm()
    
    if form.validate_on_submit():
        # Find user by username or email
        user = User.query.filter(
            db.or_(
                User.username == form.username.data,
                User.email == form.username.data
            )
        ).first()
        
        # Verify user and password
        if user is None or not user.check_password(form.password.data):
            flash('Invalid username or password.', 'danger')
            return render_template('auth/login.html', form=form)
        
        # Check if user is active
        if not user.is_active:
            flash('Your account has been deactivated.', 'warning')
            return render_template('auth/login.html', form=form)
        
        # Log in user
        login_user(user, remember=form.remember.data)
        
        # Update last login
        user.last_login = datetime.utcnow()
        db.session.commit()
        
        # Redirect to next page or home
        next_page = request.args.get('next')
        if not next_page or not next_page.startswith('/'):
            next_page = url_for('index')
        
        flash(f'Welcome back, {user.username}!', 'success')
        return redirect(next_page)
    
    return render_template('auth/login.html', form=form)
```

### Logout

```python
@auth_bp.route('/logout')
@login_required
def logout():
    """User logout."""
    username = current_user.username
    logout_user()
    flash(f'Goodbye, {username}!', 'info')
    return redirect(url_for('index'))
```

---

## Protected Routes

### Using Login Required Decorator

```python
from flask_login import login_required, current_user

@app.route('/profile')
@login_required
def profile():
    """User profile (authentication required)."""
    return render_template('profile.html', user=current_user)

@app.route('/admin')
@login_required
def admin_panel():
    """Admin panel (authentication + authorization)."""
    if not current_user.is_admin:
        flash('Access denied. Admin privileges required.', 'danger')
        abort(403)
    
    return render_template('admin/panel.html')
```

### Custom Authorization Decorators

```python
from functools import wraps
from flask import abort, flash, redirect, url_for

def admin_required(f):
    """Decorator for admin-only routes."""
    @wraps(f)
    @login_required
    def decorated_function(*args, **kwargs):
        if not current_user.is_admin:
            flash('Admin access required.', 'danger')
            abort(403)
        return f(*args, **kwargs)
    return decorated_function

def confirmed_required(f):
    """Decorator for confirmed users only."""
    @wraps(f)
    @login_required
    def decorated_function(*args, **kwargs):
        if not current_user.confirmed:
            flash('Please confirm your email address.', 'warning')
            return redirect(url_for('auth.unconfirmed'))
        return f(*args, **kwargs)
    return decorated_function

def permission_required(permission):
    """Decorator for permission-based access."""
    def decorator(f):
        @wraps(f)
        @login_required
        def decorated_function(*args, **kwargs):
            if not current_user.has_permission(permission):
                abort(403)
            return f(*args, **kwargs)
        return decorated_function
    return decorator

# Usage
@app.route('/admin/users')
@admin_required
def manage_users():
    users = User.query.all()
    return render_template('admin/users.html', users=users)

@app.route('/create-post')
@confirmed_required
def create_post():
    return render_template('create_post.html')

@app.route('/moderate')
@permission_required('moderate_content')
def moderate():
    return render_template('moderate.html')
```

---

## Session Management

### Session Configuration

```python
# app.py
from datetime import timedelta

app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(days=7)
app.config['SESSION_COOKIE_SECURE'] = True  # HTTPS only
app.config['SESSION_COOKIE_HTTPONLY'] = True  # No JavaScript access
app.config['SESSION_COOKIE_SAMESITE'] = 'Lax'  # CSRF protection

@app.before_request
def before_request():
    """Update user activity before each request."""
    if current_user.is_authenticated:
        current_user.last_seen = datetime.utcnow()
        db.session.commit()
```

### Session Data Management

```python
from flask import session

@app.route('/set-preference')
@login_required
def set_preference():
    """Store user preferences in session."""
    session['theme'] = request.args.get('theme', 'light')
    session['language'] = request.args.get('lang', 'en')
    session.permanent = True  # Use PERMANENT_SESSION_LIFETIME
    
    return jsonify({'status': 'Preferences saved'})

@app.route('/get-preference')
@login_required
def get_preference():
    """Get user preferences from session."""
    theme = session.get('theme', 'light')
    language = session.get('language', 'en')
    
    return jsonify({'theme': theme, 'language': language})
```

---

## Remember Me Functionality

### Implementing Remember Me

```python
# The remember parameter in login_user handles this automatically
@auth_bp.route('/login', methods=['POST'])
def login():
    form = LoginForm()
    
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.username.data).first()
        
        if user and user.check_password(form.password.data):
            # Remember me for 30 days if checked
            login_user(user, remember=form.remember.data, duration=timedelta(days=30))
            return redirect(url_for('index'))
    
    return render_template('auth/login.html', form=form)

# Custom remember me implementation
class User(UserMixin, db.Model):
    remember_token = db.Column(db.String(100), unique=True)
    remember_token_expires = db.Column(db.DateTime)
    
    def generate_remember_token(self):
        """Generate remember me token."""
        import secrets
        self.remember_token = secrets.token_urlsafe(32)
        self.remember_token_expires = datetime.utcnow() + timedelta(days=30)
        db.session.commit()
        return self.remember_token
    
    @staticmethod
    def verify_remember_token(token):
        """Verify remember token."""
        user = User.query.filter_by(remember_token=token).first()
        
        if user and user.remember_token_expires > datetime.utcnow():
            return user
        
        return None
```

---

## Password Reset

### Password Reset Flow

```python
import secrets
from datetime import datetime, timedelta

def generate_reset_token(user):
    """Generate password reset token."""
    user.reset_token = secrets.token_urlsafe(32)
    user.reset_token_expires = datetime.utcnow() + timedelta(hours=1)
    db.session.commit()
    return user.reset_token

def verify_reset_token(token):
    """Verify password reset token."""
    user = User.query.filter_by(reset_token=token).first()
    
    if user and user.reset_token_expires > datetime.utcnow():
        return user
    
    return None

@auth_bp.route('/forgot-password', methods=['GET', 'POST'])
def forgot_password():
    """Request password reset."""
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    
    form = ForgotPasswordForm()
    
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        
        if user:
            token = generate_reset_token(user)
            
            # Send email with reset link
            reset_url = url_for('auth.reset_password', token=token, _external=True)
            send_password_reset_email(user.email, reset_url)
            
        # Always show success message (security best practice)
        flash('If an account exists, a password reset link has been sent.', 'info')
        return redirect(url_for('auth.login'))
    
    return render_template('auth/forgot_password.html', form=form)

@auth_bp.route('/reset-password/<token>', methods=['GET', 'POST'])
def reset_password(token):
    """Reset password with token."""
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    
    user = verify_reset_token(token)
    
    if not user:
        flash('Invalid or expired reset link.', 'danger')
        return redirect(url_for('auth.forgot_password'))
    
    form = ResetPasswordForm()
    
    if form.validate_on_submit():
        user.set_password(form.password.data)
        user.reset_token = None
        user.reset_token_expires = None
        db.session.commit()
        
        flash('Your password has been reset. Please log in.', 'success')
        return redirect(url_for('auth.login'))
    
    return render_template('auth/reset_password.html', form=form)
```

---

## Forms

### Authentication Forms

```python
# forms.py
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, BooleanField, SubmitField
from wtforms.validators import DataRequired, Email, EqualTo, Length, ValidationError
from models import User

class RegistrationForm(FlaskForm):
    username = StringField('Username', 
        validators=[DataRequired(), Length(min=3, max=20)])
    email = StringField('Email', 
        validators=[DataRequired(), Email()])
    password = PasswordField('Password', 
        validators=[DataRequired(), Length(min=8)])
    password_confirm = PasswordField('Confirm Password',
        validators=[DataRequired(), EqualTo('password', message='Passwords must match')])
    submit = SubmitField('Register')
    
    def validate_username(self, field):
        """Check if username is taken."""
        if User.query.filter_by(username=field.data).first():
            raise ValidationError('Username already taken.')
    
    def validate_email(self, field):
        """Check if email is registered."""
        if User.query.filter_by(email=field.data).first():
            raise ValidationError('Email already registered.')

class LoginForm(FlaskForm):
    username = StringField('Username or Email', validators=[DataRequired()])
    password = PasswordField('Password', validators=[DataRequired()])
    remember = BooleanField('Remember Me')
    submit = SubmitField('Login')

class ChangePasswordForm(FlaskForm):
    current_password = PasswordField('Current Password', validators=[DataRequired()])
    new_password = PasswordField('New Password', 
        validators=[DataRequired(), Length(min=8)])
    confirm_password = PasswordField('Confirm New Password',
        validators=[DataRequired(), EqualTo('new_password')])
    submit = SubmitField('Change Password')

class ForgotPasswordForm(FlaskForm):
    email = StringField('Email', validators=[DataRequired(), Email()])
    submit = SubmitField('Send Reset Link')

class ResetPasswordForm(FlaskForm):
    password = PasswordField('New Password', 
        validators=[DataRequired(), Length(min=8)])
    password_confirm = PasswordField('Confirm Password',
        validators=[DataRequired(), EqualTo('password')])
    submit = SubmitField('Reset Password')
```

---

## Complete Authentication Routes

```python
# routes/auth.py
from flask import Blueprint, render_template, request, flash, redirect, url_for
from flask_login import login_user, logout_user, login_required, current_user
from models import User, db
from forms import RegistrationForm, LoginForm, ChangePasswordForm

auth_bp = Blueprint('auth', __name__, url_prefix='/auth')

@auth_bp.route('/register', methods=['GET', 'POST'])
def register():
    """User registration."""
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    
    form = RegistrationForm()
    
    if form.validate_on_submit():
        user = User(
            username=form.username.data,
            email=form.email.data
        )
        user.set_password(form.password.data)
        
        db.session.add(user)
        db.session.commit()
        
        # Optional: Send confirmation email
        send_confirmation_email(user)
        
        flash('Registration successful! Please check your email.', 'success')
        return redirect(url_for('auth.login'))
    
    return render_template('auth/register.html', form=form)

@auth_bp.route('/login', methods=['GET', 'POST'])
def login():
    """User login."""
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    
    form = LoginForm()
    
    if form.validate_on_submit():
        user = User.query.filter(
            db.or_(
                User.username == form.username.data,
                User.email == form.username.data
            )
        ).first()
        
        if user and user.check_password(form.password.data):
            if not user.is_active:
                flash('Your account has been deactivated.', 'warning')
                return render_template('auth/login.html', form=form)
            
            login_user(user, remember=form.remember.data)
            
            user.last_login = datetime.utcnow()
            db.session.commit()
            
            next_page = request.args.get('next')
            if not next_page or not next_page.startswith('/'):
                next_page = url_for('index')
            
            return redirect(next_page)
        else:
            flash('Invalid credentials.', 'danger')
    
    return render_template('auth/login.html', form=form)

@auth_bp.route('/logout')
@login_required
def logout():
    """User logout."""
    logout_user()
    flash('You have been logged out.', 'info')
    return redirect(url_for('index'))

@auth_bp.route('/change-password', methods=['GET', 'POST'])
@login_required
def change_password():
    """Change user password."""
    form = ChangePasswordForm()
    
    if form.validate_on_submit():
        if current_user.check_password(form.current_password.data):
            current_user.set_password(form.new_password.data)
            db.session.commit()
            
            flash('Password changed successfully.', 'success')
            return redirect(url_for('profile'))
        else:
            flash('Current password is incorrect.', 'danger')
    
    return render_template('auth/change_password.html', form=form)
```

---

## Email Confirmation

### Email Confirmation Flow

```python
from itsdangerous import URLSafeTimedSerializer

def generate_confirmation_token(email):
    """Generate email confirmation token."""
    serializer = URLSafeTimedSerializer(app.config['SECRET_KEY'])
    return serializer.dumps(email, salt='email-confirm')

def confirm_token(token, expiration=3600):
    """Verify email confirmation token."""
    serializer = URLSafeTimedSerializer(app.config['SECRET_KEY'])
    try:
        email = serializer.loads(
            token,
            salt='email-confirm',
            max_age=expiration
        )
        return email
    except:
        return None

@auth_bp.route('/confirm/<token>')
@login_required
def confirm_email(token):
    """Confirm user email."""
    if current_user.confirmed:
        flash('Account already confirmed.', 'info')
        return redirect(url_for('index'))
    
    email = confirm_token(token)
    
    if email and email == current_user.email:
        current_user.confirmed = True
        current_user.confirmed_on = datetime.utcnow()
        db.session.commit()
        
        flash('Email confirmed! Thank you.', 'success')
    else:
        flash('Invalid or expired confirmation link.', 'danger')
    
    return redirect(url_for('index'))

@auth_bp.route('/resend-confirmation')
@login_required
def resend_confirmation():
    """Resend confirmation email."""
    if current_user.confirmed:
        flash('Account already confirmed.', 'info')
        return redirect(url_for('index'))
    
    token = generate_confirmation_token(current_user.email)
    confirm_url = url_for('auth.confirm_email', token=token, _external=True)
    send_confirmation_email(current_user.email, confirm_url)
    
    flash('Confirmation email sent.', 'info')
    return redirect(url_for('index'))
```

---

## Two-Factor Authentication (2FA)

### Basic 2FA Implementation

```python
import pyotp
import qrcode
from io import BytesIO
import base64

# Add to User model
class User(UserMixin, db.Model):
    two_factor_secret = db.Column(db.String(32))
    two_factor_enabled = db.Column(db.Boolean, default=False)
    
    def generate_2fa_secret(self):
        """Generate 2FA secret."""
        self.two_factor_secret = pyotp.random_base32()
        db.session.commit()
        return self.two_factor_secret
    
    def get_2fa_uri(self):
        """Get 2FA provisioning URI."""
        return pyotp.totp.TOTP(self.two_factor_secret).provisioning_uri(
            name=self.email,
            issuer_name='Your App Name'
        )
    
    def verify_2fa_token(self, token):
        """Verify 2FA token."""
        if not self.two_factor_enabled:
            return True
        
        totp = pyotp.TOTP(self.two_factor_secret)
        return totp.verify(token)

@auth_bp.route('/enable-2fa')
@login_required
def enable_2fa():
    """Enable two-factor authentication."""
    if current_user.two_factor_enabled:
        flash('2FA is already enabled.', 'info')
        return redirect(url_for('profile'))
    
    # Generate secret
    secret = current_user.generate_2fa_secret()
    
    # Generate QR code
    uri = current_user.get_2fa_uri()
    qr = qrcode.make(uri)
    
    buffer = BytesIO()
    qr.save(buffer, format='PNG')
    qr_data = base64.b64encode(buffer.getvalue()).decode()
    
    return render_template('auth/enable_2fa.html', 
                         qr_code=qr_data,
                         secret=secret)

@auth_bp.route('/verify-2fa', methods=['POST'])
@login_required
def verify_2fa():
    """Verify and enable 2FA."""
    token = request.form.get('token')
    
    if current_user.verify_2fa_token(token):
        current_user.two_factor_enabled = True
        db.session.commit()
        
        flash('Two-factor authentication enabled!', 'success')
        return redirect(url_for('profile'))
    else:
        flash('Invalid token. Please try again.', 'danger')
        return redirect(url_for('auth.enable_2fa'))

@auth_bp.route('/login-2fa', methods=['GET', 'POST'])
def login_2fa():
    """Second step of login with 2FA."""
    user_id = session.get('pending_2fa_user_id')
    
    if not user_id:
        return redirect(url_for('auth.login'))
    
    user = User.query.get(user_id)
    
    if request.method == 'POST':
        token = request.form.get('token')
        
        if user.verify_2fa_token(token):
            session.pop('pending_2fa_user_id')
            login_user(user)
            
            flash('Login successful!', 'success')
            return redirect(url_for('index'))
        else:
            flash('Invalid 2FA token.', 'danger')
    
    return render_template('auth/login_2fa.html')
```

---

## OAuth Integration

### Google OAuth Example

```bash
pip install authlib
```

```python
# app.py
from authlib.integrations.flask_client import OAuth

oauth = OAuth(app)

google = oauth.register(
    name='google',
    client_id='YOUR_GOOGLE_CLIENT_ID',
    client_secret='YOUR_GOOGLE_CLIENT_SECRET',
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid email profile'}
)

@auth_bp.route('/login/google')
def google_login():
    """Initiate Google OAuth login."""
    redirect_uri = url_for('auth.google_callback', _external=True)
    return google.authorize_redirect(redirect_uri)

@auth_bp.route('/login/google/callback')
def google_callback():
    """Handle Google OAuth callback."""
    token = google.authorize_access_token()
    user_info = token.get('userinfo')
    
    if user_info:
        # Find or create user
        user = User.query.filter_by(email=user_info['email']).first()
        
        if not user:
            user = User(
                username=user_info['email'].split('@')[0],
                email=user_info['email'],
                confirmed=True,
                confirmed_on=datetime.utcnow()
            )
            # OAuth users don't have password
            user.set_password(secrets.token_urlsafe(32))
            db.session.add(user)
            db.session.commit()
        
        login_user(user)
        flash('Logged in successfully!', 'success')
        return redirect(url_for('index'))
    
    flash('Login failed.', 'danger')
    return redirect(url_for('auth.login'))
```

---

## Role-Based Access Control (RBAC)

### RBAC Implementation

```python
# models.py
class Role(db.Model):
    __tablename__ = 'roles'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), unique=True, nullable=False)
    description = db.Column(db.String(200))
    
    users = db.relationship('User', backref='role', lazy='dynamic')
    permissions = db.relationship('Permission', secondary='role_permissions', backref='roles')

class Permission(db.Model):
    __tablename__ = 'permissions'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), unique=True, nullable=False)
    description = db.Column(db.String(200))

role_permissions = db.Table('role_permissions',
    db.Column('role_id', db.Integer, db.ForeignKey('roles.id'), primary_key=True),
    db.Column('permission_id', db.Integer, db.ForeignKey('permissions.id'), primary_key=True)
)

class User(UserMixin, db.Model):
    # ... existing fields ...
    role_id = db.Column(db.Integer, db.ForeignKey('roles.id'))
    
    def has_permission(self, permission_name):
        """Check if user has specific permission."""
        if not self.role:
            return False
        
        return any(p.name == permission_name for p in self.role.permissions)
    
    def has_role(self, role_name):
        """Check if user has specific role."""
        return self.role and self.role.name == role_name

# Decorator for permission-based access
def permission_required(permission):
    def decorator(f):
        @wraps(f)
        @login_required
        def decorated_function(*args, **kwargs):
            if not current_user.has_permission(permission):
                flash('You do not have permission to access this page.', 'danger')
                abort(403)
            return f(*args, **kwargs)
        return decorated_function
    return decorator

# Usage
@app.route('/admin/delete-post/<int:post_id>', methods=['POST'])
@permission_required('delete_posts')
def delete_post(post_id):
    post = Post.query.get_or_404(post_id)
    db.session.delete(post)
    db.session.commit()
    return jsonify({'status': 'deleted'})
```

---

## Security Best Practices

### Account Lockout

```python
class User(UserMixin, db.Model):
    failed_login_attempts = db.Column(db.Integer, default=0)
    locked_until = db.Column(db.DateTime)
    
    def is_locked(self):
        """Check if account is locked."""
        if self.locked_until and self.locked_until > datetime.utcnow():
            return True
        return False
    
    def record_failed_login(self):
        """Record failed login attempt."""
        self.failed_login_attempts += 1
        
        if self.failed_login_attempts >= 5:
            # Lock for 15 minutes
            self.locked_until = datetime.utcnow() + timedelta(minutes=15)
        
        db.session.commit()
    
    def record_successful_login(self):
        """Reset failed login attempts."""
        self.failed_login_attempts = 0
        self.locked_until = None
        self.last_login = datetime.utcnow()
        db.session.commit()

# In login route
@auth_bp.route('/login', methods=['POST'])
def login():
    form = LoginForm()
    
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.username.data).first()
        
        if user:
            if user.is_locked():
                minutes_left = (user.locked_until - datetime.utcnow()).seconds // 60
                flash(f'Account locked. Try again in {minutes_left} minutes.', 'danger')
                return render_template('auth/login.html', form=form)
            
            if user.check_password(form.password.data):
                user.record_successful_login()
                login_user(user, remember=form.remember.data)
                return redirect(url_for('index'))
            else:
                user.record_failed_login()
                flash('Invalid password.', 'danger')
        else:
            flash('User not found.', 'danger')
    
    return render_template('auth/login.html', form=form)
```

### CSRF Protection

```python
# Already enabled with Flask-WTF
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect(app)

# For AJAX requests
<meta name="csrf-token" content="{{ csrf_token() }}">

# JavaScript
fetch('/api/endpoint', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': document.querySelector('meta[name="csrf-token"]').content
    },
    body: JSON.stringify(data)
});
```

### Secure Cookies

```python
# app.py
app.config['SESSION_COOKIE_SECURE'] = True       # HTTPS only
app.config['SESSION_COOKIE_HTTPONLY'] = True     # No JavaScript access
app.config['SESSION_COOKIE_SAMESITE'] = 'Lax'    # CSRF protection
app.config['REMEMBER_COOKIE_SECURE'] = True
app.config['REMEMBER_COOKIE_HTTPONLY'] = True
app.config['REMEMBER_COOKIE_DURATION'] = timedelta(days=30)
```

---

## Template Examples

### Login Template

```jinja2
{# templates/auth/login.html #}
{% extends "base.html" %}

{% block content %}
<div class="auth-container">
    <h2>Login</h2>
    
    <form method="POST" novalidate>
        {{ form.csrf_token }}
        
        <div class="form-group">
            {{ form.username.label }}
            {{ form.username(class="form-control", autofocus=true) }}
            {% if form.username.errors %}
                <div class="error">
                    {% for error in form.username.errors %}
                        <span>{{ error }}</span>
                    {% endfor %}
                </div>
            {% endif %}
        </div>
        
        <div class="form-group">
            {{ form.password.label }}
            {{ form.password(class="form-control") }}
            {% if form.password.errors %}
                <div class="error">
                    {% for error in form.password.errors %}
                        <span>{{ error }}</span>
                    {% endfor %}
                </div>
            {% endif %}
        </div>
        
        <div class="form-group">
            {{ form.remember() }}
            {{ form.remember.label }}
        </div>
        
        <button type="submit" class="btn btn-primary">{{ form.submit.label.text }}</button>
    </form>
    
    <div class="auth-links">
        <p>Don't have an account? <a href="{{ url_for('auth.register') }}">Register</a></p>
        <p><a href="{{ url_for('auth.forgot_password') }}">Forgot password?</a></p>
    </div>
    
    <div class="oauth-buttons">
        <a href="{{ url_for('auth.google_login') }}" class="btn btn-google">
            Login with Google
        </a>
    </div>
</div>
{% endblock %}
```

### Navigation with Authentication

```jinja2
{# templates/components/navbar.html #}
<nav class="navbar">
    <a href="{{ url_for('index') }}" class="brand">My App</a>
    
    <ul class="nav-links">
        <li><a href="{{ url_for('index') }}">Home</a></li>
        
        {% if current_user.is_authenticated %}
            <li><a href="{{ url_for('profile') }}">Profile</a></li>
            <li><a href="{{ url_for('dashboard') }}">Dashboard</a></li>
            
            {% if current_user.is_admin %}
                <li><a href="{{ url_for('admin.panel') }}">Admin</a></li>
            {% endif %}
            
            <li>
                <span>Hello, {{ current_user.username }}!</span>
                <a href="{{ url_for('auth.logout') }}">Logout</a>
            </li>
        {% else %}
            <li><a href="{{ url_for('auth.login') }}">Login</a></li>
            <li><a href="{{ url_for('auth.register') }}">Register</a></li>
        {% endif %}
    </ul>
</nav>
```

---

## API Authentication

### Token-Based Authentication

```python
import secrets
from datetime import datetime, timedelta

class User(UserMixin, db.Model):
    api_token = db.Column(db.String(64), unique=True, index=True)
    api_token_expires = db.Column(db.DateTime)
    
    def generate_api_token(self, expires_in=3600):
        """Generate API token."""
        self.api_token = secrets.token_urlsafe(32)
        self.api_token_expires = datetime.utcnow() + timedelta(seconds=expires_in)
        db.session.commit()
        return self.api_token
    
    def revoke_api_token(self):
        """Revoke API token."""
        self.api_token = None
        self.api_token_expires = None
        db.session.commit()
    
    @staticmethod
    def verify_api_token(token):
        """Verify API token."""
        user = User.query.filter_by(api_token=token).first()
        
        if user and user.api_token_expires > datetime.utcnow():
            return user
        
        return None

# API authentication decorator
from functools import wraps

def api_token_required(f):
    """Decorator for API token authentication."""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        token = request.headers.get('Authorization')
        
        if not token:
            return jsonify({'error': 'No token provided'}), 401
        
        # Remove 'Bearer ' prefix if present
        if token.startswith('Bearer '):
            token = token[7:]
        
        user = User.verify_api_token(token)
        
        if not user:
            return jsonify({'error': 'Invalid or expired token'}), 401
        
        # Make user available in route
        g.current_user = user
        
        return f(*args, **kwargs)
    return decorated_function

# Usage
@app.route('/api/protected')
@api_token_required
def protected_api():
    """Protected API endpoint."""
    return jsonify({
        'message': 'Success',
        'user': g.current_user.username
    })

@app.route('/api/token', methods=['POST'])
def generate_token():
    """Generate API token with credentials."""
    data = request.get_json()
    
    user = User.query.filter_by(username=data.get('username')).first()
    
    if user and user.check_password(data.get('password')):
        token = user.generate_api_token(expires_in=86400)  # 24 hours
        
        return jsonify({
            'token': token,
            'expires_in': 86400
        })
    
    return jsonify({'error': 'Invalid credentials'}), 401
```

---

## Best Practices

### Security Checklist

```python
# ✅ DO's
# 1. Use strong secret keys
app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY') or secrets.token_hex(32)

# 2. Hash passwords properly
user.set_password(password)  # Never store plain text

# 3. Use HTTPS in production
app.config['SESSION_COOKIE_SECURE'] = True

# 4. Set appropriate cookie flags
app.config['SESSION_COOKIE_HTTPONLY'] = True
app.config['SESSION_COOKIE_SAMESITE'] = 'Lax'

# 5. Implement rate limiting on login
@auth_bp.route('/login', methods=['POST'])
@rate_limit(max_requests=5, window=300)  # 5 attempts per 5 minutes
def login():
    pass

# 6. Use CSRF protection
csrf = CSRFProtect(app)

# 7. Validate and sanitize input
from bleach import clean
user_input = clean(user_input, tags=[], strip=True)

# 8. Implement account lockout
if user.failed_login_attempts >= 5:
    user.locked_until = datetime.utcnow() + timedelta(minutes=15)

# ❌ DON'Ts
# 1. Never store plain text passwords
# 2. Never commit secrets to version control
# 3. Never use weak session secrets
# 4. Never trust user input without validation
# 5. Never expose sensitive data in error messages
```

---

## Complete Authentication Example

```python
# app.py - Complete setup
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from flask_wtf.csrf import CSRFProtect

app = Flask(__name__)
app.config.from_object('config.Config')

db = SQLAlchemy(app)
login_manager = LoginManager(app)
csrf = CSRFProtect(app)

login_manager.login_view = 'auth.login'
login_manager.login_message = 'Please log in to access this page.'

@login_manager.user_loader
def load_user(user_id):
    from models import User
    return User.query.get(int(user_id))

# Register blueprints
from routes.auth import auth_bp
app.register_blueprint(auth_bp)

if __name__ == '__main__':
    app.run(debug=True)
```

---

## Testing Authentication

```python
# tests/test_auth.py
import unittest
from app import app, db
from models import User

class AuthTestCase(unittest.TestCase):
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
    
    def test_registration(self):
        """Test user registration."""
        response = self.client.post('/auth/register', data={
            'username': 'testuser',
            'email': 'test@example.com',
            'password': 'password123',
            'password_confirm': 'password123'
        }, follow_redirects=True)
        
        self.assertEqual(response.status_code, 200)
        
        with app.app_context():
            user = User.query.filter_by(username='testuser').first()
            self.assertIsNotNone(user)
    
    def test_login(self):
        """Test user login."""
        # Create user
        with app.app_context():
            user = User(username='testuser', email='test@example.com')
            user.set_password('password123')
            db.session.add(user)
            db.session.commit()
        
        # Test login
        response = self.client.post('/auth/login', data={
            'username': 'testuser',
            'password': 'password123'
        }, follow_redirects=True)
        
        self.assertEqual(response.status_code, 200)
    
    def test_logout(self):
        """Test user logout."""
        with self.client:
            # Login first
            self.client.post('/auth/login', data={
                'username': 'testuser',
                'password': 'password123'
            })
            
            # Logout
            response = self.client.get('/auth/logout', follow_redirects=True)
            self.assertEqual(response.status_code, 200)
```

---

## Related Documentation

- [Flask-Login Documentation](https://flask-login.readthedocs.io/)
- [Werkzeug Security Utilities](https://werkzeug.palletsprojects.com/en/latest/utils/#module-werkzeug.security)
- [Flask Security Best Practices](https://flask.palletsprojects.com/en/latest/security/)
