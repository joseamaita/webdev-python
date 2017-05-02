# A Social Blogging Application

## User Roles

Not all users of web applications are created equal. In most 
applications, a small percentage of users are trusted with extra powers 
to help keep the application running smoothly. Administrators are the 
best example, but in many cases middle-level power users such as content 
moderators exist as well.

There are several ways to implement roles in an application. The 
appropriate method largely depends on how many roles need to be 
supported and how elaborate they are. For example, a simple application 
may need just two roles, one for regular users and one for 
administrators. In this case, having an `is_administrator` Boolean field 
in the `User` model may be all that is necessary. A more complex 
application may need additional roles with varying levels of power in 
between regular users and administrators. In some applications it may 
not even make sense to talk about discrete roles; instead, giving users 
a combination of *permissions* may be the right approach.

The user role implementation presented in this part is a hybrid between 
discrete roles and permissions. Users are assigned a discrete role, but 
the roles are defined in terms of permissions.

### Database Representation of Roles

The following example shows an improved `Role` model with some 
additions:

```
class Role(db.Model):
    __tablename__ = 'roles'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(64), unique=True)
    default = db.Column(db.Boolean, default=False, index=True)
    permissions = db.Column(db.Integer)
    users = db.relationship('User', backref='role', lazy='dynamic')
```

The `default` field should be set to `True` for only one role 
and `False` for all the others. The role marked as default will be the 
one assigned to new users upon registration.

The second addition to the model is the `permissions` field, which is an 
integer that will be used as bit flags. Each task will be assigned a bit 
position, and for each role the tasks that are allowed for that role 
will have their bits set to 1.

The list of tasks for which permissions are needed is obviously 
application specific. For Flasky, the list of tasks is shown here:

Task name | Bit value | Description
------------ | ------------- | -------------
Follow users | 0b00000001 (0x01) | Follow other users
Comment on posts made by others | 0b00000010 (0x02) | Comment on articles written by others
Write articles | 0b00000100 (0x04) | Write original articles
Moderate comments made by others | 0b00001000 (0x08) | Suppress offensive comments made by others
Administration access | 0b10000000 (0x80) | Administrative access to the site

Note that a total of eight bits was allocated to tasks, and so far only 
five have been used. The remaining three are left for future expansion.

The code representation of the previous table is shown here:

```python
class Permission:
    FOLLOW = 0x01
    COMMENT = 0x02
    WRITE_ARTICLES = 0x04
    MODERATE_COMMENTS = 0x08
    ADMINISTER = 0x80
```

The following table shows the list of user roles that will be supported, 
along with the permission bits that define it:

User role | Permissions | Description
------------ | ------------- | -------------
Anonymous | 0b00000000 (0x00) | User who is not logged in. Read-only access to the application.
User | 0b00000111 (0x07) | Basic permissions to write articles and comments and to follow other users. This is the default for new users.
Moderator | 0b00001111 (0x0f) | Adds permission to suppress comments deemed offensive or inappropriate.
Administrator | 0b11111111 (0xff) | Full access, which includes permission to change the roles of other users.

Organizing the roles with permissions lets you add new roles in the 
future that use different combinations of permissions.

Adding the roles to the database manually is time consuming and error 
prone. Instead, a class method will be added to the `Role` class for 
this purpose, as shown here:

```python
class Role(db.Model):
    # ...
    @staticmethod
    def insert_roles():
        roles = {
            'User': (Permission.FOLLOW |
                     Permission.COMMENT |
                     Permission.WRITE_ARTICLES, True),
            'Moderator': (Permission.FOLLOW |
                          Permission.COMMENT |
                          Permission.WRITE_ARTICLES |
                          Permission.MODERATE_COMMENTS, False),
            'Administrator': (0xff, False)
        }
        for r in roles:
            role = Role.query.filter_by(name=r).first()
            if role is None:
                role = Role(name=r)
            role.permissions = roles[r][0]
            role.default = roles[r][1]
            db.session.add(role)
        db.session.commit()
```

