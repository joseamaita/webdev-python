# A Social Blogging Application

## Blog Posts

This part is dedicated to the implementation of Flasky's main feature, 
which is to allow users to read and write blog posts. Here you will 
learn a few new techniques for reuse of templates, pagination of long 
lists of items, and working with rich text.

### Blog Post Submission and Display

To support blog posts, a new database model that represents them is 
necessary. This model is shown here:

```python
class Post(db.Model):
    __tablename__ = 'posts'
    id = db.Column(db.Integer, primary_key=True)
    body = db.Column(db.Text)
    timestamp = db.Column(db.DateTime, index=True, default=datetime.utcnow)
    author_id = db.Column(db.Integer, db.ForeignKey('users.id'))

class User(UserMixin, db.Model):
    # ...
    posts = db.relationship('Post', backref='author', lazy='dynamic')
```

A blog post is represented by a body, a timestamp, and a one-to-many 
relationship from the `User` model. The `body` field is defined with 
type `db.Text` so that there is no limitation on the length.

The form that will be shown in the main page of the application lets 
users write a blog post. This form is very simple; it contains just a 
text area where the blog post can be typed and a submit button. The form 
definition is shown here:

```python
class PostForm(FlaskForm):
    body = TextAreaField("What's on your mind?", validators=[Required()])
    submit = SubmitField('Submit')
```

The `index()` view function handles the form and passes the list of old 
blog posts to the template, as shown here:

```python
...
from .forms import PostForm
...
from ..models import Permission, Post
...

@main.route('/', methods=['GET', 'POST'])
def index():
    form = PostForm()
    if current_user.can(Permission.WRITE_ARTICLES) and \
            form.validate_on_submit():
        post = Post(body=form.body.data,
                    author=current_user._get_current_object())
        db.session.add(post)
        return redirect(url_for('.index'))
    posts = Post.query.order_by(Post.timestamp.desc()).all()
    return render_template('index.html', form=form, posts=posts)
```

This view function passes the form and the complete list of blog posts 
to the template. The list of posts is ordered by their timestamp in 
descending order. The blog post form is handled in the usual manner, 
with the creation of a new `Post` instance when a valid submission is 
received. The current user's permission to write articles is checked 
before allowing the new post.

Note the way the `author` attribute of the new post object is set to the 
expression `current_user._get_current_object()`. The `current_user` 
variable from Flask-Login, like all context variables, is implemented as 
a thread-local proxy object. This object behaves like a user object but 
is really a thin wrapper that contains the actual user object inside. 
The database needs a real user object, which is obtained by 
calling `_get_current_object()`.

The form is rendered below the greeting in the *index.html* template, 
followed by the blog posts. The list of blog posts is a first attempt to 
create a blog post timeline, with all the blog posts in the database 
listed in chronological order from newest to oldest. The changes to the 
template are shown here:

```html
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

{% block title %}Flasky{% endblock %}

{% block page_content %}
<div class="page-header">
    <h1>Hello, 
    {% if current_user.is_authenticated %}
    {{ current_user.username }}
    {% else %}
    Stranger
    {% endif %}!
    </h1>
</div>
<div>
    {% if current_user.can(Permission.WRITE_ARTICLES) %}
    {{ wtf.quick_form(form) }}
    {% endif %}
</div>
<ul class="posts">
    {% for post in posts %}
    <li class="post">
        <div class="post-thumbnail">
            <a href="{{ url_for('.user', 
                                username=post.author.username) }}">
                <img class="img-rounded profile-thumbnail" 
                src="{{ post.author.gravatar(size=40) }}">
            </a>
        </div>
        <div class="post-content">
            <div class="post-date">
            {{ moment(post.timestamp).fromNow() }}</div>
            <div class="post-author"><a href="{{ url_for('.user', 
            username=post.author.username) }}">
            {{ post.author.username }}</a></div>
            <div class="post-body">{{ post.body }}</div>
        </div>
    </li>
    {% endfor %}
</ul>
{% endblock %}
```

Note that the `User.can()` method is used to skip the blog post form for 
users who do not have the `WRITE_ARTICLES` permission in their role. The 
blog post list is implemented as an HTML unordered list, with CSS 
classes giving it nicer formatting. A small avatar of the author is 
rendered on the left side, and both the avatar and the author's username 
are rendered as links to the user profile page. The CSS styles used are 
stored in a *styles.css* file in the application's *static* folder.

To update the database with the `Post` model, do this:

```
(flask01) $ python manage.py db migrate -m "post model"
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added table 'posts'
INFO  [alembic.autogenerate.compare] Detected added index 'ix_posts_timestamp' on '['timestamp']'
  Generating /home/flasky/flasky.git/migrations/versions/dd05f8e45e7b_post_model.py ... done
(flask01) $ python manage.py db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade dee7999c04f7 -> dd05f8e45e7b, post model
```

