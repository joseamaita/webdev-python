# A Social Blogging Application

## User Profiles

All socially aware sites give their users a profile page, where a 
summary of the user's participation in the website is presented. Users 
can advertise their presence on the website by sharing the URL to their 
profile page, so it is important that the URLs be short and easy to 
remember.

### Profile Information

To make user profile pages more interesting, some additional information 
about users can be recorded. In the following example the `User` model 
is extended with several new fields:

```python
from datetime import datetime
...
class User(UserMixin, db.Model):
    # ...
    name = db.Column(db.String(64))
    location = db.Column(db.String(64))
    about_me = db.Column(db.Text())
    member_since = db.Column(db.DateTime(), default=datetime.utcnow)
    last_seen = db.Column(db.DateTime(), default=datetime.utcnow)
```

The new fields store the user's real name, location, self-written 
description, date of registration, and date of last visit. 
The `about_me` field is assigned the type `db.Text()`. The difference 
between `db.String` and `db.Text` is that `db.Text` does not need a 
maximum length.

The two timestamps are given a default value of the current time. Note 
that `datetime.utcnow` is missing the `()` at the end. This is because 
the `default` argument to `db.Column()` can take a function as a default 
value, so each time a default value needs to be generated the function 
is invoked to produce it. This default value is all that is needed to 
manage the `member_since` field.

The `last_seen` field is also initialized to the current time upon 
creation, but it needs to be refreshed each time the user accesses the 
site. A method in the `User` class can be added to perform this update. 
This is shown here:

```python
class User(UserMixin, db.Model):
    # ...
    
    def ping(self):
        self.last_seen = datetime.utcnow()
        db.session.add(self)
```

The `ping()` method must be called each time a request from the user is 
received. Because the `before_app_request` handler in the `auth` 
blueprint runs before every request, it can do this easily, as shown 
here:

```python
@auth.before_app_request
def before_request():
    if current_user.is_authenticated:
        current_user.ping()
        if not current_user.confirmed \
                and request.endpoint \
                and request.endpoint[:5] != 'auth.' \
                and request.endpoint != 'static':
            return redirect(url_for('auth.unconfirmed'))
```

### User Profile Page

Creating a profile page for each user does not present any new 
challenges. Let's see the route definition 
in *application/main/views.py*:

```python
@main.route('/user/<username>')
def user(username):
    user = User.query.filter_by(username=username).first_or_404()
    return render_template('user.html', user=user)
```

This route is added in the `main` blueprint. For a user named `john`, 
the profile page will be at *http://localhost:5000/user/john*. The 
username given in the URL is searched in the database and, if found, 
the *user.html* template is rendered with it as the argument. An invalid 
username sent into this route will cause a 404 error to be returned. 
The *user.html* template should render the information stored in the 
user object. An initial version of this template is shown here:

```html
{% extends "base.html" %}

{% block title %}Flasky - {{ user.username }}{% endblock %}

{% block page_content %}
<div class="page-header">
    <h1>{{ user.username }}</h1>
    {% if user.name or user.location %}
    <p>
        {% if user.name %}{{ user.name }}{% endif %}
        {% if user.location %}
            From <a href="http://maps.google.com/?q={{ user.location }}">{{ user.location }}</a>
        {% endif %}
    </p>
    {% endif %}
    {% if current_user.is_administrator() %}
    <p><a href="mailto:{{ user.email }}">{{ user.email }}</a></p>
    {% endif %}
    {% if user.about_me %}<p>{{ user.about_me }}</p>{% endif %}
    <p>Member since {{ moment(user.member_since).format('L') }}. Last seen {{ moment(user.last_seen).fromNow() }}.</p>
</div>
{% endblock %}
```

This template has a few interesting implementation details:

* The `name` and `location` fields are rendered inside a single `<p>` 
element. Only when at least one of the fields is defined is the `<p>` 
element created.
* The user `location` field is rendered as a link to a Google Maps 
query.
* If the logged-in user is an administrator, then email addresses are 
shown, rendered as a *mailto* link.

