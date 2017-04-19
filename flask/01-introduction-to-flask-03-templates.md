# Introduction to Flask

## Templates

A template is a file that contains the text of a response, with 
placeholder variables for the dynamic parts that will be known only in 
the context of a request. The process that replaces the variables with 
actual values and returns a final response string is called *rendering*. 
For the task of rendering templates, Flask uses a powerful template 
engine called *Jinja2*.

### Basic "Hello, World!" Application with Jinja2 Templates

To built this example, do this:

* Create a `templates` folder within the main folder.
* Inside `templates`, create a HTML file and name it `index.html`.
* Inside `templates`, create a HTML file and name it `user.html`.
* Contents of "index.html":

```html
<h1>Hello, World!</h1>
```

* Contents of "user.html":

```html
<h1>Hello, {{ name }}</h1>
```

The main application is:

```python
# hello-world-jinja2-01.py

from flask import Flask, render_template
app = Flask(__name__)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/user/<name>')
def user(name):
    return render_template('user.html', name=name)

if __name__ == "__main__":
    app.run(debug=True)

```

### Working with Variables and Jinja2 Templates

Jinja2 recognizes variables of any type, even complex types such as 
lists, dictionaries and objects. The following are some more examples of 
variables used in templates:

```html
<p>A value from a dictionary: {{ mydict['key'] }}.</p>
<p>A value from a list: {{ mylist[3] }}.</p>
<p>A value from a list, with a variable index: {{ mylist[myintvar] }}.</p>
<p>A value from an object's method: {{ myobj.somemethod() }}.</p>
```

Variables can be modified with *filters*, which are added after the 
variable name with a pipe character as separator. Let's see:

```html
<p>Name variable capitalized: Hello, {{ name|capitalize }}</p>
```

Commonly used Jinja2 filters are:

Filter name | Description
------------ | -------------
safe | Renders the value without applying escaping
capitalize | Converts the first character of the value to uppercase and the rest to lowercase
lower | Converts the value to lowercase characters
upper | Converts the value to uppercase characters
title | Capitalizes each word in the value
trim | Removes leading and trailing whitespace from the value
striptags | Removes any HTML tags from the value before rendering

### Working with Control Structures

The following example shows how conditional statements can be entered in 
a template:

```html
{% if user %}
    Hello, {{ user }}!
{% else %}
    Hello, Stranger!
{% endif %}
```

Another common need in templates is to render a list of elements. This 
example shows how this can be done with a `for` loop:

```html
<ul>
    {% for comment in comments %}
        <li>{{ comment }}</li>
    {% endfor %}
</ul>
```

Jinja2 also supports *macros*, which are similar to functions in Python 
code. For example:

```html
{% macro render_comment(comment) %}
    <li>{{ comment }}</li>
{% endmacro %}

<ul>
    {% for comment in comments %}
        {{ render_comment(comment) }}
    {% endfor %}
</ul>
```

To make macros more reusable, they can be stored in standalone files 
that are then *imported* from all the templates that need them:

```html
{% import 'macros.html' as macros %}
<ul>
    {% for comment in comments %}
        {{ macros.render_comment(comment) }}
    {% endfor %}
</ul>
```

Portions of template code that need to be repeated in several places can 
be stored in a separate file and *included* from all the templates to 
avoid repetition:

```html
{% include 'common.html' %}
```

### Reuse Using Template Inheritance

First, a base template is created with the name `base.html`:

```html
<html>
    <head>
    {% block head %}
        <title>{% block title %}{% endblock %} - My Application</title>
    {% endblock %}
    </head>
    <body>
    {% block body %}
    {% endblock %}
    </body>
</html>
```

The following example is a derived template of the base template:

```html
{% extends "base.html" %}
{% block title %}Index{% endblock %}
{% block head %}
    {{ super() }}
    <style>
    </style>
{% endblock %}
{% block body %}
<h1>Hello, World!</h1>
{% endblock %}
```

### Twitter Bootstrap Integration with Flask-Bootstrap

Bootstrap is an open source framework from Twitter that provides user 
interface components to create clean and attractive web pages that are 
compatible with all modern web browsers.

The extension is installed with pip:

```
(flask01) $ pip install flask-bootstrap
```

To initialize, do this:

