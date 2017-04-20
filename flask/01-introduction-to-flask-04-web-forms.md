# Introduction to Flask

## Web Forms

The request object exposes all the information sent by the client with a 
request. In particular, `request.form` provides access to form data 
submitted in `POST` requests.

Although the support provided in Flask's request object is sufficient 
for the handling of web forms, there are a number of tasks that can 
become tedious and repetitive. Two good examples are the generation of 
HTML code for forms and the validation of the submitted form data.

The Flask-WTF extension makes working with web forms a much more 
pleasant experience. This extension is a Flask integration wrapper 
around the framework-agnostic WTForms package.

The extension is installed with pip:

```
(flask01) $ pip install flask-wtf
```

### Cross-Site Request Forgery (CSRF) Protection

By default, Flask-WTF protects all forms against Cross-Site Request 
Forgery (CSRF) attacks. A CSRF attack occurs when a malicious website 
sends requests to a different website on which the victim is logged in.

To implement CSRF protection, Flask-WTF needs the application to 
configure an encryption key. Flask-WTF uses this key to generate 
encrypted tokens that are used to verify the authenticity of requests 
with form data.

This example shows how to configure an encryption key:

```python
...
app = Flask(__name__)
app.config['SECRET_KEY'] = 'hard to guess string'
...
```

The `app.config` dictionary is a general-purpose place to store 
configuration variables used by the framework, the extensions, or the 
application itself. Configuration values can be added to 
the `app.config` object using standard dictionary syntax. The 
configuration object also has methods to import configuration values 
from files or the environment.

The `SECRET_KEY` configuration variable is used as a general-purpose 
encryption key by Flask and several third-party extensions. As its name 
implies, the strength of the encryption depends on the value of this 
variable being secret. Pick a different secret key in each application 
that you build and make sure that this string is not known by anyone.

As a warning, remember this: for added security, the secret key should 
be stored in an environment variable instead of being embedded in the 
code.

### Form Classes

The following example shows a simple web form that has a text field and 
a submit button:

```python
...
from flask_wtf import FlaskForm
from wtforms import StringField, SubmitField
from wtforms.validators import Required

class NameForm(FlaskForm):
    name = StringField('What is your name?', validators=[Required()])
    submit = SubmitField('Submit')
```

The list of standard HTML fields supported by WTForms is this:

Field type | Description
------------ | -------------
StringField | Text field
TextAreaField | Multiple-line text field
PasswordField | Password text field
HiddenField | Hidden text field
DateField | Text field that accepts a `datetime.date` value in a given format
DateTimeField | Text field that accepts a `datetime.datetime` value in a given format
IntegerField | Text field that accepts an integer value
DecimalField | Text field that accepts a `decimal.Decimal` value
FloatField | Text field that accepts a floating-point value
BooleanField | Checkbox with `True` and `False` values
RadioField | List of radio buttons
SelectField | Drop-down list of choices
SelectMultipleField | Drop-down list of choices with multiple selection
FileField | File upload field
SubmitField | Form submission button
FormField | Embed a form as a field in a container form
FieldList | List of fields of a given type

The list of WTForms built-in validators is this:

Validator | Description
------------ | -------------
Email | Validates an email address
EqualTo | Compares the values of two fields; useful when requesting a password to be entered twice for confirmation
IPAddress | Validates an IPv4 network address
Length | Validates the length of the string entered
NumberRange | Validates that the value entered is within a numeric range
Optional | Allows empty input on the field, skipping additional validators
Required | Validates that the field contains data
Regexp | Validates the input against a regular expression
URL | Validates a URL
AnyOf | Validates that the input is one of a list of possible values
NoneOf | Validates that the input is none of a list of possible values

### HTML Rendering of Forms

This is an example of how we might use Flask-WTF and Flask-Bootstrap to 
render a form at `index.html`:

```html
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

{% block title %}Flasky{% endblock %}

{% block page_content %}
<div class="page-header">
    <h1>Hello, {% if name %}{{ name }}{% else %}Stranger{% endif %}!</h1>
</div>
{{ wtf.quick_form(form) }}
{% endblock %}
```

### Form Handling in View Functions

This is an example of a view function for form handling:

```python
@app.route('/', methods=['GET', 'POST'])
def index():
    name = None
    form = NameForm()
    if form.validate_on_submit():
        name = form.name.data
        form.name.data = ''
    return render_template('index.html', form=form, name=name)
```

### Redirects and User Sessions

Applications can "remember" things from one request to the next by 
storing them in the *user session*, private storage that is available to 
each connected client. The user session was introduced as one of the 
variables associated with the request context. It's called `session` and 
is accessed like a standard Python dictionary.

By default, user sessions are stored in client-side cookies that are
cryptographically signed using the configured `SECRET_KEY`. Any 
tampering with the cookie content would render the signature invalid, 
thus invalidating the session.

This is an example that shows a new version of the `index()` view 
function that implements redirects and user sessions:

```python
...
from flask import Flask, render_template, session, redirect, url_for
...
@app.route('/', methods=['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        session['name'] = form.name.data
        return redirect(url_for('index'))
    return render_template('index.html', form=form, name=session.get('name'))
```

### Message Flashing

Sometimes it is useful to give the user a status update after a request 
is completed. This could be a confirmation message, a warning, or an 
error. A typical example is when you submit a login form to a website 
with a mistake and the server responds by rendering the login form again 
with a message above it that informs you that your username or password 
is invalid.

Flask includes this functionality as a core feature. The following 
example shows how the `flash()` function can be used for this purpose:

```python
...
from flask import Flask, render_template, session, redirect, url_for, flash
...
@app.route('/', methods=['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        old_name = session.get('name')
        if old_name is not None and old_name != form.name.data:
            flash('Looks like you have changed your name!')
        session['name'] = form.name.data
        form.name.data = ''
        return redirect(url_for('index'))
    return render_template('index.html', form = form, name = session.get('name'))
```

Calling `flash()` is not enough to get messages displayed; the templates 
used by the application need to render these messages. The best place to 
render flashed messages is the base template, because that will enable 
these messages in all pages. Flask makes a `get_flashed_messages()` 
function available to templates to retrieve the messages and render 
them, as shown here in `base.html`:

```html
...
{% block content %}
<div class="container">
    {% for message in get_flashed_messages() %}
    <div class="alert alert-warning">
        <button type="button" class="close" data-dismiss="alert">&times;</button>
        {{ message }}
    </div>
    {% endfor %}
    
    {% block page_content %}{% endblock %}
</div>
{% endblock %}
```
