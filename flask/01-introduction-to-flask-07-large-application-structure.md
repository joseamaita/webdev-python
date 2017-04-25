# Introduction to Flask

## Large Application Structure

Although having small web applications stored in a single script can be 
very convenient, this approach does not scale well. As the application 
grows in complexity, working with a single large source file becomes 
problematic.

Unlike most other web frameworks, Flask does not impose a specific 
organization for large projects; the way to structure the application is 
left entirely to the developer. In this part, a possible way to organize 
a large application in packages and modules is presented.

### Project Structure

The structure for the project `flasky.git`  has three (03) top-level 
folders:

* The Flask application lives inside a package generically 
named *application*.
* The *migrations* folder contains the database migration scripts, as 
before.
* Unit tests are written in a *tests* package.

There are also a few new files:

* The file *requirements.txt* lists the package dependencies so that it 
is easy to regenerate an identical virtual environment on a different 
computer.
* The file *config.py* stores the configuration settings.
* The file *manage.py* launches the application and other application 
tasks.

To help you fully understand this structure, the following sections 
describe the process to convert the *hello.py* application to it.

### Configuration Options

Applications often need several configuration sets. The best example of 
this is the need to use different databases during development, testing, 
and production so that they don't interfere with each other.

Instead of the simple dictionary-like structure configuration used 
by *hello.py*, a hierarchy of configuration classes can be used. This is 
shown in the *config.py* file.

### Application Package

The application package is where all the application code, templates, 
and static files live. It is called simply *application*, though it can 
be given an application-specific name if desired. The *templates* 
and *static* folders are part of the application package, so these two 
folders are moved inside *application*. The database models and the 
email support functions are also moved inside this package, each in its 
own module as *application/models.py* and *application/email.py*.

**Using an Application Factory**

The way the application is created in the single-file version is very 
convenient, but it has one big drawback. Because the application is 
created in the global scope, there is no way to apply configuration 
changes dynamically: by the time the script is running, the application 
instance has already been created, so it is already too late to make 
configuration changes. This is particularly important for unit tests 
because sometimes it is necessary to run the application under different 
configuration settings for better test coverage.

The solution to this problem is to delay the creation of the application 
by moving it into a *factory function* that can be explicitly invoked 
from the script. This not only gives the script time to set the 
configuration but also the ability to create multiple application 
instances, something that can also be very useful during testing. The 
application factory function is defined in the *application* package 
constructor.

**Implementing Application Functionality in a Blueprint**

The conversion to an application factory introduces a complication for 
routes. In single-script applications, the application instance exists 
in the global scope, so routes can be easily defined using 
the `app.route` decorator. But now that the application is created at 
runtime, the `app.route` decorator begins to exist only 
after `create_app()` is invoked, which is too late. Like routes, custom 
error page handlers present the same problem, as these are defined with 
the `app.errorhandler` decorator.

Luckily Flask offers a better solution using *blueprints*. A blueprint 
is similar to an application in that it can also define routes. The 
difference is that routes associated with a blueprint are in a dormant 
state until the blueprint is registered with an application, at which 
point the routes become part of it. Using a blueprint defined in the 
global scope, the routes of the application can be defined in almost the 
same way as in the single-script application.

Like applications, blueprints can be defined all in a single file or can 
be created in a more structured way with multiple modules inside a 
package. To allow for the greatest flexibility, a subpackage inside the 
application package will be created to host the blueprint 
(*application/main/__init__.py*)). The following example shows the 
package constructor, which creates the blueprint:

```python
...
from flask import Blueprint

main = Blueprint('main', __name__)

from . import views, errors
...
```

Blueprints are created by instantiating an object of class `Blueprint`. 
The constructor for this class takes two required arguments: the 
blueprint name and the module or package where the blueprint is located. 
As with applications, Python's `__name__` variable is in most cases the 
correct value for the second argument.

The routes of the application are stored in 
an *application/main/views.py* module inside the package, and the error 
handlers are in *application/main/errors.py*. Importing these modules 
causes the routes and error handlers to be associated with the 
blueprint. It is important to note that the modules are imported at the 
bottom of the *application/__init__.py* script to avoid circular 
dependencies, because *views.py* and *errors.py* need to import 
the `main` blueprint.

The blueprint is registered with the application inside 
the `create_app()` factory function.

A difference when writing error handlers inside a blueprint is that if 
the `errorhandler` decorator is used, the handler will only be invoked 
for errors that originate in the blueprint. To install application-wide 
error handlers, the `app_errorhandler` must be used instead.

To complete the changes to the application page, the form objects are 
also stored inside the blueprint in an *application/main/forms.py* 
module.