### Blog Posts on Profile Pages

The user profile page can be improved by showing a list of blog posts 
authored by the user. The following example shows the changes to the 
view function at *application/main/views.py* to obtain the post list:

```python
@main.route('/user/<username>')
def user(username):
    user = User.query.filter_by(username=username).first_or_404()
    posts = user.posts.order_by(Post.timestamp.desc()).all()
    return render_template('user.html', user=user, posts=posts)
```

The list of blog posts for a user is obtained from the `User.posts` 
relationship, which is a query object, so filters such as `order_by()` 
can be used on it as well.

The *user.html* template requires the `<ul>` HTML tree that renders a 
list of blog posts like the one in *index.html*. Having to maintain two 
identical copies of a piece of HTML is not ideal, so for cases like 
this, Jinja2's `include()` directive is very useful. The *user.html* 
template includes the list from an external file, as shown in the 
following example:

```python
...
<p>{{ user.posts.count() }} blog posts.</p>
...
...
<h3>Posts by {{ user.username }}</h3>
{% include '_posts.html' %}
...
```

To complete this reorganization, the `<ul>` tree from *index.html* is 
moved to the new template *_posts.html*, and replaced with 
another `include()` directive. Note that the use of an underscore prefix 
in the `_posts.html` template name is not a requirement; this is merely 
a convention to distinguish standalone and partial templates.

### Paginating Long Blog Post Lists

As the site grows and the number of blog posts increases, it will become 
slow and impractical to show the complete list of posts on the home and 
profile pages. Big pages take longer to generate, download, and render 
in the web browser, so the quality of the user experience decreases as 
the pages get larger. The solution is to paginate the data and render it 
in chunks.

**Creating Fake Blog Post Data**

To be able to work with multiple pages of blog posts, it is necessary to 
have a test database with a large volume of data. Manually adding new 
database entries is time consuming and tedious; an automated solution is 
more appropriate. There are several Python packages that can be used to 
generate fake information. A fairly complete one is *ForgeryPy*, which 
is installed with pip:

```
(flask01) $ pip install forgerypy
```

The ForgeryPy package is not, strictly speaking, a dependency of the 
application, because it is needed only during development. To separate 
the production dependencies from the development dependencies, 
the *requirements.txt* file can be replaced with a *requirements* folder 
that stores different sets of dependencies. Inside this new folder 
a *dev.txt* file can list the dependencies that are necessary for 
development and a *prod.txt* file can list the dependencies that are 
needed in production. As there is a large number of dependencies that 
will be in both lists, a *common.txt* file is added for those, and then 
the *dev.txt* and *prod.txt* lists use the `-r` prefix to include it. 
The following example shows the *dev.txt* file:

```txt
-r common.txt
ForgeryPy==0.1
```

The following example shows class methods added to the `User` and `Post` 
models that can generate fake data at *application/models.py*:

```python
class User(UserMixin, db.Model):
    # ...
    
    @staticmethod
    def generate_fake(count=100):
        from sqlalchemy.exc import IntegrityError
        from random import seed
        import forgery_py

        seed()
        for i in range(count):
            u = User(email=forgery_py.internet.email_address(),
                     username=forgery_py.internet.user_name(True),
                     password=forgery_py.lorem_ipsum.word(),
                     confirmed=True,
                     name=forgery_py.name.full_name(),
                     location=forgery_py.address.city(),
                     about_me=forgery_py.lorem_ipsum.sentence(),
                     member_since=forgery_py.date.date(True))
            db.session.add(u)
            try:
                db.session.commit()
            except IntegrityError:
                db.session.rollback()

class Post(db.Model):
    # ...
    
    @staticmethod
    def generate_fake(count=100):
        from random import seed, randint
        import forgery_py

        seed()
        user_count = User.query.count()
        for i in range(count):
            u = User.query.offset(randint(0, user_count - 1)).first()
            p = Post(body=forgery_py.lorem_ipsum.sentences(randint(1, 5)),
                     timestamp=forgery_py.date.date(True),
                     author=u)
            db.session.add(p)
            db.session.commit()
```

The attributes of these fake objects are generated with ForgeryPy random 
information generators, which can generate real-looking names, emails, 
sentences, and many more attributes.

The email addresses and usernames of users must be unique, but since 
ForgeryPy generates these in completely random fashion, there is a risk 
of having duplicates. In this unlikely event, the database session 
commit will throw an `IntegrityError` exception. This exception is 
handled by rolling back the session before continuing. The loop 
iterations that produce a duplicate will not write a user to the 
database, so the total number of fake users added can be less than the 
number requested.

The random post generation must assign a random user to each post. For 
this the `offset()` query filter is used. This filter discards the 
number of results given as an argument. By setting a random offset and 
then calling `first()`, a different random user is obtained each time.