The `insert_roles()` function does not directly create new role objects. 
Instead, it tries to find existing roles by name and update those. A new 
role object is created only for role names that aren't in the database 
already. This is done so that the role list can be updated in the future 
when changes need to be made. To add a new role or change the permission 
assignments for a role, change the `roles` array and rerun the function. 
Note that the "Anonymous" role does not need to be represented in the 
database, as it is designed to represent users who are not in the 
database.

To apply these roles to the database, a shell session can be used:

```python
(flask01) $ python manage.py db migrate -m "user roles"
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added column 'roles.default'
INFO  [alembic.autogenerate.compare] Detected added column 'roles.permissions'
INFO  [alembic.autogenerate.compare] Detected added index 'ix_roles_default' on '['default']'
  Generating /home/flasky.git/migrations/versions/7baeeee0c01e_user_roles.py ... done

(flask01) $ python manage.py db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade b27e33ecdb0b -> 7baeeee0c01e, user roles
/home/jose-alberto/.virtualenvs/flask01/lib/python3.6/site-packages/alembic/util/messaging.py:69: UserWarning: Skipping unsupported ALTER for creation of implicit constraint
  warnings.warn(msg)

(flask01) $ python manage.py shell
>>> Role.insert_roles()
>>> Role.query.all()
[<Role 'User'>, <Role 'Moderator'>, <Role 'Administrator'>]
```

### Role Assignment

When users register an account with the application, the correct role 
should be assigned to them. For most users, the role assigned at 
registration time will be the "User" role, as that is the role that is 
marked as a default role. The only exception is made for the 
administrator, which needs to be assigned the "Administrator" role from 
the start. This user is identified by an email address stored in 
the `FLASKY_ADMIN` configuration variable, so as soon as that email 
address appears in a registration request it can be given the correct 
role. The following example shows how this is done in the `User` model 
constructor:

```python
class User(UserMixin, db.Model):
    # ...
    def __init__(self, **kwargs):
        super(User, self).__init__(**kwargs)
        if self.role is None:
            if self.email == current_app.config['FLASKY_ADMIN']:
                self.role = Role.query.filter_by(permissions=0xff).first()
            if self.role is None:
                self.role = Role.query.filter_by(default=True).first()
    # ...
```

The `User` constructor first invokes the constructors of the base 
classes, and if after that the object does not have a role defined, it 
sets the administrator or default roles depending on the email address.

### Role Verification

To simplify the implementation of roles and permissions, a helper method 
can be added to the `User` model that checks whether a given permission 
is present, as shown here:

```python
...
from flask_login import UserMixin, AnonymousUserMixin
...
class User(UserMixin, db.Model):
    # ...
    
    def can(self, permissions):
        return self.role is not None and \
            (self.role.permissions & permissions) == permissions

    def is_administrator(self):
        return self.can(Permission.ADMINISTER)


class AnonymousUser(AnonymousUserMixin):
    def can(self, permissions):
        return False

    def is_administrator(self):
        return False

login_manager.anonymous_user = AnonymousUser
```

The `can()` method added to the `User` model performs a *bitwise and* 
operation between the requested permissions and the permissions of the 
assigned role. The method returns `True` if all the requested bits are 
present in the role, which means that the user should be allowed to 
perform the task. The check for administration permissions is so common 
that it is also implemented as a standalone `is_administrator()` method.

For consistency, a custom `AnonymousUser` class that implements 
the `can()` and `is_administrator()` methods is created. This object 
inherits from Flask-Login's `AnonymousUserMixin` class and is registered 
as the class of the object that is assigned to `current_user` when the 
user is not logged in. This will enable the application to freely 
call `current_user.can()` and `current_user.is_administrator()` without 
having to check whether the user is logged in first.

For cases in which an entire view function needs to be made available 
only to users with certain permissions, a custom decorator can be used. 
The following example in *application/decorators.py* shows the 
implementation of two decorators, one for generic permission checks and 
one that checks specifically for administrator permission.

