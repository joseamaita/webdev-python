# Introduction to Flask

## Databases

A *database* stores application data in an organized way. The 
application then issues *queries* to retrieve specific portions as they 
are needed. The most commonly used databases for web applications are 
those based on the *relational* model, also called SQL databases in 
reference to the Structured Query Language they use. But in recent years
*document-oriented* and *key-value* databases, informally known together 
as NoSQL databases, have become popular alternatives.

### SQL Databases

Relational databases store data in *tables*, which model the different 
entities in the application's domain. For example, a database for an 
order management application will likely have `customers`, `products`, 
and `orders` tables.

### NoSQL Databases

Databases that do not follow the relational model are collectively 
referred to as *NoSQL* databases. One common organization for NoSQL 
databases uses *collections* instead of tables and *documents* instead 
of records.

### SQL or NoSQL?

SQL databases excel at storing structured data in an efficient and 
compact form. These databases go to great lengths to preserve 
consistency. NoSQL databases relax some of the consistency requirements 
and as a result can sometimes get a performance edge.

For small to medium-size applications, both SQL and NoSQL databases are 
perfectly capable and have practically equivalent performance.

### Python Database Frameworks

Python has packages for most database engines, both open source and 
commercial. Flask puts no restrictions on what database packages can be 
used, so you can work with MySQL, PostgreSQL, SQLite, Redis, MongoDB, or 
CouchDB if any of these is your favorite.

As if those weren't enough choices, there are also a number of database 
abstraction layer packages such as SQLAlchemy or MongoEngine that allow 
you to work at a higher level with regular Python objects instead of 
database entities such as tables, documents, or query languages.

There are a number of factors to evaluate when choosing a database 
framework:

* Ease of use
* Performance
* Portability
* Flask integration

### Database Management with Flask-SQLAlchemy

Flask-SQLAlchemy is a Flask extension that simplifies the use of 
SQLAlchemy inside Flask applications. SQLAlchemy is a powerful 
relational database framework that supports several database backends. 
It offers a high-level ORM and low level access to the database's native 
SQL functionality.

Like most other extensions, Flask-SQLAlchemy is installed with pip:

```
(flask01) $ pip install flask-sqlalchemy
```

In Flask-SQLAlchemy, a database is specified as a URL. The following 
table lists the format of database URLs for the three most popular 
database engines:

Database engine | URL
------------ | -------------
MySQL | mysql://username:password@hostname/database
PostgreSQL | postgresql://username:password@hostname/database
SQLite (Unix) | sqlite:////absolute/path/to/database
SQLite (Windows) | sqlite:///c:/absolute/path/to/database

The following example shows how to initialize and configure a simple 
SQLite database:

```python
from flask_sqlalchemy import SQLAlchemy

basedir = os.path.abspath(os.path.dirname(__file__))

app = Flask(__name__)

app.config['SQLALCHEMY_DATABASE_URI'] =\
    'sqlite:///' + os.path.join(basedir, 'data.sqlite')
app.config['SQLALCHEMY_COMMIT_ON_TEARDOWN'] = True

db = SQLAlchemy(app)
```

The `db` object instantiated from class SQLAlchemy represents the 
database and provides access to all the functionality of 
Flask-SQLAlchemy.

### Model Definition

The term *model* is used to refer to the persistent entities used by the 
application. In the context of an ORM, a model is typically a Python 
class with attributes that match the columns of a corresponding database 
table.

The database instance from Flask-SQLAlchemy provides a base class for 
models as well as a set of helper classes and functions that are used to 
define their structure.

This example shows how to define a "Role" and "User" model:

```python
class Role(db.Model):
    __tablename__ = 'roles'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(64), unique=True)
    
    def __repr__(self):
        return '<Role %r>' % self.name

class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, index=True)
    
    def __repr__(self):
        return '<User %r>' % self.username
```

The following table lists some of the column types that are available, 
along with the Python type used in the model:

Type name | Python type | Description
------------ | ------------- | -------------
Integer | `int` | Regular integer, typically 32 bits
SmallInteger | `int` | Short-range integer, typically 16 bits
BigInteger | `int` | Unlimited precision integer
Float | `float` | Floating-point number
Numeric | `decimal.Decimal` | Fixed-point number
String | `str` | Variable-length string
Text | `str` | Variable-length string, optimized for large or unbound length
Boolean | `bool` | Boolean value
Date | `datetime.date` | Date value
Time | `datetime.time` | Time value
DateTime | `datetime.datetime` | Date and time value
Interval | `datetime.timedelta` | Time interval
Enum | `str` | List of string values
PickleType | Any Python object | Automatic Pickle serialization
LargeBinary | `str` | Binary blob

