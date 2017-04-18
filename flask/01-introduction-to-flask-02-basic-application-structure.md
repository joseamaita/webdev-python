# Introduction to Flask

## Basic Application Structure

### Basic "Hello, World!" Application 01

This is our first basic "Hello, World!" application:

```python
# hello-world-01.py

from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, World!"

if __name__ == "__main__":
    app.run()
```

Run the application by typing:

```
(flask01) $ python hello-world-01.py
```

Notice the message shown in the console:

```
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

Open a web browser and navigate to `http://127.0.0.1:5000/` and make 
sure the "Hello, World!" message is shown.

### Basic "Hello, World!" Application 02

This is our second basic "Hello, World!" application:

```python
# hello-world-02.py

from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
    return '<h1>Hello, World!</h1>'

if __name__ == "__main__":
    app.run()
```

### Basic "Hello, World!" Application 03

Here, we enable the "debug mode" to the web development server:

```python
# hello-world-03.py

from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
    return '<h1>Hello, World!</h1>'

if __name__ == "__main__":
    app.run(debug=True)
```

Notice the message shown in the console:

```
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 319-948-371
```

### Using Variable or Dynamic Section in the Route 01

```python
# hello-name-01.py

from flask import Flask
app = Flask(__name__)

@app.route('/user/<name>')
def user(name):
    return "<h1>Hello, {0}</h1>".format(name)

if __name__ == "__main__":
    app.run()
```

Run the application by typing:

```
(flask01) $ python hello-name-01.py
```

Open a web browser and navigate to `http://127.0.0.1:5000/user/myname` 
and make sure the "Hello, myname!" message is shown. Try multiple times 
with different names.

### Using Variable or Dynamic Section in the Route 02

```python
# hello-number-01.py

from flask import Flask
app = Flask(__name__)

@app.route('/user/<int:id>')
def user(id):
    return "<h1>Hello, this is your number -> {0}</h1>".format(id)

if __name__ == "__main__":
    app.run()
```

### A More Complete Application 01

```python
# hello-complete-01.py

from flask import Flask
app = Flask(__name__)

@app.route('/')
def index():
    return '<h1>Hello, World!</h1>'

@app.route('/user/<name>')
def user(name):
    return "<h1>Hello, {0}</h1>".format(name)

if __name__ == "__main__":
    app.run(debug=True)
```

### A More Complete Application Using Request Contexts

```python
# hello-complete-02.py

from flask import Flask
from flask import request
app = Flask(__name__)

@app.route('/')
def index():
    return '<h1>Hello, World!</h1>'

@app.route('/browser')
def browser():
    user_agent = request.headers.get('User-Agent')
    return "<p>Your browser is, {0}</p>".format(user_agent)

@app.route('/user/<name>')
def user(name):
    return "<h1>Hello, {0}</h1>".format(name)

if __name__ == "__main__":
    app.run(debug=True)
```

There are two contexts in Flask: the *application context* and 
the *request context*. The following table shows the variables exposed 
by each of these contexts:

Variable name | Context | Description
------------ | ------------- | -------------
current_app | Application context | The application instance for the active application.
g | Application context | An object that the application can use for temporary storage during the handling of a request. This variable is reset with each request.
request | Request context | The request object, which encapsulates the contents of a HTTP request sent by the client.
session | Request context | The user session, a dictionary that the application can use to store values that are "remembered" between requests.

The following Python shell session demonstrates how the application 
context works:

```python
>>> from hellocomplete02 import app
>>> from flask import current_app
>>> current_app.name
Traceback (most recent call last):
  ...
    raise RuntimeError(_app_ctx_err_msg)
RuntimeError: Working outside of application context.

This typically means that you attempted to use functionality that needed
to interface with the current application object in a way.  To solve
this set up an application context with app.app_context().  See the
documentation for more information.
>>> app_ctx = app.app_context()
>>> app_ctx.push()
>>> current_app.name
'hellocomplete02'
>>> app_ctx.pop()
```

### How Request Dispatching Works

To see what the URL map in a Flask application looks like, you can 
inspect the map created for `hellocomplete02.py` in the Python shell. 
Let's see:

