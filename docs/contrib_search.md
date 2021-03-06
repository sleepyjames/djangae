# djangae.contrib.search

This is a minimal search engine built on the Google Cloud Datastore
that in part mimics the old Google App Engine Search API.

It exists for two reasons:

1. An upgrade path from the GAE Search API.
2. To aid smaller projects that don't need to run their own ElasticSearch instance.


# Query Format

The original App Engine Search query format was pretty tricky to parse, and not often used
extensively. The query format used in djangae.contrib.search is much simpler.

## OR operator

You can create a multiple branch search with the OR operator. ORs are not nested, you can simply
use:

```
marathon OR race OR run
```

The OR can be lower case, as 'or' is a common token that is not indexed

## Exact match

You can combine tokens together to return documents that have an exact match by using quotes. e.g.

```
"tallest building"
```

This can in turn be used with the OR operator:

```
"tallest building" OR tower
```

## Field match

Finally, you can use the `:` operator to specify a Document field to search:

```
name:james
```

To combine with the OR operator you'll need to duplicate the `:` operator:

```
name:james OR name:jim OR last_name:kirk
```

You can also combine with exact matching:

```
name:"james kirk" OR name:spock
```

# Documents and Indexes

Contrib search is built around the concept of Indexes and Documents. To make your data searchable,
you must define a Document subclass that defines the structure in search fields:


```python
class MyDocument(search.Document):
    name = search.TextField()
    age = search.NumberField()
```

You can then add the document to an index. Indexes are created automatically in the database when you instantiate
them if they don't already exist:

```python
index = Index(name="my_index")
index.add(MyDocument(name="Lister", age=3000030))
index.add(MyDocument(name="Cat", age=40))
```

Finally, you can search for documents using `index.search`:

```python
results = index.search("lister", MyDocument)
```

The second parameter to `search` is the Document subclass that results
are returned as.


# Field Types

The App Engine Search API had an array of field types. Currently djangae.contrib.search only
supports the following:

 - TextField - A blob of text up to 1024 ** 2 chars in length
 - DateField - A field for storing a Python datetime or date field.
 - NumberField - A field for storing an integer

Fields under construction (do not use!):

 - AtomField - A field
 - FuzzyTextField - A field that supports stemmed matching

# Django Model Integration

It's a very common need to be able to index and search Django model instances. contrib.search
provides a mechanism for this that is similar to the Django admin registration.

First, in your Django app, create a file called `search.py`. Inside `search.py` you define a subclass
of `ModelDocument` (which is simular to Django's `ModelAdmin`), and use that to register your model
as searchable:

```python
from djangae.contrib import search
from .models import MyModel

class MyModelDocument(search.ModelDocument):
    class Meta:
        fields = ("name", "age")


search.register(MyModel, MyModelDocument)
```

This has the following effect:

 - Saving an instance of MyModel will automatically index the name and age fields
 - The managers on MyModel (e.g. MyModel.objects) will gain a `search(...)` method

This means you can search for models in the same way you'd use Django filters:

```python
instances = MyModel.objects.filter(age=10).search("cat")
```

There's no need to specify the Document subclass when searching for models.