The following table lists some of the options available for the 
remaining arguments to `db.Column`:

Option name | Description
------------ | -------------
`primary_key` | If set to `True`, the column is the table's primary key.
`unique` | If set to `True`, do not allow duplicate values for this column.
`index` | If set to `True`, create an index for this column, so that queries are more efficient.
`nullable` | If set to `True`, allow empty values for this column. If set to `False`, the column will not allow null values.
`default` | Define a default value for the column.

Flask-SQLAlchemy requires all models to define a *primary key* column, 
which is normally named `id`.

Although it's not strictly necessary, the two models include 
a `__repr__()` method to give them a readable string representation that 
can be used for debugging and testing purposes.

### Relationships

Relational databases establish connections between rows in different 
tables through the use of relationships.

This example shows how the one-to-many relationship is represented in 
the model classes:

```python
class Role(db.Model):
    # ...
    users = db.relationship('User', backref='role')

class User(db.Model):
    # ...
    role_id = db.Column(db.Integer, db.ForeignKey('roles.id'))
```

### Database Operations

The models are now fully configured according to the database idea and 
are ready to be used. The best way to learn how to work with these 
models is in a Python shell. The following sections will walk you 
through the most common database operations.

**Creating the Tables**

The very first thing to do is to instruct Flask-SQLAlchemy to create a 
database based on the model classes. The `db.create_all()` function does 
this:

```python
(flask01) $ python hello.py shell
>>> from hello import db
>>> db.create_all()
```

The `db.create_all()` function will not re-create or update a database 
table if it already exists in the database. This can be inconvenient 
when the models are modified and the changes need to be applied to an 
existing database. The brute-force solution to update existing database 
tables is to remove the old tables first:

```python
>>> db.drop_all()
>>> db.create_all()
```

Unfortunately, this method has the undesired side effect of destroying 
all the data in the old database.

**Inserting the Rows**

The following example creates a few roles and users:

```python
>>> from hello import Role, User
>>> admin_role = Role(name='Admin')
>>> mod_role = Role(name='Moderator')
>>> user_role = Role(name='User')
>>> user_john = User(username='john', role=admin_role)
>>> user_susan = User(username='susan', role=user_role)       
>>> user_david = User(username='david', role=user_role)
```

The objects exist only on the Python side so far; they have not been
written to the database yet. Because of that their `id` value has not 
yet been assigned:

```python
>>> print(admin_role.id)
None
>>> print(mod_role.id)
None
>>> print(user_role.id)
None
```

Changes to the database are managed through a database *session*, which 
Flask-SQLAlchemy provides as `db.session`. To prepare objects to be 
written to the database, they must be added to the session:

```python
>>> db.session.add(admin_role)
>>> db.session.add(mod_role)
>>> db.session.add(user_role)
>>> db.session.add(user_john)
>>> db.session.add(user_susan)
>>> db.session.add(user_david)
```

Or, more concisely:

```python
>>> db.session.add_all([admin_role, mod_role, user_role, user_john, user_susan, user_david])
```

To write the objects to the database, the session needs to be 
*committed* by calling its `commit()` method:

```python
>>> db.session.commit()
```

Check the `id` attributes again; they are now set:

```python
>>> print(admin_role.id)
1
>>> print(mod_role.id)
2
>>> print(user_role.id)
3
```

The `db.session` database session is not related to the Flask `session` 
object discussed previously. Database sessions are also called 
*transactions*.

Database sessions are extremely useful in keeping the database 
consistent. The commit operation writes all the objects that were added 
to the session atomically. If an error occurs while the session is being 
written, the whole session is discarded. If you always commit related 
changes together in a session, you are guaranteed to avoid database 
inconsistencies due to partial updates.

A database session can also be *rolled back*. If `db.session.rollback()` 
is called, any objects that were added to the database session are 
restored to the state they have in the database.

**Modifying Rows**

The `add()` method of the database session can also be used to update 
models. Continuing in the same shell session, the following example 
renames the `"Admin"` role to `"Administrator"`:

```python
>>> admin_role.name = 'Administrator'
>>> db.session.add(admin_role)
>>> db.session.commit()
```

**Deleting Rows**

The database session also has a `delete()` method. The following example 
deletes the `"Moderator"` role from the database:

```python
>>> db.session.delete(mod_role)
>>> db.session.commit()
```

Note that deletions, like insertions and updates, are executed only when 
the database session is committed.

**Querying Rows**

Flask-SQLAlchemy makes a `query` object available in each model class. 
The most basic query for a model is the one that returns the entire 
contents of the corresponding table:

