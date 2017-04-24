# Introduction to Flask

## Email

Many types of applications need to notify users when certain events 
occur, and the usual method of communication is email. Although 
the `smtplib` package from the Python standard library can be used to 
send email inside a Flask application, the Flask-Mail extension wraps 
smtplib and integrates it nicely with Flask.

### Email Support with Flask-Mail

Flask-Mail is installed with pip:

```
(flask01) $ pip install flask-mail
```

The extension connects to a Simple Mail Transfer Protocol (SMTP) server 
and passes emails to it for delivery. If no configuration is given, 
Flask-Mail connects to *localhost* at port 25 and sends email without 
authentication. The following table shows the list of configuration keys 
that can be used to configure the SMTP server.

Key | Default | Description
------------ | ------------- | -------------
`MAIL_HOSTNAME` | *localhost* | Hostname or IP address of the email server
`MAIL_PORT` | `25` | Port of the email server
`MAIL_USE_TLS` | `False` | Enable Transport Layer Security (TLS) security
`MAIL_USE_SSL` | `False` | Enable Secure Sockets Layer (SSL) security
`MAIL_USERNAME` | `None` | Mail account username
`MAIL_PASSWORD` | `None` | Mail account password

During development it may be more convenient to connect to an external 
SMTP server. The following example shows how to configure the 
application to send email through a Google Gmail account:

```python
import os
# ...
app.config['MAIL_SERVER'] = 'smtp.googlemail.com'
app.config['MAIL_PORT'] = 587
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USERNAME'] = os.environ.get('MAIL_USERNAME')
app.config['MAIL_PASSWORD'] = os.environ.get('MAIL_PASSWORD')
```

Never write account credentials directly in your scripts, particularly 
if you plan to release your work as open source. To protect your account 
information, have your script import sensitive information from the 
environment.

Flask-Mail is initialized as shown here:

```python
...
from flask_mail import Mail
...
mail = Mail(app)
```

The two environment variables that hold the email server username and 
password need to be defined in the environment. If you are on Linux or 
Mac OS X using bash, you can set these variables as follows:

```
(flask01) $ export MAIL_USERNAME=<Gmail username>
(flask01) $ export MAIL_PASSWORD=<Gmail password>
```

For Microsoft Windows users, the environment variables are set as 
follows:

```
(flask01) $ set MAIL_USERNAME=<Gmail username>
(flask01) $ set MAIL_PASSWORD=<Gmail password>
```

**Sending Email from the Python Shell**

To test the configuration, you can start a shell session and send a test 
email:

```python
(flask01) $ python hello.py shell
>>> from flask_mail import Message
>>> from hello import mail
>>> msg = Message('test subject #01', sender='MyName <sender@gmail.com>', 
...     recipients=['recipient@gmail.com'])
>>> msg.body = 'text body #01'
>>> msg.html = '<b>HTML</b> body'
>>> with app.app_context():
...     mail.send(msg)
... 
```

Note that Flask-Mail's `send()` function uses `current_app`, so it needs 
to be executed with an activated application context.

**Integrating Emails with the Application**

To avoid having to create email messages manually every time, it is a 
good idea to abstract the common parts of the application's email 
sending functionality into a function. As an added benefit, this 
function can render email bodies from Jinja2 templates to have the most 
flexibility. The implementation is shown here:

```python
...
from flask_mail import Message
...
app.config['FLASKY_MAIL_SUBJECT_PREFIX'] = '[Flasky]'
app.config['FLASKY_MAIL_SENDER'] = 'Flasky Admin <flasky@example.com>'
...
def send_email(to, subject, template, **kwargs):
    msg = Message(app.config['FLASKY_MAIL_SUBJECT_PREFIX'] + ' ' + subject,
                  sender=app.config['FLASKY_MAIL_SENDER'], recipients=[to])
    msg.body = render_template(template + '.txt', **kwargs)
    msg.html = render_template(template + '.html', **kwargs)
    mail.send(msg)
```

The function relies on two application-specific configuration keys that 
define a prefix string for the subject and the address that will be used 
as sender. The `send_email` function takes the destination address, a 
subject line, a template for the email body, and a list of keyword 
arguments. The template name must be given without the extension, so 
that two versions of the template can be used for the plain- and 
rich-text bodies. The keyword arguments passed by the caller are given 
to the `render_template()` calls so that they can be used by the 
templates that generate the email body.

The `index()` view function can be easily expanded to send an email to 
the administrator whenever a new name is received with the form. The 
following example shows this change:

```python
...
app.config['FLASKY_ADMIN'] = os.environ.get('FLASKY_ADMIN')
...
@app.route('/', methods=['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.name.data).first()
        if user is None:
            user = User(username=form.name.data)
            db.session.add(user)
            session['known'] = False
            if app.config['FLASKY_ADMIN']:
                send_email(app.config['FLASKY_ADMIN'], 'New User',
                           'mail/new_user', user=user)
        else:
            session['known'] = True
        session['name'] = form.name.data
        return redirect(url_for('index'))
    return render_template('index.html', form=form, name=session.get('name'),
                           known=session.get('known', False))
```

The recipient of the email is given in the `FLASKY_ADMIN` environment 
variable loaded into a configuration variable of the same name during 
startup. Two template files need to be created for the text and HTML 
versions of the email. These files are stored in a *mail* subfolder 
inside *templates* to keep them separate from regular templates. The 
email templates expect the user to be given as a template argument, so 
the call to `send_email()` includes it as a keyword argument.

In addition to the `MAIL_USERNAME` and `MAIL_PASSWORD` environment 
variables described earlier, the application also needs 
the `FLASKY_ADMIN` environment variable.

With these environment variables set, you can test the application and 
receive an email every time you enter a new name in the form.

**Sending Asynchronous Email**

If you sent a few test emails, you likely noticed that the `mail.send()` 
function blocks for a few seconds while the email is sent, making the 
browser look unresponsive during that time. To avoid unnecessary delays 
during request handling, the email send function can be moved to a 
background thread. The following example shows this change:

```python
...
from threading import Thread
...
def send_async_email(app, msg):
    with app.app_context():
        mail.send(msg)

def send_email(to, subject, template, **kwargs):
    msg = Message(app.config['FLASKY_MAIL_SUBJECT_PREFIX'] + ' ' + subject,
                  sender=app.config['FLASKY_MAIL_SENDER'], recipients=[to])
    msg.body = render_template(template + '.txt', **kwargs)
    msg.html = render_template(template + '.html', **kwargs)
    thr = Thread(target=send_async_email, args=[app, msg])
    thr.start()
    return thr
```

This implementation highlights an interesting problem. Many Flask 
extensions operate under the assumption that there are active 
application and request contexts. Flask-Mail's `send()` function 
uses `current_app`, so it requires the application context to be active. 
But when the `mail.send()` function executes in a different thread, the 
application context needs to be created artificially 
using `app.app_context()`.

At this point in the application, you will notice that it is much more 
responsive, but keep in mind that for applications that send a large 
volume of email, having a job dedicated to sending email is more 
appropriate than starting a new thread for every email. For example, the 
execution of the `send_async_email()` function can be sent to a 
[Celery](http://www.celeryproject.org/) task queue.
