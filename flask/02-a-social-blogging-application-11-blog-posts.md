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