```python
>>> Role.query.all()
[<Role 'Administrator'>, <Role 'User'>]
>>> User.query.all()
[<User 'john'>, <User 'susan'>, <User 'david'>]
```

A query object can be configured to issue more specific database 
searches through the use of *filters*. The following example finds all 
the users that were assigned the `"User"` role:

```python
>>> User.query.filter_by(role=user_role).all()
[<User 'susan'>, <User 'david'>]
```

It is also possible to inspect the native SQL query that SQLAlchemy 
generates for a given query by converting the query object to a string:

```python
>>> str(User.query.filter_by(role=user_role))
'SELECT users.id AS users_id, users.username AS users_username, users.role_id AS users_role_id \nFROM users \nWHERE ? = users.role_id'
```

If you exit the shell session, the objects created in the previous 
examples will cease to exist as Python objects but will continue to 
exist as rows in their respective database tables. If you then start a 
brand new shell session, you have to re-create Python objects from their 
database rows. The following example issues a query that loads the user 
role with name `"User"`:

```python
(flask01) $ python hello.py shell
>>> from hello import db, Role, User
>>> user_role = Role.query.filter_by(name='User').first()
>>> user_role
<Role 'User'>
```

Filters such as `filter_by()` are invoked on a query object and return a 
new refined query. Multiple filters can be called in sequence until the 
query is configured as needed.

The following table shows some of the most common filters available to 
queries:

Option | Description
------------ | -------------
`filter()` | Returns a new query that adds an additional filter to the original query
`filter_by()` | Returns a new query that adds an additional equality filter to the original query
`limit()` | Returns a new query that limits the number of results of the original query to the given number
`offset()` | Returns a new query that applies an offset into the list of results of the original query
`order_by()` | Returns a new query that sorts the results of the original query according to the given criteria
`group_by()` | Returns a new query that groups the results of the original query according to the given criteria

After the desired filters have been applied to the query, a call 
to `all()` will cause the query to execute and return the results as a 
list, but there are other ways to trigger the execution of a query 
besides `all()`. The following table shows other query execution 
methods:

Option | Description
------------ | -------------
`all()` | Returns all the results of a query as a list
`first()` | Returns the first result of a query, or `None` if there are no results
`first_or_404()` | Returns the first result of a query, or aborts the request and sends a 404 error as response if there are no results
`get()` | Returns the row that matches the given primary key, or `None` if no matching row is found
`get_or_404()` | Returns the row that matches the given primary key. If the key is not found it aborts the request and sends a 404 error as response
`count()` | Returns the result count of the query
`paginate()` | Returns a `Pagination` object that contains the specified range of results

Relationships work similarly to queries. The following example queries 
the one-to-many relationship between roles and users from both ends:

```python
>>> users = user_role.users
>>> users
<sqlalchemy.orm.dynamic.AppenderBaseQuery object at 0x7f6a3151ee48>
>>> users[0].role
<Role 'User'>
```

With the relationship configured in this way 
( `users = db.relationship('User', backref='role', lazy='dynamic')` ), 
the statement `user_role.users` returns a query that hasn't executed 
yet, so filters can be added to it:

```python
>>> user_role.users.order_by(User.username).all()
[<User 'david'>, <User 'susan'>]
>>> user_role.users.count()
2
```

### Database Use in View Functions

The database operations described in the previous sections can be used 
directly inside view functions. The following example shows a new 
version of the home page route that records names entered by users in 
the database:

```python
@app.route('/', methods=['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.name.data).first()
        if user is None:
            user = User(username=form.name.data)
            db.session.add(user)
            session['known'] = False
        else:
            session['known'] = True
        session['name'] = form.name.data
        return redirect(url_for('index'))
    return render_template('index.html', form=form, name=session.get('name'),
                           known=session.get('known', False))
```

The new version of the associated template in `index.html` is shown 
here. This template uses the `known` argument to add a second line to 
the greeting that is different for known and new users:

```html
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

{% block title %}Flasky{% endblock %}

{% block page_content %}
<div class="page-header">
    <h1>Hello, {% if name %}{{ name }}{% else %}Stranger{% endif %}!</h1>
    {% if not known %}
    <p>Pleased to meet you!</p>
    {% else %}
    <p>Happy to see you again!</p>
    {% endif %}
</div>
{{ wtf.quick_form(form) }}
{% endblock %}
```

### Integration with the Python Shell

Having to import the database instance and the models each time a shell 
session is started is tedious work. To avoid having to constantly repeat 
these imports, the Flask-Script's shell command can be configured to 
automatically import certain objects.

To add objects to the import list the shell command needs to be 
registered with a `make_context` callback function. This is shown here:

