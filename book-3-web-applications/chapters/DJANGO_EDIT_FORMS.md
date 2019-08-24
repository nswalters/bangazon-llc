# Editing Resources

It's time to revisit editing resources in a database via user interactions in the browser - everyone's favorite part of NSS. ❤️

Here's the process, which is slightly different that doing it in a single page application like you did in the client-side course.

## Process for Editing a Resource

1. Place an edit button on the detail view of the resource by modifying the template. The button should be in a form along with a hidden input field specifying the action.
1. Code in the view will check for the hidden input field of `actual_method` and see if its value is "EDIT".
1. The view will then query the database for the single record to be edited.
1. The server will return a template, with the single resource object in its context.
1. The template will populate the `value` attribute of all of the input fields with their corresponding value from the context's resource object.
1. Once the user has changed the values, and clicks the Update button, the form will be submitted to the detail view for the resource.
1. The view will then check for the `actual_method` field again, and see if its value is "PUT".
1. Then the view will perform an UPDATE statement in the database, and then redirect the user back to the detail view.

This is a difficult signal flow to visualize, and takes months _(sometimes years)_ of practice before it feels comfortable. You won't get it right the first, second, third, or fourth time you try it. Maybe on the 15th time.

Let's start.

### The Edit Button

Add the following code to your `templates/books/detail.html` template.

```html
<form action="{% url 'libraryapp:book_details' book.id %}" method="POST">
    {% csrf_token %}
    <input type="hidden" name="actual_method" value="EDIT">
    <button>Edit</button>
</form>
```

The `url 'libraryapp:book_details'` part of the action looks in the `urls.py` file and generates the matching URL pattern for that named view. In your application, it matches this named route.

```py
url(r'^books/(?P<book_id>[0-9]+)/$', book_details, name="book_details"),
```

So Django will build the string `/books`. It's not done. That URL pattern also has `(?P<book_id>[0-9]+)` in it, which is the route parameter. Therefore, you have to pass in the id of the book, which is the third part of the `action` method for your form. Django will then put that id at the end of the generated URL - `/books/12`

This is the final HTML that gets generated if you clicked on the edit button for a book whose primary key in the database is 12.

```html
<form action="/books/12/" method="POST">
    <input type="hidden" name="csrfmiddlewaretoken" value="...">
    <input type="hidden" name="actual_method" value="EDIT">
    <button>Edit</button>
</form>
```

### Refactor Details View

When the user wants to edit a book, all we have in the request is the `id` of the book since it's in the URL as a route parameter. This means we need to go to the database to get all the rest of the values. In `views/books/details.py` you already have SQL code that queries the database for a single book. It's in the `GET` handler, so instead of writing duplicate code, you are going to add a new function named `get_book()` that both the `GET` handler and the `EDIT` handler can both use.

Add the following function to `details.py` above the `book_details` function.

```py
def get_book(book_id):
    with sqlite3.connect(Connection.db_path) as conn:
        conn.row_factory = model_factory(Book)
        db_cursor = conn.cursor()

        db_cursor.execute("""
        SELECT
            b.id,
            b.title,
            b.isbn,
            b.author,
            b.year_published,
            b.librarian_id,
            b.location_id
        FROM libraryapp_book b
        WHERE b.id = ?
        """, (book_id,))

        return db_cursor.fetchone()
```

Then refactor the `book_details()` function to use it.

```py
@login_required
def book_details(request, book_id):
    if request.method == 'GET':
        book = get_book(book_id)
        return render(request, 'books/detail.html', {'book': book})

```

### Returning the Edit Form

Now it's time to handle the request to edit the book. Since this request is a POST, you need to add another `if` condition to handle the case in which the user clicked the Edit button. The only way to know if the user clicked the edit button is to look for the input field named `actual_method` and see if its value is EDIT.

```py
if request.method == 'POST':
    form_data = request.POST

    # Check if this request is for deleting a book
    if (
        "actual_method" in form_data
        and form_data["actual_method"] == "DELETE"
    ):
        with sqlite3.connect(Connection.db_path) as conn:
            db_cursor = conn.cursor()

            db_cursor.execute("""
            DELETE FROM libraryapp_book
            WHERE id = ?
            """, (book_id,))

        return redirect(reverse('libraryapp:list_books'))

    # Check if the request is for editing a book
    if (
        "actual_method" in form_data
        and form_data["actual_method"] == "EDIT"
    ):
        book = get_book(book_id)
        libraries = get_libraries()

        return render(request, 'books/form.html', {
            'book': book,
            'all_libraries': libraries
        })
```

Notice the the template - `books/form.html` - is the same template you used for creating a new book. What's different in this case is that we are providing a context dictionary to the template with a `book` key and an `all_libraries` key.

### Create and Edit Form

Since you are now sending the `form.html` template to the client in two different cases...

* When a book is being created
* When a book is being edited

...you need to modify the template. Replace your template with the code below. In this version, note that the `value` attribute of each of the input fields interpolates the corresponding property on a Book object.

In the dropdown, there is an inline `{% if %}` statement that pre-selects the library that the book is assigned to.

Then at the end, a hidden input field is added **only** when the form is being used for edit. This hidden input field will store the `id` property of the book, and will be used by the view when posted.

```html
{% load staticfiles %}
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Bangazon</title>
  </head>
  <body>
    <h1>Inventory Book</h1>

    <form action="{% url 'libraryapp:create_book' %}" method="post">
      {% csrf_token %}
      <fieldset>
          <label for="title">Title: </label>
          <input id="title" type="text" name="title" value="{{ book.title }}">
      </fieldset>
      <fieldset>
          <label for="author">Author: </label>
          <input id="author" type="text" name="author" value="{{ book.author }}">
      </fieldset>
      <fieldset>
          <label for="year_published">Year of publication: </label>
          <input id="year_published" type="number" name="year_published" value="{{ book.year_published }}">
      </fieldset>
      <fieldset>
          <label for="isbn">ISBN: </label>
          <input id="isbn" type="text" name="isbn" value="{{ book.isbn }}">
      </fieldset>
      <fieldset>
          <label for="location">Library: </label>
          <select id="location" type="text" name="location">
                {% for library in all_libraries %}
                    <option
                        {% if library.id == book.location_id %}
                            selected
                        {% endif %}
                        value="{{ library.id }}">
                        {{ library.title }}
                    </option>
                {% endfor %}
          </select>
      </fieldset>

      {% if book.id is not None %}
        <input type="hidden" name="book_id" value="{{ book.id }}">
        <input type="submit" value="Update" />
      {% else %}
        <input type="submit" value="Create" />
      {% endif %}

    </form>
  </body>
</html>
```


Python is very forgiving. If the `book` object is not in the context dictionary for the template, then the form fields will not be populated with a value, even though the code explicitly accesses a property on an object that does not exist. You might expect an exception to be thrown, but it does not.