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

The new methods make it easy to create a large number of fake users and 
posts from the Python shell:

```python
(flask01) $ python manage.py shell
>>> User.generate_fake(100)
>>> Post.generate_fake(100)
```

If you run the application now, you will see a long list of random blog 
posts on the home page.

**Rendering Data on Pages**

The following example shows the changes to the home page route to 
support pagination:

```python
@main.route('/', methods=['GET', 'POST'])
def index():
    # ...
    page = request.args.get('page', 1, type=int)
    pagination = Post.query.order_by(Post.timestamp.desc()).paginate(
        page, per_page=current_app.config['FLASKY_POSTS_PER_PAGE'],
        error_out=False)
    posts = pagination.items
    return render_template('index.html', form=form, posts=posts,
                           pagination=pagination)
```

The page number to render is obtained from the request's query string, 
which is available as `request.args`. When an explicit page isn't given, 
a default page of `1` (the first page) is used. The `type=int` argument 
ensures that if the argument cannot be converted to an integer, the 
default value is returned.

To load a single page of records, the call to `all()` is replaced with 
Flask-SQLAlchemy's `paginate()`. The `paginate()` method takes the page 
number as the first and only required argument. An optional `per_page` 
argument can be given to indicate the size of each page, in number of 
items. If this argument is not specified, the default is 20 items
per page. Another optional argument called `error_out` can be set 
to `True` (the default) to issue a code 404 error when a page outside of 
the valid range is requested. If `error_out` is `False`, pages outside 
of the valid range are returned with an empty list of items. To make the 
page sizes configurable, the value of the `per_page` argument is read 
from an application-specific configuration variable 
called `FLASKY_POSTS_PER_PAGE`.

With these changes, the blog post list in the home page will show a 
limited number of items. To see the second page of posts, add 
a `?page=2` query string to the URL in the browser's address bar.

**Adding a Pagination Widget**

The return value of `paginate()` is an object of class `Pagination`, a 
class defined by Flask-SQLAlchemy. This object contains several 
properties that are useful to generate page links in a template, so it 
is passed to the template as an argument. A summary of the attributes of 
the pagination object is shown here:

Attribute | Description
------------ | -------------
`items` | The records in the current page
`query` | The source query that was paginated
`page` | The current page number
`prev_num` | The previous page number
`next_num` | The next page number
`has_next` | `True` if there is a next page
`has_prev` | `True` if there is a previous page
`pages` | The total number of pages for the query
`per_page` | The number of items per page
`total` | The total number of items returned by the query

The pagination object also has some methods, listed here:

Method | Description
------------ | -------------
`iter_pages(left_edge=2, left_current=2, right_current=5, right_edge=2)` | An iterator that returns the sequence of page numbers to display in a pagination widget. The list will have `left_edge` pages on the left side, `left_current` pages to the left of the current page, `right_current` pages to the right of the current page, and `right_edge` pages on the right side. For example, for page 50 of 100 this iterator configured with default values will return the following pages: 1, 2, `None`, 48, 49, 50, 51, 52, 53, 54, 55, `None`, 99, 100. A `None` value in the sequence indicates a gap in the sequence of pages.
`prev()` | A pagination object for the previous page.
`next()` | A pagination object for the next page.

Armed with this powerful object and Bootstrap's pagination CSS classes, 
it is quite easy to build a pagination footer in the template. The 
implementation shown in the following example is done as a reusable 
Jinja2 macro:

```html
{% macro pagination_widget(pagination, endpoint) %}
<ul class="pagination">
    <li{% if not pagination.has_prev %} class="disabled"{% endif %}>
        <a href="{% if pagination.has_prev %}{{ url_for(endpoint, page=pagination.prev_num, **kwargs) }}{% else %}#{% endif %}">
            &laquo;
        </a>
    </li>
    {% for p in pagination.iter_pages() %}
        {% if p %}
            {% if p == pagination.page %}
            <li class="active">
                <a href="{{ url_for(endpoint, page = p, **kwargs) }}">{{ p }}</a>
            </li>
            {% else %}
            <li>
                <a href="{{ url_for(endpoint, page = p, **kwargs) }}">{{ p }}</a>
            </li>
            {% endif %}
        {% else %}
        <li class="disabled"><a href="#">&hellip;</a></li>
        {% endif %}
    {% endfor %}
    <li{% if not pagination.has_next %} class="disabled"{% endif %}>
        <a href="{% if pagination.has_next %}{{ url_for(endpoint, page=pagination.next_num, **kwargs) }}{% else %}#{% endif %}">
            &raquo;
        </a>
    </li>
</ul>
{% endmacro %}
```

The macro creates a Bootstrap pagination element, which is a styled 
unordered list. It defines the following page links inside it:

* A "previous page" link. This link gets the disabled class if the 
current page is the first page.
* Links to the all pages returned by the pagination 
object's `iter_pages()` iterator. These pages are rendered as links with 
an explicit page number, given as an argument to `url_for()`. The page 
currently displayed is highlighted using the `active` CSS class. Gaps in 
the sequence of pages are rendered with the ellipsis character.
* A "next page" link. This link will appear disabled if the current page 
is the last page.

Jinja2 macros always receive keyword arguments without having to 
include `**kwargs` in the argument list. The pagination macro passes all 
the keyword arguments it receives to the `url_for()` call that generates 
the pagination links. This approach can be used with routes such as the 
profile page that have a dynamic part.