As most users will want easy access to their own profile page, a link to 
it can be added to the navigation bar. The relevant changes to 
the *base.html* template are shown here:

```html
{% if current_user.is_authenticated %}
<li>
    <a href="{{ url_for('main.user', username=current_user.username) }}">Profile</a>
</li>
{% endif %}
```

Using a conditional for the profile page link is necessary because the 
navigation bar is also rendered for nonauthenticated users, in which 
case the profile link is skipped.

### Profile Editor

There are two different use cases related to editing of user profiles. 
The most obvious is that users need to have access to a page where they 
can enter information about themselves to present in their profile 
pages. A less obvious but also important requirement is to let 
administrators edit the profile of any users, not only the personal 
information items but also other fields in the `User` model to which 
users have no direct access, such as the user role. Because the two 
profile editing requirements are substantially different, two different 
forms will be created.

**User-Level Profile Editor**

The profile edit form for regular users is shown here:

```python
class EditProfileForm(FlaskForm):
    name = StringField('Real name', validators=[Length(0, 64)])
    location = StringField('Location', validators=[Length(0, 64)])
    about_me = TextAreaField('About me')
    submit = SubmitField('Submit')
```

Note that as all the fields in this form are optional, the length 
validator allows a length of zero. The route definition that uses this 
form is shown in *application/main/views.py*:

```python
@main.route('/edit-profile', methods=['GET', 'POST'])
@login_required
def edit_profile():
    form = EditProfileForm()
    if form.validate_on_submit():
        current_user.name = form.name.data
        current_user.location = form.location.data
        current_user.about_me = form.about_me.data
        db.session.add(current_user)
        flash('Your profile has been updated.')
        return redirect(url_for('.user', username=current_user.username))
    form.name.data = current_user.name
    form.location.data = current_user.location
    form.about_me.data = current_user.about_me
    return render_template('edit_profile.html', form=form)
```

This view function sets initial values for all the fields before 
presenting the form. For any given field, this is done by assigning the 
initial value to `form.<field-name>.data`. 
When `form.validate_on_submit()` is `False`, the three fields in this 
form are initialized from the corresponding fields in `current_user`. 
Then, when the form is submitted, the `data` attributes of the form 
fields contain the updated values, so these are moved back into the 
fields of the user object and the object is added to the database 
session.

To make it easy for users to reach this page, a direct link can be added 
in the profile page, as shown in *application/templates/user.html*:

```html
{% if user == current_user %}
<a class="btn btn-default" href="{{ url_for('.edit_profile') }}">Edit Profile</a>
{% endif %}
```

The conditional that encloses the link will make the link appear only 
when users are viewing their own profiles.

**Administrator-Level Profile Editor**

The profile edit form for administrators is more complex than the one 
for regular users. In addition to the three profile information fields, 
this form allows administrators to edit a user's email, username, 
confirmed status, and role. The form is shown here:

```python
class EditProfileAdminForm(FlaskForm):
    email = StringField('Email', 
                        validators=[Required(), 
                                    Length(1, 64), 
                                    Email()])
    username = StringField('Username', 
                           validators=[Required(), 
                                       Length(1, 64), 
                                       Regexp('^[A-Za-z][A-Za-z0-9_.]*$', 0, 
                                       'Usernames must have only letters, '
                                       'numbers, dots or underscores')])
    confirmed = BooleanField('Confirmed')
    role = SelectField('Role', coerce=int)
    name = StringField('Real name', validators=[Length(0, 64)])
    location = StringField('Location', validators=[Length(0, 64)])
    about_me = TextAreaField('About me')
    submit = SubmitField('Submit')

    def __init__(self, user, *args, **kwargs):
        super(EditProfileAdminForm, self).__init__(*args, **kwargs)
        self.role.choices = [(role.id, role.name)
                             for role in Role.query.order_by(Role.name).all()]
        self.user = user

    def validate_email(self, field):
        if field.data != self.user.email and \
                User.query.filter_by(email=field.data).first():
            raise ValidationError('Email already registered.')

    def validate_username(self, field):
        if field.data != self.user.username and \
                User.query.filter_by(username=field.data).first():
            raise ValidationError('Username already in use.')
```