```python
...
from flask_bootstrap import Bootstrap
...
bootstrap = Bootstrap(app)
```

Once Flask-Bootstrap is initialized, a base template that includes all 
the Bootstrap files is available to the application. This template takes 
advantage of Jinja2's template inheritance; the application extends a 
base template that has the general structure of the page including the 
elements that import Bootstrap.

The following example shows a new version of `user.html` as a derived 
template:

```html
{% extends "bootstrap/base.html" %}

{% block title %}Flasky{% endblock %}

{% block navbar %}
<div class="navbar navbar-inverse" role="navigation">
    <div class="container">
        <div class="navbar-header">
            <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-collapse">
                <span class="sr-only">Toggle navigation</span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
            </button>
            <a class="navbar-brand" href="/">Flasky</a>
        </div>
        <div class="navbar-collapse collapse">
            <ul class="nav navbar-nav">
                <li><a href="/">Home</a></li>
            </ul>
        </div>
    </div>
</div>
{% endblock %}

{% block content %}
<div class="container">
    <div class="page-header">
        <h1>Hello, {{ name }}!</h1>
    </div>
</div>
{% endblock %}
```

### Custom Error Pages

This is a basic template of a code 404 error page named `404.html`:

```html
{% extends "base.html" %}

{% block content %}

<h1>File Not Found</h1>
<p><a href="{{ url_for('home') }}">Back</a></p>
{% endblock %}
```

Another example is:

```html
{% extends "base.html" %}

{% block title %}Page Not Found{% endblock %}

{% block page_content %}
<div class="page-header">
    <h1>Not Found</h1>
</div>
{% endblock %}
```

This is a basic template of a code 500 error page named `500.html`:

```html
{% extends "base.html" %}

{% block content %}

<h1>An unexpected error has occurred</h1>
<p>The administrator has been notified. Sorry for the inconvenience!</p>
<p><a href="{{ url_for('home') }}">Back</a></p>
{% endblock %}
```

Another example is:

```html
{% extends "base.html" %}

{% block title %}Internal Server Error{% endblock %}

{% block page_content %}
<div class="page-header">
    <h1>Internal Server Error</h1>
</div>
{% endblock %}
```

### Working with Links

In the main file:

```python
...
from flask import Flask, url_for
...
l1 = url_for('user', name='jose')    # l1 = /user/jose
l2 = url_for('index', page=2)        # l2 = /?page=2
l3 = url_for('index', page=2, _external=True)
# l3 = http://localhost:5000/?page=2
```

### Working with Static Files

An example when working with this type of files in `base.html` is:

```html
{% block head %}
{{ super() }}
<link rel="shortcut icon" href="{{ url_for('static', filename='favicon.ico') }}" type="image/x-icon">
<link rel="icon" href="{{ url_for('static', filename='favicon.ico') }}" type="image/x-icon">
{% endblock %}
```

### Localization of Dates and Times with Flask-Moment

The extension is installed with pip:

```
(flask01) $ pip install flask-moment
```

To initialize, do this in `hello.py`:

```python
...
from flask_moment import Moment
...
moment = Moment(app)
```

Import the `moment.js` library to `base.html`:

```html
{% block scripts %}
{{ super() }}
{{ moment.include_moment() }}
{% endblock %}
```

Add a datetime variable in `hello.py`:

```python
...
from datetime import datetime
...
@app.route('/')
def index():
    return render_template('index.html', current_time=datetime.utcnow())
...
```

A timestamp rendering with Flask-Moment in `index.html` is:

```html
<p>The local date and time is {{ moment(current_time).format('LLL') }}.</p>
<p>That was {{ moment(current_time).fromNow(refresh=True) }}</p>
```

The `format('LLL')` format renders the date and time according to the 
time zone and locale settings in the client computer. The argument 
determines the rendering style, from `'L'` to `'LLLL'` for different 
levels of verbosity. The `format()` function can also accept custom 
format specifiers.

The `fromNow()` render style shown in the second line renders a relative 
timestamp and automatically refreshes it as time passes. Initially this 
timestamp will be shown as "a few seconds ago," but the refresh option 
will keep it updated as time passes, so if you leave the page open for a 
few minutes you will see the text changing to "a minute ago," then "2 
minutes ago," and so on.
