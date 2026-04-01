# Jinja2 Template Syntax Guide

Jinja2 is Flask's default templating engine for rendering dynamic HTML content.

## Table of Contents
- [Basic Syntax](#basic-syntax)
- [Variables](#variables)
- [Control Structures](#control-structures)
- [Filters](#filters)
- [Template Inheritance](#template-inheritance)
- [Macros](#macros)
- [Dynamic HTML Tables](#dynamic-html-tables)

---

## Basic Syntax

### Three Main Delimiters

```jinja2
{# Comments - not rendered in output #}

{{ variable }}              {# Print variable #}

{% statement %}             {# Control structures #}
```

---

## Variables

### Rendering Variables

```python
# app.py
@app.route('/profile/<username>')
def profile(username):
    user = {
        'username': username,
        'email': 'user@example.com',
        'age': 25,
        'interests': ['Python', 'Flask', 'Web Development']
    }
    return render_template('profile.html', user=user)
```

```jinja2
{# profile.html #}
<h1>{{ user.username }}</h1>
<p>Email: {{ user['email'] }}</p>
<p>Age: {{ user.age }}</p>

{# Accessing list items #}
<p>First interest: {{ user.interests[0] }}</p>

{# Safe rendering (escapes HTML) #}
<p>{{ user.bio }}</p>

{# Render raw HTML (dangerous - use carefully) #}
<div>{{ user.html_content | safe }}</div>

{# Default values #}
<p>{{ user.phone | default('No phone provided') }}</p>
```

### Variable Types

```python
# app.py
@app.route('/demo')
def demo():
    context = {
        'title': 'Flask Demo',
        'count': 42,
        'price': 19.99,
        'is_active': True,
        'items': ['apple', 'banana', 'cherry'],
        'user': {'name': 'John', 'role': 'admin'},
        'empty': None
    }
    return render_template('demo.html', **context)
```

```jinja2
{# demo.html #}
<h1>{{ title }}</h1>

{# Numbers #}
<p>Count: {{ count }}</p>
<p>Price: ${{ price }}</p>

{# Booleans #}
{% if is_active %}
    <span class="active">Active</span>
{% endif %}

{# Lists #}
<p>Total items: {{ items | length }}</p>

{# Dictionaries #}
<p>User: {{ user.name }} ({{ user.role }})</p>

{# None/Null handling #}
<p>{{ empty | default('Not set') }}</p>
```

---

## Control Structures

### If Statements

```jinja2
{% if user.is_authenticated %}
    <p>Welcome back, {{ user.username }}!</p>
{% elif user.is_guest %}
    <p>Welcome, guest!</p>
{% else %}
    <p>Please log in.</p>
{% endif %}

{# Logical operators #}
{% if user.age >= 18 and user.verified %}
    <button>Access Premium Content</button>
{% endif %}

{% if user.role == 'admin' or user.role == 'moderator' %}
    <a href="/admin">Admin Panel</a>
{% endif %}

{# Negation #}
{% if not user.is_banned %}
    <p>Your account is in good standing.</p>
{% endif %}

{# Testing membership #}
{% if 'admin' in user.roles %}
    <button>Manage Users</button>
{% endif %}
```

### For Loops

```python
# app.py
@app.route('/products')
def products():
    products = [
        {'id': 1, 'name': 'Laptop', 'price': 999.99, 'stock': 5},
        {'id': 2, 'name': 'Mouse', 'price': 29.99, 'stock': 50},
        {'id': 3, 'name': 'Keyboard', 'price': 79.99, 'stock': 0}
    ]
    return render_template('products.html', products=products)
```

```jinja2
{# products.html #}
<ul>
{% for product in products %}
    <li>
        {{ loop.index }}. {{ product.name }} - ${{ product.price }}
        
        {% if product.stock > 0 %}
            <span class="in-stock">In Stock ({{ product.stock }})</span>
        {% else %}
            <span class="out-of-stock">Out of Stock</span>
        {% endif %}
    </li>
{% endfor %}
</ul>

{# Loop with else (when list is empty) #}
<ul>
{% for item in items %}
    <li>{{ item }}</li>
{% else %}
    <li>No items found.</li>
{% endfor %}
</ul>

{# Loop variables #}
{% for user in users %}
    <div class="{% if loop.first %}first{% endif %} {% if loop.last %}last{% endif %}">
        {{ loop.index }}. {{ user.name }}
        {# loop.index0 - zero-based index #}
        {# loop.revindex - reverse index #}
        {# loop.length - total items #}
        {# loop.cycle('odd', 'even') - alternate values #}
    </div>
{% endfor %}

{# Filtering in loop #}
{% for user in users if user.is_active %}
    <p>{{ user.name }}</p>
{% endfor %}
```

---

## Filters

### Common Filters

```jinja2
{# String manipulation #}
{{ name | upper }}                    {# JOHN DOE #}
{{ name | lower }}                    {# john doe #}
{{ name | title }}                    {# John Doe #}
{{ name | capitalize }}               {# John doe #}
{{ text | truncate(20) }}            {# Truncate to 20 chars... #}
{{ text | wordwrap(40) }}            {# Wrap text at 40 chars #}
{{ "hello world" | replace("world", "flask") }}  {# hello flask #}

{# Lists #}
{{ items | length }}                  {# Number of items #}
{{ items | first }}                   {# First item #}
{{ items | last }}                    {# Last item #}
{{ items | join(', ') }}              {# Join with commas #}
{{ items | sort }}                    {# Sort list #}
{{ items | reverse }}                 {# Reverse order #}
{{ users | map(attribute='name') | join(', ') }}  {# Extract attribute #}

{# Numbers #}
{{ price | round(2) }}                {# Round to 2 decimals #}
{{ 42 | abs }}                        {# Absolute value #}
{{ numbers | sum }}                   {# Sum of list #}

{# Defaults #}
{{ variable | default('N/A') }}       {# Default if undefined #}
{{ value | default('N/A', true) }}    {# Default if falsy #}

{# Date formatting (requires import) #}
{{ date | datetimeformat('%Y-%m-%d') }}

{# HTML escaping #}
{{ html_content | escape }}           {# Escape HTML #}
{{ html_content | e }}                {# Shorthand for escape #}
{{ trusted_html | safe }}             {# Don't escape #}

{# URL encoding #}
{{ url | urlencode }}
```

### Custom Filters

```python
# app.py
from datetime import datetime

@app.template_filter('datetimeformat')
def datetimeformat(value, format='%Y-%m-%d %H:%M'):
    """Format a datetime object."""
    if value is None:
        return ""
    return value.strftime(format)

@app.template_filter('currency')
def currency_filter(value):
    """Format value as currency."""
    return f"${value:,.2f}"

@app.template_filter('pluralize')
def pluralize_filter(count, singular='', plural='s'):
    """Return plural suffix if count != 1."""
    return singular if count == 1 else plural
```

```jinja2
{# Using custom filters #}
<p>Date: {{ user.created_at | datetimeformat }}</p>
<p>Price: {{ product.price | currency }}</p>
<p>{{ item_count }} item{{ item_count | pluralize }}</p>
```

---

## Template Inheritance

### Base Template

```jinja2
{# templates/base.html #}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}My App{% endblock %}</title>
    
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
    
    {% block extra_css %}{% endblock %}
</head>
<body>
    <nav>
        <a href="{{ url_for('index') }}">Home</a>
        {% if current_user.is_authenticated %}
            <a href="{{ url_for('profile') }}">Profile</a>
            <a href="{{ url_for('logout') }}">Logout</a>
        {% else %}
            <a href="{{ url_for('login') }}">Login</a>
        {% endif %}
    </nav>
    
    {% with messages = get_flashed_messages(with_categories=true) %}
        {% if messages %}
            <div class="flash-messages">
                {% for category, message in messages %}
                    <div class="alert alert-{{ category }}">{{ message }}</div>
                {% endfor %}
            </div>
        {% endif %}
    {% endwith %}
    
    <main>
        {% block content %}{% endblock %}
    </main>
    
    <footer>
        {% block footer %}
            <p>&copy; 2026 My App</p>
        {% endblock %}
    </footer>
    
    <script src="{{ url_for('static', filename='js/main.js') }}"></script>
    {% block extra_js %}{% endblock %}
</body>
</html>
```

### Child Template

```jinja2
{# templates/products.html #}
{% extends "base.html" %}

{% block title %}Products - {{ super() }}{% endblock %}

{% block extra_css %}
    <link rel="stylesheet" href="{{ url_for('static', filename='css/products.css') }}">
{% endblock %}

{% block content %}
    <h1>Our Products</h1>
    
    {% for product in products %}
        <div class="product-card">
            <h2>{{ product.name }}</h2>
            <p>{{ product.description }}</p>
            <p class="price">{{ product.price | currency }}</p>
        </div>
    {% endfor %}
{% endblock %}

{% block extra_js %}
    <script src="{{ url_for('static', filename='js/products.js') }}"></script>
{% endblock %}
```

---

## Macros

### Defining Macros

```jinja2
{# templates/macros.html #}
{% macro render_field(field, label_text) %}
    <div class="form-group">
        <label for="{{ field.id }}">{{ label_text }}</label>
        {{ field(class="form-control") }}
        {% if field.errors %}
            <div class="errors">
                {% for error in field.errors %}
                    <span class="error">{{ error }}</span>
                {% endfor %}
            </div>
        {% endif %}
    </div>
{% endmacro %}

{% macro render_pagination(pagination, endpoint) %}
    <nav class="pagination">
        {% if pagination.has_prev %}
            <a href="{{ url_for(endpoint, page=pagination.prev_num) }}">Previous</a>
        {% endif %}
        
        {% for page in pagination.iter_pages() %}
            {% if page %}
                <a href="{{ url_for(endpoint, page=page) }}"
                   class="{% if page == pagination.page %}active{% endif %}">
                    {{ page }}
                </a>
            {% else %}
                <span>...</span>
            {% endif %}
        {% endfor %}
        
        {% if pagination.has_next %}
            <a href="{{ url_for(endpoint, page=pagination.next_num) }}">Next</a>
        {% endif %}
    </nav>
{% endmacro %}

{% macro alert(message, category='info') %}
    <div class="alert alert-{{ category }}" role="alert">
        {{ message }}
    </div>
{% endmacro %}
```

### Using Macros

```jinja2
{# templates/form.html #}
{% from 'macros.html' import render_field, alert %}

{% extends "base.html" %}

{% block content %}
    {{ alert('Please fill out the form', 'info') }}
    
    <form method="POST">
        {{ form.csrf_token }}
        
        {{ render_field(form.username, 'Username') }}
        {{ render_field(form.email, 'Email Address') }}
        {{ render_field(form.password, 'Password') }}
        
        <button type="submit">Submit</button>
    </form>
{% endblock %}
```

---

## Dynamic HTML Tables

### Simple Table with Dynamic Content

```python
# app.py
@app.route('/users')
def users_list():
    users = User.query.order_by(User.created_at.desc()).all()
    return render_template('users_table.html', users=users)
```

```jinja2
{# templates/users_table.html #}
{% extends "base.html" %}

{% block content %}
<div class="container">
    <h1>User Management</h1>
    
    <table class="table">
        <thead>
            <tr>
                <th>#</th>
                <th>Username</th>
                <th>Email</th>
                <th>Status</th>
                <th>Registered</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
        {% for user in users %}
            <tr class="{{ 'active' if user.is_active else 'inactive' }}">
                <td>{{ loop.index }}</td>
                <td>
                    <a href="{{ url_for('user_profile', user_id=user.id) }}">
                        {{ user.username }}
                    </a>
                </td>
                <td>{{ user.email }}</td>
                <td>
                    {% if user.is_active %}
                        <span class="badge badge-success">Active</span>
                    {% else %}
                        <span class="badge badge-danger">Inactive</span>
                    {% endif %}
                </td>
                <td>{{ user.created_at.strftime('%Y-%m-%d') }}</td>
                <td>
                    <a href="{{ url_for('edit_user', user_id=user.id) }}" class="btn btn-sm btn-primary">Edit</a>
                    <a href="{{ url_for('delete_user', user_id=user.id) }}" 
                       class="btn btn-sm btn-danger"
                       onclick="return confirm('Are you sure?')">Delete</a>
                </td>
            </tr>
        {% else %}
            <tr>
                <td colspan="6" class="text-center">No users found.</td>
            </tr>
        {% endfor %}
        </tbody>
    </table>
    
    {# Pagination #}
    {% if users.has_prev or users.has_next %}
    <nav>
        <ul class="pagination">
            {% if users.has_prev %}
                <li><a href="{{ url_for('users_list', page=users.prev_num) }}">Previous</a></li>
            {% endif %}
            
            {% for page_num in users.iter_pages(left_edge=1, right_edge=1, left_current=1, right_current=2) %}
                {% if page_num %}
                    <li class="{{ 'active' if page_num == users.page }}">
                        <a href="{{ url_for('users_list', page=page_num) }}">{{ page_num }}</a>
                    </li>
                {% else %}
                    <li class="disabled"><span>...</span></li>
                {% endif %}
            {% endfor %}
            
            {% if users.has_next %}
                <li><a href="{{ url_for('users_list', page=users.next_num) }}">Next</a></li>
            {% endif %}
        </ul>
    </nav>
    {% endif %}
</div>
{% endblock %}
```

### Advanced Table with Sorting and Filtering

```python
# app.py
@app.route('/orders')
def orders():
    sort_by = request.args.get('sort', 'date')
    filter_status = request.args.get('status', 'all')
    
    query = Order.query
    
    if filter_status != 'all':
        query = query.filter_by(status=filter_status)
    
    if sort_by == 'date':
        query = query.order_by(Order.order_date.desc())
    elif sort_by == 'amount':
        query = query.order_by(Order.total_amount.desc())
    elif sort_by == 'customer':
        query = query.order_by(Order.customer_name)
    
    orders = query.paginate(page=request.args.get('page', 1, type=int), per_page=20)
    
    return render_template('orders.html', orders=orders, sort_by=sort_by, filter_status=filter_status)
```

```jinja2
{# templates/orders.html #}
{% extends "base.html" %}

{% block content %}
<div class="container">
    <h1>Orders</h1>
    
    {# Filters #}
    <form method="GET" class="filters">
        <select name="status" onchange="this.form.submit()">
            <option value="all" {{ 'selected' if filter_status == 'all' }}>All Status</option>
            <option value="pending" {{ 'selected' if filter_status == 'pending' }}>Pending</option>
            <option value="completed" {{ 'selected' if filter_status == 'completed' }}>Completed</option>
            <option value="cancelled" {{ 'selected' if filter_status == 'cancelled' }}>Cancelled</option>
        </select>
    </form>
    
    <table class="table table-striped">
        <thead>
            <tr>
                <th>
                    <a href="{{ url_for('orders', sort='date', status=filter_status) }}">
                        Order Date {{ '↓' if sort_by == 'date' }}
                    </a>
                </th>
                <th>Order ID</th>
                <th>
                    <a href="{{ url_for('orders', sort='customer', status=filter_status) }}">
                        Customer {{ '↓' if sort_by == 'customer' }}
                    </a>
                </th>
                <th>Items</th>
                <th>
                    <a href="{{ url_for('orders', sort='amount', status=filter_status) }}">
                        Total {{ '↓' if sort_by == 'amount' }}
                    </a>
                </th>
                <th>Status</th>
            </tr>
        </thead>
        <tbody>
        {% for order in orders.items %}
            <tr>
                <td>{{ order.order_date.strftime('%Y-%m-%d %H:%M') }}</td>
                <td><a href="{{ url_for('order_detail', order_id=order.id) }}">#{{ order.id }}</a></td>
                <td>{{ order.customer_name }}</td>
                <td>{{ order.items | length }} items</td>
                <td>${{ "%.2f" | format(order.total_amount) }}</td>
                <td>
                    <span class="badge badge-{{ order.status }}">
                        {{ order.status | capitalize }}
                    </span>
                </td>
            </tr>
        {% else %}
            <tr>
                <td colspan="6" class="text-center">No orders found.</td>
            </tr>
        {% endfor %}
        </tbody>
    </table>
    
    {# Summary row #}
    {% if orders.items %}
    <tfoot>
        <tr>
            <td colspan="4" class="text-right"><strong>Total:</strong></td>
            <td><strong>${{ orders.items | sum(attribute='total_amount') | round(2) }}</strong></td>
            <td></td>
        </tr>
    </tfoot>
    {% endif %}
</div>
{% endblock %}
```

---

## Template Tests

```jinja2
{# Check variable types #}
{% if variable is defined %}
    <p>Variable exists</p>
{% endif %}

{% if variable is none %}
    <p>Variable is None</p>
{% endif %}

{% if value is number %}
    <p>{{ value }} is a number</p>
{% endif %}

{% if text is string %}
    <p>{{ text }} is a string</p>
{% endif %}

{% if items is iterable %}
    {% for item in items %}
        {{ item }}
    {% endfor %}
{% endif %}

{# Other tests #}
{{ value is even }}
{{ value is odd }}
{{ value is divisibleby(3) }}
{{ text is upper }}
{{ text is lower }}
```

---

## Include and Import

### Include

```jinja2
{# templates/dashboard.html #}
{% extends "base.html" %}

{% block content %}
    <h1>Dashboard</h1>
    
    {# Include navigation #}
    {% include 'components/sidebar.html' %}
    
    {# Include with variables #}
    {% include 'components/user_card.html' with context %}
    
    {# Include without parent context #}
    {% include 'components/footer.html' without context %}
{% endblock %}
```

### Import

```jinja2
{# Import macros from another template #}
{% import 'macros.html' as macros %}

{{ macros.render_field(form.username, 'Username') }}
{{ macros.alert('Success!', 'success') }}

{# Import specific macros #}
{% from 'macros.html' import render_field, alert %}

{{ render_field(form.email, 'Email') }}
```

---

## Advanced Patterns

### Set Variables in Templates

```jinja2
{% set total = 0 %}
{% for item in items %}
    {% set total = total + item.price %}
{% endfor %}
<p>Total: ${{ total }}</p>

{# Set multiple variables #}
{% set nav_class, show_header = 'dark', true %}
```

### Expressions and Operators

```jinja2
{# Arithmetic #}
{{ (price * quantity) + tax }}
{{ 10 // 3 }}  {# Integer division = 3 #}
{{ 10 % 3 }}   {# Modulo = 1 #}
{{ 2 ** 3 }}   {# Power = 8 #}

{# Comparison #}
{{ age >= 18 }}
{{ status == 'active' }}
{{ name != 'admin' }}

{# Logical #}
{{ is_logged_in and is_verified }}
{{ is_admin or is_moderator }}
{{ not is_banned }}

{# Ternary operator #}
{{ 'Active' if user.is_active else 'Inactive' }}

{# List literals #}
{% set colors = ['red', 'green', 'blue'] %}

{# Dict literals #}
{% set person = {'name': 'John', 'age': 30} %}
```

### Whitespace Control

```jinja2
{# Remove whitespace before #}
{%- if true %}
    Content
{% endif %}

{# Remove whitespace after #}
{% if true -%}
    Content
{% endif %}

{# Remove both #}
{%- if true -%}
    Content
{%- endif -%}
```

---

## Template Context Processors

```python
# app.py
from datetime import datetime

@app.context_processor
def inject_now():
    """Make 'now' available in all templates."""
    return {'now': datetime.utcnow()}

@app.context_processor
def utility_processor():
    """Inject utility functions."""
    def format_price(amount, currency='$'):
        return f'{currency}{amount:.2f}'
    return dict(format_price=format_price)
```

```jinja2
{# Available in all templates #}
<p>Current time: {{ now.strftime('%Y-%m-%d %H:%M') }}</p>
<p>Price: {{ format_price(19.99) }}</p>
```

---

## Form Rendering Example

```python
# forms.py
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, TextAreaField, SelectField
from wtforms.validators import DataRequired, Email, Length

class RegistrationForm(FlaskForm):
    username = StringField('Username', validators=[DataRequired(), Length(min=3, max=20)])
    email = StringField('Email', validators=[DataRequired(), Email()])
    password = PasswordField('Password', validators=[DataRequired(), Length(min=8)])
    bio = TextAreaField('Bio', validators=[Length(max=500)])
    role = SelectField('Role', choices=[('user', 'User'), ('admin', 'Admin')])
```

```jinja2
{# templates/register.html #}
{% extends "base.html" %}

{% block content %}
<div class="form-container">
    <h2>Register</h2>
    
    <form method="POST" novalidate>
        {{ form.csrf_token }}
        
        <div class="form-group">
            {{ form.username.label }}
            {{ form.username(class="form-control", placeholder="Enter username") }}
            {% if form.username.errors %}
                <div class="invalid-feedback">
                    {% for error in form.username.errors %}
                        <p>{{ error }}</p>
                    {% endfor %}
                </div>
            {% endif %}
        </div>
        
        <div class="form-group">
            {{ form.email.label }}
            {{ form.email(class="form-control", type="email") }}
            {% for error in form.email.errors %}
                <span class="error">{{ error }}</span>
            {% endfor %}
        </div>
        
        <div class="form-group">
            {{ form.password.label }}
            {{ form.password(class="form-control") }}
        </div>
        
        <div class="form-group">
            {{ form.bio.label }}
            {{ form.bio(class="form-control", rows=4) }}
            <small>{{ form.bio.data | length }}/500 characters</small>
        </div>
        
        <div class="form-group">
            {{ form.role.label }}
            {{ form.role(class="form-select") }}
        </div>
        
        <button type="submit" class="btn btn-primary">Register</button>
    </form>
</div>
{% endblock %}
```

---

## Complete Example: Blog Post List

```python
# app.py
@app.route('/blog')
def blog():
    page = request.args.get('page', 1, type=int)
    posts = Post.query.order_by(Post.published_at.desc()).paginate(
        page=page, per_page=10, error_out=False
    )
    
    return render_template('blog.html', 
                         posts=posts,
                         current_year=datetime.now().year)
```

```jinja2
{# templates/blog.html #}
{% extends "base.html" %}

{% block title %}Blog - {{ super() }}{% endblock %}

{% block content %}
<div class="blog-container">
    <h1>Blog Posts</h1>
    
    {% for post in posts.items %}
        <article class="post-card">
            <header>
                <h2>
                    <a href="{{ url_for('post_detail', slug=post.slug) }}">
                        {{ post.title }}
                    </a>
                </h2>
                <div class="post-meta">
                    <span class="author">By {{ post.author.username }}</span>
                    <span class="date">{{ post.published_at.strftime('%B %d, %Y') }}</span>
                    <span class="reading-time">{{ post.reading_time }} min read</span>
                </div>
            </header>
            
            <div class="post-excerpt">
                {{ post.content | truncate(200) | safe }}
            </div>
            
            <div class="post-tags">
                {% for tag in post.tags %}
                    <span class="tag">{{ tag.name }}</span>
                {% endfor %}
            </div>
            
            <footer>
                <span>{{ post.comments | length }} comment{{ post.comments | length | pluralize }}</span>
                <span>{{ post.likes }} like{{ post.likes | pluralize }}</span>
            </footer>
        </article>
    {% else %}
        <p class="no-posts">No blog posts yet. Check back soon!</p>
    {% endfor %}
    
    {# Pagination component #}
    {% if posts.pages > 1 %}
    <nav class="pagination">
        {% if posts.has_prev %}
            <a href="{{ url_for('blog', page=posts.prev_num) }}">&laquo; Prev</a>
        {% endif %}
        
        {% for page_num in range(1, posts.pages + 1) %}
            {% if page_num == posts.page %}
                <span class="current">{{ page_num }}</span>
            {% else %}
                <a href="{{ url_for('blog', page=page_num) }}">{{ page_num }}</a>
            {% endif %}
        {% endfor %}
        
        {% if posts.has_next %}
            <a href="{{ url_for('blog', page=posts.next_num) }}">Next &raquo;</a>
        {% endif %}
    </nav>
    {% endif %}
</div>
{% endblock %}
```

---

## Template Global Functions

### Built-in Globals

```jinja2
{# URL generation #}
<a href="{{ url_for('index') }}">Home</a>
<a href="{{ url_for('profile', username='john') }}">John's Profile</a>
<img src="{{ url_for('static', filename='logo.png') }}">

{# Get flashed messages #}
{% with messages = get_flashed_messages() %}
    {% if messages %}
        {% for message in messages %}
            <p>{{ message }}</p>
        {% endfor %}
    {% endif %}
{% endwith %}

{# Get flashed messages with categories #}
{% with messages = get_flashed_messages(with_categories=true) %}
    {% if messages %}
        {% for category, message in messages %}
            <div class="alert-{{ category }}">{{ message }}</div>
        {% endfor %}
    {% endif %}
{% endwith %}

{# Config access #}
<p>App name: {{ config.APP_NAME }}</p>

{# Request object #}
<p>Current path: {{ request.path }}</p>
<p>Method: {{ request.method }}</p>
{% if request.args.search %}
    <p>Searching for: {{ request.args.search }}</p>
{% endif %}

{# Session #}
{% if session.user_id %}
    <p>User ID: {{ session.user_id }}</p>
{% endif %}

{# Global g object #}
{% if g.user %}
    <p>Welcome, {{ g.user.username }}</p>
{% endif %}
```

---

## Auto-escaping and Security

```jinja2
{# Auto-escaped by default (safe) #}
<p>{{ user_input }}</p>

{# Explicitly escape #}
<p>{{ user_input | e }}</p>

{# Mark as safe (dangerous - only use with trusted content) #}
<div>{{ html_content | safe }}</div>

{# Escaping in JavaScript #}
<script>
    var username = {{ username | tojson }};  // Safely inject into JS
    var config = {{ config_dict | tojson }};
</script>

{# URL encoding #}
<a href="/search?q={{ query | urlencode }}">Search</a>
```

---

## Best Practices

1. **Always escape user input** - Never use `| safe` on user-provided content
2. **Use template inheritance** - Create a base template and extend it
3. **Keep logic in Python** - Templates should focus on presentation
4. **Use macros for reusable components** - Forms, cards, alerts
5. **Leverage context processors** - Make common data available globally
6. **Use filters for formatting** - Keep templates clean and readable
7. **Test for undefined variables** - Use `is defined` test
8. **Use meaningful variable names** - Make templates self-documenting

---

## Common Patterns Cheat Sheet

```jinja2
{# Conditional rendering #}
{{ value if condition else other_value }}

{# Loop with counter #}
{% for item in items %}
    {{ loop.index }}. {{ item }}
{% endfor %}

{# Safe dictionary access #}
{{ user.get('name', 'Anonymous') }}

{# Multiple conditions #}
{% if user.is_active and user.is_verified and not user.is_banned %}
    <p>Full access granted</p>
{% endif %}

{# Nested loops #}
{% for category in categories %}
    <h3>{{ category.name }}</h3>
    {% for product in category.products %}
        <p>{{ product.name }}</p>
    {% endfor %}
{% endfor %}

{# Format dates #}
{{ post.created_at.strftime('%B %d, %Y') }}

{# JSON data in JavaScript #}
<script>
    const data = {{ data | tojson }};
</script>
```

---

## Related Documentation

- [Jinja2 Official Documentation](https://jinja.palletsprojects.com/)
- [Flask Templating Guide](https://flask.palletsprojects.com/en/latest/templating/)
- [WTForms for Flask](https://flask-wtf.readthedocs.io/)