The `SelectField` is WTForm's wrapper for the `<select>` HTML form 
control, which implements a dropdown list, used in this form to select a 
user role. An instance of `SelectField` must have the items set in 
its `choices` attribute. They must be given as a list of tuples, with 
each tuple consisting of two values: an identifier for the item and the 
text to show in the control as a string. The `choices` list is set in 
the form's constructor, with values obtained from the `Role` model with 
a query that sorts all the roles alphabetically by name. The identifier 
for each tuple is set to the `id` of each role, and since these are 
integers, a `coerce=int` argument is added to the `SelectField` 
constructor so that the field values are stored as integers instead of 
the default, which is strings.

The `email` and `username` fields are constructed in the same way as in 
the authentication forms, but their validation requires some careful 
handling. The validation condition used for both these fields must first 
check whether a change to the field was made, and only when there is a 
change should it ensure that the new value does not duplicate another 
user's. When these fields are not changed, then validation should pass. 
To implement this logic, the form's constructor receives the user object 
as an argument and saves it as a member variable, which is later used in 
the custom validation methods.

The route definition for the administrator's profile editor is shown 
in *application/main/views.py*:

```python
@main.route('/edit-profile/<int:id>', methods=['GET', 'POST'])
@login_required
@admin_required
def edit_profile_admin(id):
    user = User.query.get_or_404(id)
    form = EditProfileAdminForm(user=user)
    if form.validate_on_submit():
        user.email = form.email.data
        user.username = form.username.data
        user.confirmed = form.confirmed.data
        user.role = Role.query.get(form.role.data)
        user.name = form.name.data
        user.location = form.location.data
        user.about_me = form.about_me.data
        db.session.add(user)
        flash('The profile has been updated.')
        return redirect(url_for('.user', username=user.username))
    form.email.data = user.email
    form.username.data = user.username
    form.confirmed.data = user.confirmed
    form.role.data = user.role_id
    form.name.data = user.name
    form.location.data = user.location
    form.about_me.data = user.about_me
    return render_template('edit_profile.html', form=form, user=user)
```

This route has largely the same structure as the simpler one for regular 
users. In this view function, the user is given by its `id`, so 
Flask-SQLAlchemy's `get_or_404()` convenience function can be used, 
knowing that if the `id` is invalid the request will return a code 404 
error.

The `SelectField` used for the user role also deserves to be studied. 
When setting the initial value for the field, the `role_id` is assigned 
to `field.role.data` because the list of tuples set in the `choices` 
attribute uses the numeric identifiers to reference each option. When 
the form is submitted, the `id` is extracted from the field's `data` 
attribute and used in a query to load the role object by its `id`. 
The `coerce=int` argument used in the `SelectField` declaration in the 
form ensures that the `data` attribute of this field is an integer.

To link to this page, another button is added in the user profile page, 
as shown in *application/templates/user.html*:

```html
{% if current_user.is_administrator() %}
<a class="btn btn-danger" href="{{ url_for('.edit_profile_admin', id=user.id) }}">Edit Profile [Admin]</a>
{% endif %}
```

This button is rendered with a different Bootstrap style to call 
attention to it. The conditional in this case makes the button appear in 
profile pages if the logged-in user is an administrator.

### User Avatars