```python
>>> from hellocomplete02 import app
>>> app.url_map
Map([<Rule '/browser' (GET, HEAD, OPTIONS) -> browser>,
 <Rule '/' (GET, HEAD, OPTIONS) -> index>,
 <Rule '/static/<filename>' (GET, HEAD, OPTIONS) -> static>,
 <Rule '/user/<name>' (GET, HEAD, OPTIONS) -> user>])
```

### How Request Hooks Works

Sometimes it is useful to execute code before or after each request is 
processed. For example, at the start of each request it may be necessary 
to create a database connection, or authenticate the user making the 
request. Instead of duplicating the code that does this in every view 
function, Flask gives you the option to register common functions to be 
invoked before or after a request is dispatched to a view function.

Request hooks are implemented as decorators. These are the four (04) 
hooks supported by Flask:

* before_first_request
* before_request
* after_request
* teardown_request

A common pattern to share data between request hook functions and view 
functions is to use the `g` context global. For example, 
a `before_request` handler can load the logged-in user from the database 
and store it in `g.user`. Later, when the view function is invoked, it 
can access the user from there.

### Responses

When Flask invokes a view function, it expects its return value to be 
the response to the request. In most cases the response is a simple 
string that is sent back to the client as an HTML page.

But the HTTP protocol requires more than a string as a response to a 
request. A very important part of the HTTP response is 
the *status code*, which Flask by default sets to 200, the code that 
indicates that the request was carried out successfully.

When a view function needs to respond with a different status code, it 
can add the numeric code as a second return value after the response 
text. For example, the following view function returns a 400 status 
code, the code for a bad request error:

```python
@app.route('/')
def index():
    return '<h1>Bad Request</h1>', 400
```

Responses returned by view functions can also take a third argument, a 
dictionary of headers that are added to the HTTP response. This is 
rarely needed.

Instead of returning one, two, or three values as a tuple, Flask view 
functions have the option of returning a `Response` object. 
The `make_response()` function takes one, two, or three arguments, the 
same values that can be returned from a view function, and returns 
a `Response` object. Sometimes it is useful to perform this conversion 
inside the view function and then use the methods of the response object 
to further configure the response. The following example creates a 
response object and then sets a cookie in it:

```python
# response-01.py

from flask import Flask
from flask import make_response
app = Flask(__name__)

@app.route('/')
def index():
    response = make_response('<h1>This document carries a cookie!</h1>')
    response.set_cookie('answer', '42')
    return response

if __name__ == "__main__":
    app.run(debug=True)
```

There is a special type of response called a *redirect*. This response 
does not include a page document; it just gives the browser a new URL 
from which to load a new page. Redirects are commonly used with web 
forms.

A redirect is typically indicated with a 302 response status code and 
the URL to redirect to given in a `Location` header. A redirect response 
can be generated using a three-value return, or also with a `Response` 
object, but given its frequent use, Flask provides a `redirect()` helper 
function that creates this response:

```python
# response-02.py

from flask import Flask
from flask import redirect
app = Flask(__name__)

@app.route('/')
def index():
    return redirect('http://3leautomation.com')

if __name__ == "__main__":
    app.run(debug=True)

```

Another special response is issued with the `abort` function, which is 
used for error handling. The following example returns status code 404 
if the `id` dynamic argument given in the URL does not represent a valid 
user:

```python
# response-03.py

from flask import Flask
from flask import abort
app = Flask(__name__)

@app.route('/user/<id>')
def get_user(id):
    user = load_user(id)
    if not user:
        abort(404)
    return "<h1>Hello, {0}</h1>".format(user.name)

if __name__ == "__main__":
    app.run(debug=True)

```

Note that `abort` does not return control back to the function that 
calls it but gives control back to the web server by raising an 
exception.

### Flask Extensions

Flask is designed to be extended. It intentionally stays out of areas of 
important functionality such as database and user authentication, giving 
you the freedom to select the packages that fit your application the 
best, or to write your own if you so desire.

To give you an idea of how an extension is incorporated into an 
application, the following section adds an extension 
to `hello-complete-02.py` that enhances the application with 
command-line arguments.

**Command-Line Options with Flask-Script**

Flask's development web server supports a number of startup 
configuration options, but the only way to specify them is by passing 
them as arguments to the `app.run()` call in the script. This is not 
very convenient; the ideal way to pass configuration options is through 
command-line arguments.