The `pagination_widget` macro can be added below the *_posts.html* 
template included by *index.html* and *user.html*. The following example 
shows how it is used in the application's home page:

```html
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}
{% import "_macros.html" as macros %}
...
{% include '_posts.html' %}
{% if pagination %}
<div class="pagination">
    {{ macros.pagination_widget(pagination, '.index') }}
</div>
{% endif %}
{% endblock %}
```

### Rich-Text Posts with Markdown and Flask-PageDown

Plain-text posts are sufficient for short messages and status updates, 
but users who want to write longer articles will find the lack of 
formatting very limiting. In this section, the text area field where 
posts are entered will be upgraded to support 
the [Markdown](https://daringfireball.net/projects/markdown/) syntax and 
present a rich-text preview of the post.

The implementation of this feature requires a few new packages:

* PageDown, a client-side Markdown-to-HTML converter implemented in 
JavaScript.
* Flask-PageDown, a PageDown wrapper for Flask that integrates PageDown 
with Flask-WTF forms.
* Markdown, a server-side Markdown-to-HTML converter implemented in 
Python.
* Bleach, an HTML sanitizer implemented in Python.

The Python packages can all be installed with pip:

```
(flask01) $ pip install flask-pagedown markdown bleach
```

**Using Flask-PageDown**

The Flask-PageDown extension defines a `PageDownField` class that has 
the same interface as the `TextAreaField` from WTForms. Before this 
field can be used, the extension needs to be initialized as shown here:

```python
...
from flask_pagedown import PageDown
...
pagedown = PageDown()
...
def create_app(config_name):
    # ...
    pagedown.init_app(app)
```

To convert the text area control in the home page to a Markdown 
rich-text editor, the `body` field of the `PostForm` must be changed to 
a `PageDownField` as shown here:

```python
...
from flask_pagedown.fields import PageDownField
...
class PostForm(FlaskForm):
    body = PageDownField("What's on your mind?", validators=[Required()])
    submit = SubmitField('Submit')
```

The Markdown preview is generated with the help of the PageDown 
libraries, so these must be added to the template. Flask-PageDown 
simplifies this task by providing a template macro that includes the 
required files from a CDN as shown in the following example:

```html
{% block scripts %}
{{ super() }}
{{ pagedown.include_pagedown() }}
{% endblock %}
```

With these changes, Markdown-formatted text typed in the text area field 
will be immediately rendered as HTML in the preview area below.

**Handling Rich Text on the Server**

When the form is submitted only the raw Markdown text is sent with 
the `POST` request; the HTML preview that was shown on the page is 
discarded. Sending the generated HTML preview with the form can be 
considered a security risk, as it would be fairly easy for an attacker 
to construct HTML sequences that do not match the Markdown source and 
submit them. To avoid any risks, only the Markdown source text is 
submitted, and once in the server it is converted again to HTML 
using *Markdown*, a Python Markdown-to-HTML converter. The resulting 
HTML will be sanitized with *Bleach* to ensure that only a short list of 
allowed HTML tags are used.

The conversion of the Markdown blog posts to HTML can be issued in 
the *_posts.html* template, but this is inefficient, as posts will have 
to be converted every time they are rendered to a page. To avoid this 
repetition, the conversion can be done once when the blog post is 
created. The HTML code for the rendered blog post is *cached* in a new 
field added to the `Post` model that the template can access directly. 
The original Markdown source is also kept in the database in case the 
post needs to be edited. The following example shows the changes to 
the `Post` model:

```python
...
from markdown import markdown
import bleach
...
class Post(db.Model):
    # ...
    body_html = db.Column(db.Text)
    
    # ...
    @staticmethod
    def on_changed_body(target, value, oldvalue, initiator):
        allowed_tags = ['a', 'abbr', 'acronym', 'b', 'blockquote', 'code',
                        'em', 'i', 'li', 'ol', 'pre', 'strong', 'ul',
                        'h1', 'h2', 'h3', 'p']
        target.body_html = bleach.linkify(bleach.clean(
            markdown(value, output_format='html'),
            tags=allowed_tags, strip=True))

db.event.listen(Post.body, 'set', Post.on_changed_body)
```

The `on_changed_body` function is registered as a listener of 
SQLAlchemy's "set" event for `body`, which means that it will be 
automatically invoked whenever the `body` field on any instance of the 
class is set to a new value. The function renders the HTML version of 
the body and stores it in `body_html`, effectively making the conversion 
of the Markdown text to HTML fully automatic.

The actual conversion is done in three steps:

1. First, the `markdown()` function does an initial conversion to HTML.
1. The result is passed to `clean()`, along with the list of approved 
HTML tags. The `clean()` function removes any tags not on the white 
list.
1. The final conversion is done with `linkify()`, another function 
provided by Bleach that converts any URLs written in plain text into 
proper `<a>` links. This last step is necessary because automatic link 
generation is not officially in the Markdown specification. PageDown 
supports it as an extension, so `linkify()` is used in the server to 
match.

The last change is to replace `post.body` with `post.body_html` in the 
template when available, as shown here:

```html
...
<div class="post-body">
    {% if post.body_html %}
        {{ post.body_html | safe }}
    {% else %}
        {{ post.body }}
    {% endif %}
</div>
...
```

The `| safe` suffix when rendering the HTML body is there to tell Jinja2 
not to escape the HTML elements. Jinja2 escapes all template variables 
by default as a security measure. The Markdown-generated HTML was 
generated in the server, so it is safe to render.