The look of the profile pages can be improved by showing avatar pictures 
of users. Here, you will learn how to add user avatars provided 
by [Gravatar](http://gravatar.com/), the leading avatar service. 
Gravatar associates avatar images with email addresses. Users create an 
account there and then upload their images. To generate the avatar URL 
for a given email address, its MD5 hash is calculated:

```python
(flask01) $ python
>>> hashlib.md5('john@example.com'.encode('utf-8')).hexdigest()
'd4c74594d841139328695756648b6bd6'
```

The avatar URLs are then generated by appending the MD5 hash to 
URL *http://www.gravatar.com/avatar/* 
or *https://secure.gravatar.com/avatar/*. For example, you can 
type *http://www.gravatar.com/avatar/d4c74594d841139328695756648b6bd6* 
in your browser's address bar to get the avatar image for the email 
address `john@example.com`, or a default generated image if that email 
address does not have an avatar registered. The query string of the URL 
can include several arguments that configure the characteristics of the 
avatar image, listed here:

Argument name | Description
------------ | -------------
`s` | Image size, in pixels
`r` | Image rating. Options are `"g"`, `"pg"`, `"r"`, and `"x"`.
`d` | The default image generator for users who have no avatars registered with the Gravatar service. Options are `"404"` to return a 404 error, a URL that points to a default image, or one of the following image generators: `"mm"`, `"identicon"`, `"monsterid"`, `"wavatar"`, `"retro"`, or `"blank"`.
`fd` | Force the use of default avatars.

The knowledge of how to build a Gravatar URL can be added to the `User` 
model. The implementation is shown here:

```python
...
import hashlib
from flask import request


class User(UserMixin, db.Model):
    # ...
    
    
    def gravatar(self, size=100, default='identicon', rating='g'):
        if request.is_secure:
            url = 'https://secure.gravatar.com/avatar'
        else:
            url = 'http://www.gravatar.com/avatar'
        hash = hashlib.md5(self.email.encode('utf-8')).hexdigest()
        return '{url}/{hash}?s={size}&d={default}&r={rating}'.format(
            url=url, hash=hash, size=size, default=default, rating=rating)    
```

This implementation selects the standard or secure Gravatar base URL to 
match the security of the client request. The avatar URL is generated 
from the base URL, the MD5 hash of the user's email address, and the 
arguments, all of which have default values. With this implementation it 
is easy to generate avatar URLs in the Python shell:

```python
(flask01) $ python manage.py shell
>>> u = User(email='john@example.com')
>>> u.gravatar()
'http://www.gravatar.com/avatar/d4c74594d841139328695756648b6bd6?s=100&d=identicon&r=g'
>>> u.gravatar(size=256)
'http://www.gravatar.com/avatar/d4c74594d841139328695756648b6bd6?s=256&d=identicon&r=g'
```

The `gravatar()` method can also be invoked from Jinja2 templates. The 
following example shows how a 256-pixel avatar can be added to the 
profile page:

```html
...
<img class="img-rounded profile-thumbnail" src="{{ user.gravatar(size=256) }}">
...
```

Using a similar approach, the base template adds a small thumbnail image 
of the logged-in user in the navigation bar. To better format the avatar 
pictures in the page, custom CSS classes are used. You can find these in 
the source code repository in a *styles.css* file added to the 
application's static file folder and referenced from the *base.html* 
template.

The generation of avatars requires an MD5 hash to be generated, which is 
a CPU-intensive operation. If a large number of avatars need to be 
generated for a page, then the computational work can be significant. 
Since the MD5 hash for a user will remain constant, it can be *cached* 
in the `User` model. The following example shows the changes to 
the `User` model to store the MD5 hashes in the database:

```python
class User(UserMixin, db.Model):
    # ...
    avatar_hash = db.Column(db.String(32))
    
    def __init__(self, **kwargs):
        # ...
        if self.email is not None and self.avatar_hash is None:
            self.avatar_hash = hashlib.md5(
                self.email.encode('utf-8')).hexdigest()
    
    def change_email(self, token):
        # ...
        self.email = new_email
        self.avatar_hash = hashlib.md5(
            self.email.encode('utf-8')).hexdigest()
        db.session.add(self)
        return True

    def gravatar(self, size=100, default='identicon', rating='g'):
        if request.is_secure:
            url = 'https://secure.gravatar.com/avatar'
        else:
            url = 'http://www.gravatar.com/avatar'
        hash = self.avatar_hash or hashlib.md5(
            self.email.encode('utf-8')).hexdigest()
        return '{url}/{hash}?s={size}&d={default}&r={rating}'.format(
            url=url, hash=hash, size=size, default=default, rating=rating)
```

During model initialization, the hash is calculated from the email and 
stored, and in the event that the user updates the email address the 
hash is recalculated. The `gravatar()` method uses the hash from the 
model if available; if not, it works as before and generates the hash 
from the email address.