Flask-Script is an extension for Flask that adds a command-line parser 
to your Flask application. It comes packaged with a set of 
general-purpose options and also supports custom commands.

The extension is installed with pip:

```
(flask01) $ pip install flask-script
```

These are the changes needed to add command-line parsing to 
the `hello-complete-02.py` application:

```python
# hello-complete-03.py

from flask import Flask
from flask_script import Manager
app = Flask(__name__)
manager = Manager(app)

@app.route('/')
def index():
    return '<h1>Hello, World!</h1>'

@app.route('/browser')
def browser():
    user_agent = request.headers.get('User-Agent')
    return "<p>Your browser is, {0}</p>".format(user_agent)

@app.route('/user/<name>')
def user(name):
    return "<h1>Hello, {0}</h1>".format(name)

if __name__ == "__main__":
    manager.run()
```

Extensions developed specifically for Flask are exposed under 
the `flask_*` namespace. Flask-Script exports a class named `Manager`, 
which is imported from `flask_script`.

The method of initialization of this extension is common to many 
extensions: an instance of the main class is initialized by passing the 
application instance as an argument to the constructor. The created 
object is then used as appropriate for each extension. In this case, 
the server startup is routed through `manager.run()`, where the command 
line is parsed.

With these changes, the application acquires a basic set of command-line 
options. Running `hello-complete-03.py` now shows a usage message:

```
(flask01) $ python hello-complete-03.py
usage: hello-complete-03.py [-?] {shell,runserver} ...

positional arguments:
  {shell,runserver}
    shell            Runs a Python shell inside Flask application context.
    runserver        Runs the Flask development server i.e. app.run()

optional arguments:
  -?, --help         show this help message and exit
```

The `shell` command is used to start a Python shell session in the 
context of the application. You can use this session to run maintenance 
tasks or tests, or to debug issues.

The `runserver` command, as its name implies, starts the web server. 
Running `python hello-complete-03.py runserver` starts the web server in 
non-debug mode, but there many more options available:

```
(flask01) $ python hello-complete-03.py runserver -?
usage: hello-complete-03.py runserver [-?] [-h HOST] [-p PORT] [--threaded]
                                      [--processes PROCESSES]
                                      [--passthrough-errors] [-d] [-D] [-r]
                                      [-R]

Runs the Flask development server i.e. app.run()

optional arguments:
  -?, --help            show this help message and exit
  -h HOST, --host HOST
  -p PORT, --port PORT
  --threaded
  --processes PROCESSES
  --passthrough-errors
  -d, --debug           enable the Werkzeug debugger (DO NOT use in production
                        code)
  -D, --no-debug        disable the Werkzeug debugger
  -r, --reload          monitor Python files for changes (not
                        100{'option_strings': ['-r', '--reload'], 'dest':
                        'use_reloader', 'nargs': 0, 'const': True, 'default':
                        None, 'type': None, 'choices': None, 'required':
                        False, 'help': 'monitor Python files for changes (not
                        100% safe for production use)', 'metavar': None,
                        'container': <argparse._ArgumentGroup object at
                        0x7f1e01318e48>, 'prog': 'hello-complete-03.py
                        runserver'}afe for production use)
  -R, --no-reload       do not monitor Python files for changes
```

If you want to run the web server in debug mode and as restarter, type:

```
(flask01) $ python hello-complete-03.py runserver -d -r
```

The `-h` and `-p` arguments are a useful option because they tell the 
web server what network interface to listen to for connections from 
clients. By default, Flask's development web server listens for 
connections on `localhost`, so only connections originating from within 
the computer running the server are accepted. The following command 
makes the web server listen for connections on the public network 
interface, enabling other computers in the network to connect as well:

```
(flask01) $ python hello-complete-03.py runserver -h 192.168.1.9 -p 8967 -d -r
```

The web server should now be accessible from any computer in the network 
at *http://192.168.1.9:8967*, where "192.168.1.9" is the external IP 
address of the computer running the server.

The output from the console (with a remote connection) is shown:

```
 * Running on http://192.168.1.9:8967/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 319-948-371
192.168.1.3 - - [12/Apr/2017 21:50:42] "GET / HTTP/1.1" 200 -
192.168.1.3 - - [12/Apr/2017 21:50:42] "GET /favicon.ico HTTP/1.1" 404 -
```