### Launch Script

The *manage.py* file in the top-level folder is used to start the 
application.

The script begins by creating an application. The configuration used is 
taken from the environment variable `FLASK_CONFIG` if it's defined; if 
not, the default configuration is used. Flask-Script, Flask-Migrate, and 
the custom context for the Python shell are then initialized.

As a convenience, a shebang line is added, so that on Unix-based 
operating systems the script can be executed as `./manage.py` instead of 
the more verbose `python manage.py`.

### Requirements File

Applications must include a *requirements.txt* file that records all the 
package dependencies, with the exact version numbers. This is important 
in case the virtual environment needs to be regenerated in a different 
machine, such as the machine on which the application will be deployed 
for production use. This file can be generated automatically by pip with 
the following command:

```
(flask01) $ pip freeze > requirements.txt
```

It is a good idea to refresh this file whenever a package is installed 
or upgraded. If you open the file, you should see something like this:

```
alembic==0.9.1
blinker==1.4
click==6.7
dominate==2.3.1
Flask==0.12.1
Flask-Bootstrap==3.3.7.1
Flask-Mail==0.9.1
Flask-Migrate==2.0.3
Flask-Moment==0.5.1
Flask-Script==2.0.5
Flask-SQLAlchemy==2.2
Flask-WTF==0.14.2
itsdangerous==0.24
Jinja2==2.9.6
Mako==1.0.6
MarkupSafe==1.0
python-editor==1.0.3
SQLAlchemy==1.1.9
visitor==0.1.3
Werkzeug==0.12.1
WTForms==2.1
```

When you need to build a perfect replica of the virtual environment, you 
can create a new virtual environment and run the following command on 
it:

```
(flask01) $ pip install -r requirements.txt
```

### Unit Tests

Even though there isn't a lot to test yet, let's see two simple tests as 
an example ( `test_basics.py` ):

```python
import unittest
from flask import current_app
from application import create_app, db


class BasicsTestCase(unittest.TestCase):
    def setUp(self):
        self.app = create_app('testing')
        self.app_context = self.app.app_context()
        self.app_context.push()
        db.create_all()

    def tearDown(self):
        db.session.remove()
        db.drop_all()
        self.app_context.pop()

    def test_app_exists(self):
        self.assertFalse(current_app is None)

    def test_app_is_testing(self):
        self.assertTrue(current_app.config['TESTING'])
```

The tests are written using the standard `unittest` package from the 
Python standard library. The `setUp()` and `tearDown()` methods run 
before and after each test, and any methods that have a name that begins 
with `test_` are executed as tests.

The `setUp()` method tries to create an environment for the test that is 
close to that of a running application. It first creates an application 
configured for testing and activates its context. This step ensures that 
tests have access to `current_app`, like regular requests. Then it 
creates a brand-new database that the test can use when necessary. The 
database and the application context are removed in the `tearDown()` 
method.

The first test ensures that the application instance exists. The second 
test ensures that the application is running under the testing 
configuration. To make the *tests* folder a proper package, 
a *tests/__init__.py* file needs to be added, but this can be an empty 
file, as the `unittest` package can scan all the modules and locate the 
tests.

To run the unit tests, a custom command can be added to the *manage.py* 
script. The following example shows how to add a `test` command.

```python
@manager.command
def test():
    """Run the unit tests."""
    import unittest
    tests = unittest.TestLoader().discover('tests')
    unittest.TextTestRunner(verbosity=2).run(tests)
```

The `manager.command` decorator makes it simple to implement custom 
commands. The name of the decorated function is used as the command 
name, and the function's docstring is displayed in the help messages. 
The implementation of `test()` function invokes the test runner from 
the `unittest` package.

The unit tests can be executed as follows:

```
(flask01) $ python manage.py test
test_app_exists (test_basics.BasicsTestCase) ... ok
test_app_is_testing (test_basics.BasicsTestCase) ... ok

----------------------------------------------------------------------
Ran 2 tests in 1.584s

OK
```

### Database Setup

The restructured application uses a different database than the 
single-script version.

The database URL is taken from an environment variable as a first 
choice, with a default SQLite database as an alternative. The 
environment variables and SQLite database filenames are different for 
each of the three configurations. For example, in the development 
configuration the URL is obtained from environment 
variable `DEV_DATABASE_URL`, and if that is not defined then a SQLite 
database with the name *data-dev.sqlite* is used.

Regardless of the source of the database URL, the database tables must 
be created for the new database. When working with Flask-Migrate to keep 
track of migrations, database tables can be created or upgraded to the 
latest revision with a single command:

```
(flask01) $ python manage.py db upgrade
```