```python
from functools import wraps
from flask import abort
from flask_login import current_user
from .models import Permission


def permission_required(permission):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if not current_user.can(permission):
                abort(403)
            return f(*args, **kwargs)
        return decorated_function
    return decorator


def admin_required(f):
    return permission_required(Permission.ADMINISTER)(f)
```

These decorators are built with the help of the *functools* package from 
the Python standard library, and return an error code 403, the 
"Forbidden" HTTP error, when the current user does not have the 
requested permissions. Previously, custom error pages were created for 
errors 404 and 500, so now a page for the 403 error needs to be added as 
well.

The following are two examples that demonstrate the usage of these 
decorators:

```python
from decorators import admin_required, permission_required

@main.route('/admin')
@login_required
@admin_required
def for_admins_only():
    return "For administrators!"

@main.route('/moderator')
@login_required
@permission_required(Permission.MODERATE_COMMENTS)
def for_moderators_only():
    return "For comment moderators!"
```

Permissions may also need to be checked from templates, so 
the `Permission` class with all the bit constants needs to be accessible 
to them. To avoid having to add a template argument in 
every `render_template()` call, a *context processor* can be used. 
Context processors make variables globally available to all templates. 
This change is shown in *application/main/__init__.py*:

```python
...
from ..models import Permission


@main.app_context_processor
def inject_permissions():
    return dict(Permission=Permission)
```

The new roles and permissions can be exercised in unit tests. The 
following example shows two simple tests that also serve as a 
demonstration of the usage:

```python
class UserModelTestCase(unittest.TestCase):
    # ...
    def test_roles_and_permissions(self):
        u = User(email='john@example.com', password='cat')
        self.assertTrue(u.can(Permission.WRITE_ARTICLES))
        self.assertFalse(u.can(Permission.MODERATE_COMMENTS))

    def test_anonymous_user(self):
        u = AnonymousUser()
        self.assertFalse(u.can(Permission.FOLLOW))
```

To re-create or update the development database so that all the user 
accounts that were created before roles and permissions existed have a 
role assigned, do this:

* Delete the current development database.
* Run the upgrade process:

```
(flask01)$ python manage.py db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 0a7b024d4999, initial migration
INFO  [alembic.runtime.migration] Running upgrade 0a7b024d4999 -> fb95e2893a13, Initial migration
INFO  [alembic.runtime.migration] Running upgrade fb95e2893a13 -> b27e33ecdb0b, account confirmation
/home/jose-alberto/.virtualenvs/flask01/lib/python3.6/site-packages/alembic/util/messaging.py:69: UserWarning: Skipping unsupported ALTER for creation of implicit constraint
  warnings.warn(msg)
INFO  [alembic.runtime.migration] Running upgrade b27e33ecdb0b -> 7baeeee0c01e, user roles
```

* Add the accounts using the form in the web application.
* Insert the roles in the database using the shell:

```python
(flask01)$ python manage.py shell
>>> Role.insert_roles()
>>> Role.query.all()
[<Role 'User'>, <Role 'Moderator'>, <Role 'Administrator'>]
```

* Update existing users and give them a role using the shell:

```python
(flask01)$ python manage.py shell
>>> user1 = User.query.filter_by(username='Jose').first()
>>> user1
<User 'Jose'>
>>> user1.role = Role.query.filter_by(default=True).first()
>>> user1.role
<Role 'User'>
>>> db.session.add(user1.role)
>>> db.session.commit()
```

* Check briefly the assignments by typing:

```
(flask01)$ sqlite3 data-dev.sqlite
SQLite version 3.8.7.1 2014-10-29 13:59:56
Enter ".help" for usage hints.
sqlite> select * from roles;
1|User|1|7
2|Moderator|0|15
3|Administrator|0|255
sqlite> select * from users;
1|Robert|3|bob@example.com|pbkdf2:sha256:50000$5qtGsUDw$...|0
2|David|2|dave@mailexample.com|pbkdf2:sha256:50000$DPeIteLT$...|0
3|Jose|1|jose@email.com|pbkdf2:sha256:50000$yYqOIulM$...|0
```