```python
...
from flask_script import Manager, Shell
...
def make_shell_context():
    return dict(app=app, db=db, User=User, Role=Role)
manager.add_command("shell", Shell(make_context=make_shell_context))
```

The `make_shell_context()` function registers the application and 
database instances and the models so that they are automatically 
imported into the shell:

```python
(flask01) $ python hello.py shell
>>> app
<Flask 'hello'>
>>> db
<SQLAlchemy engine=sqlite:////home/jose-alberto/flasky.git/data.sqlite>
>>> User
<class '__main__.User'>
```

### Database Migrations with Flask-Migrate

As you make progress developing an application, you will find that your 
database models need to change, and when that happens the database needs 
to be updated as well.

Flask-SQLAlchemy creates database tables from models only when they do 
not exist already, so the only way to make it update tables is by 
destroying the old tables first, but of course this causes all the data 
in the database to be lost.

A better solution is to use a *database migration* framework. In the 
same way source code version control tools keep track of changes to 
source code files, a database migration framework keeps track of changes 
to a database *schema*, and then incremental changes can be applied to 
the database.

The lead developer of SQLAlchemy has written a migration framework 
called [Alembic](http://alembic.zzzcomputing.com/en/latest/), but 
instead of using Alembic directly, Flask applications can use 
the [Flask-Migrate](https://flask-migrate.readthedocs.io/en/latest/) 
extension, a lightweight Alembic wrapper that integrates with 
Flask-Script to provide all operations through Flask-Script commands.

**Creating a Migration Repository**

To begin, Flask-Migrate must be installed in the virtual environment:

```
(flask01) $ pip install flask-migrate
```

The following example shows how the extension is initialized:

```python
...
from flask_migrate import Migrate, MigrateCommand
...
migrate = Migrate(app, db)
...
manager.add_command('db', MigrateCommand)
```

To expose the database migration commands, Flask-Migrate exposes 
a `MigrateCommand` class that is attached to Flask-Script's `manager` 
object. In the example above, the command is attached using `db`.

Before database migrations can be maintained, it is necessary to create 
a migration repository with the `init` subcommand:

```
(flask01) $ python hello.py db init
  Creating directory /home/flasky.git/migrations ... done
  Creating directory /home/flasky.git/migrations/versions ... done
  Generating /home/flasky.git/migrations/env.py ... done
  Generating /home/flasky.git/migrations/README ... done
  Generating /home/flasky.git/migrations/script.py.mako ... done
  Generating /home/flasky.git/migrations/alembic.ini ... done
  Please edit configuration/connection/logging settings in 
  '/home/flasky.git/migrations/alembic.ini' before proceeding.
```

This command creates a *migrations* folder, where all the migration 
scripts will be stored.

The files in a database migration repository must always be added to
version control along with the rest of the application.

**Creating a Migration Script**

In Alembic, a database migration is represented by a *migration script*. 
This script has two functions called `upgrade()` and `downgrade()`. 
The `upgrade()` function applies the database changes that are part of 
the migration, and the `downgrade()` function removes them. By having 
the ability to add and remove the changes, Alembic can reconfigure a 
database to any point in the change history.

Alembic migrations can be created manually or automatically using 
the `revision` and `migrate` commands, respectively. A manual migration 
creates a migration skeleton script with empty `upgrade()` 
and `downgrade()` functions that need to be implemented by the developer 
using directives exposed by Alembic's `Operations` object. An automatic 
migration, on the other hand, generates the code for the `upgrade()` 
and `downgrade()` functions by looking for differences between the model 
definitions and the current state of the database.

Automatic migrations are not always accurate and can miss some details. 
Migration scripts generated automatically should always be reviewed.

The `migrate` subcommand creates an automatic migration script:

```
(flask01) $ python hello.py db migrate -m "initial migration"
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added table 'roles'
INFO  [alembic.autogenerate.compare] Detected added table 'users'
INFO  [alembic.autogenerate.compare] Detected added index 'ix_users_username' on '['username']'
Generating /home/flasky.git/migrations/versions/0a7b024d4999_initial_migration.py ... done
```

**Upgrading the Database**

Once a migration script has been reviewed and accepted, it can be 
applied to the database using the `db upgrade` command:

```
(flask01) $ python hello.py db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 0a7b024d4999, initial migration
```

For a first migration, this is effectively equivalent to 
calling `db.create_all()`, but in successive migrations the `upgrade` 
command applies updates to the tables without affecting their contents.

The workflow in the beginning is:

* Delete your `data.sqlite` file.
* Run `(flask01) $ python hello.py db upgrade` to regenerate the 
database through the migration framework.
* Run the migration process `(flask01) $ python hello.py db migrate -m "initial migration"`.
* If the database is changed, run the `db upgrade` command.
