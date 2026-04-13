---
title: 5 Django ORM Performance Problems and How to Solve Them
date: 2026-04-10
draft: false
tags: [django, python, performance, orm, database, best-practices]
categories: [django, performance, coding-tips]
description: Learn the 5 most common performance problems in Django ORM applications and their proven solutions with practical code examples and detailed explanations.
cover: django-performance-orm.png
---

As Django developers, we often focus on making our applications work correctly. We write clean code, follow best practices, and test thoroughly. But there's a silent killer that can bring even the most elegant code to its knees: **database performance**.

In this comprehensive guide, I'll walk you through the **5 most common performance problems** in Django ORM. For each problem, I'll explain:

- **Why** it happens (the root cause)
- **How** to identify it in your code
- **What** happens in the database
- **The exact solution** with code examples

Let's dive in!

______________________________________________________________________

## The 5 Performance Problems at a Glance

| # | Problem | Solution | Queries (Slow) | Queries (Fast) | Impact |
|---|---------|-----------|----------------|----------------|--------|
| 1 | N+1 Query (ForeignKey) | `select_related()` | N+1 | 1 | 🔴 Critical |
| 2 | N+1 Query (ManyToMany) | `prefetch_related()` | N+1 | 2 | 🔴 Critical |
| 3 | SELECT * unnecessary | `only()` / `defer()` | 1 | 1 | 🟡 Medium |
| 4 | len() vs count() | `.count()` | 1 | 2 | 🟡 Medium |
| 5 | Missing pagination | `Paginator` | All | Page size | 🔴 Critical |

______________________________________________________________________

## Problem 1: N+1 Query with ForeignKey

### 🔍 What Is It?

The N+1 query problem is the **most common and most damaging** performance issue in Django applications. It occurs when you fetch a list of objects and then access a ForeignKey relationship for each one.

### 🧠 Why Does It Happen?

When you call `Book.objects.all()`, Django executes:

```sql
SELECT * FROM book;
```

This returns 50 books. But here's the thing: **Django is lazy**. It doesn't load the related `author` data until you actually access it.

When your template or code does:

```html
{% for book in books %}
    {{ book.author.name }}  <!-- Accessing ForeignKey here -->
{% endfor %}
```

Django thinks: "Oh, you need the author now? Let me fetch it." So it runs:

```sql
SELECT * FROM author WHERE id = 1;
SELECT * FROM author WHERE id = 2;
SELECT * FROM author WHERE id = 3;
-- ... and so on for each book!
```

### 📊 The Math

| Books | SQL Queries (Slow) | SQL Queries (Fast) |
|-------|-------------------|-------------------|
| 10 | 11 | 1 |
| 50 | 51 | 1 |
| 100 | 101 | 1 |
| 1,000 | 1,001 | 1 |
| 10,000 | 10,001 | 1 |

With just 1,000 books, you're making **1,001 database round trips**! Each round trip has latency (typically 1-10ms), so you're wasting 1-10 seconds on database communication alone.

### 💻 The Code

**❌ Slow Version (The Problem):**

```python
# views.py
def books_slow(request):
    """This triggers 51 queries!"""
    books = Book.objects.all()[:50]  # Query 1: SELECT * FROM book
    for book in books:
        print(book.author.name)  # Queries 2-51: One per book!
    return render(request, 'books.html', {'books': books})
```

**SQL Generated:**

```sql
-- Query 1
SELECT * FROM book LIMIT 50;

-- Queries 2-51 (one for each book!)
SELECT * FROM author WHERE id = 1;
SELECT * FROM author WHERE id = 2;
...
SELECT * FROM author WHERE id = 50;
```

**✅ Fast Version (The Solution):**

```python
# views.py
def books_fast(request):
    """This triggers only 1 query!"""
    books = Book.objects.select_related('author')[:50]
    for book in books:
        print(book.author.name)  # Already in memory!
    return render(request, 'books.html', {'books': books})
```

**SQL Generated:**

```sql
-- Single query with JOIN!
SELECT book.*, author.*
FROM book
INNER JOIN author ON book.author_id = author.id
LIMIT 50;
```

### 🎯 The Solution: `select_related()`

The `select_related()` method performs an **SQL JOIN** at the database level. The related data comes back in a single query.

```python
# Syntax
Book.objects.select_related('author')              # 1 level
Book.objects.select_related('author', 'publisher') # Multiple relations
Book.objects.select_related('author__country')     # Nested (2 levels)
```

### 🧪 Real-World Template Example

**❌ Slow:**

```html
<!-- templates/books_slow.html -->
<h1>Books</h1>
<table>
    <tr>
        <th>Title</th>
        <th>Author</th>
        <th>Price</th>
    </tr>
    {% for book in books %}
    <tr>
        <td>{{ book.title }}</td>
        <td>{{ book.author.name }}</td>  <!-- N+1 happens here! -->
        <td>{{ book.price }}</td>
    </tr>
    {% endfor %}
</table>
```

**✅ Fast:**

```html
<!-- templates/books_fast.html -->
<!-- Just add select_related('author') in the view! -->
```

### 📝 When to Use

- `select_related()` for **ForeignKey** relationships
- `select_related()` for **OneToOneField** relationships
- Can chain with nested: `select_related('author__country')`

______________________________________________________________________

## Problem 2: N+1 Query with ManyToMany

### 🔍 What Is It?

Similar to the ForeignKey problem, but occurs with **ManyToManyField** and **Reverse ForeignKey** relationships. Each access to `.all()` on a ManyToMany triggers a new query.

### 🧠 Why Does It Happen?

ManyToMany relationships require a **join table** in the database. Django can't just fetch the related items in a simple JOIN like ForeignKey.

**Slow code:**

```python
books = Book.objects.all()[:50]  # Query 1: Get books
for book in books:
    for tag in book.tags.all():   # Queries 2-51: Get tags for each book!
        print(tag.name)
```

### 💻 The Code

**❌ Slow Version:**

```python
def tags_slow(request):
    """This triggers 51 queries!"""
    books = Book.objects.all()[:50]  # 1 query for books
    for book in books:
        tags = book.tags.all()  # 1 query per book = 50 queries
        list(tags)
    return render(request, 'tags.html', {'books': books})
```

**SQL Generated:**

```sql
-- Query 1
SELECT * FROM book LIMIT 50;

-- Queries 2-51
SELECT * FROM tag
INNER JOIN book_tags ON tag.id = book_tags.tag_id
WHERE book_tags.book_id = 1;

-- (repeated for each book)
```

**✅ Fast Version:**

```python
def tags_fast(request):
    """This triggers only 2 queries!"""
    books = Book.objects.prefetch_related('tags')[:50]
    for book in books:
        tags = book.tags.all()  # Uses prefetched cache
        list(tags)
    return render(request, 'tags.html', {'books': books})
```

**SQL Generated:**

```sql
-- Query 1: Get books
SELECT * FROM book LIMIT 50;

-- Query 2: Get ALL related tags in one go
SELECT tag.*, book_tags.book_id
FROM tag
INNER JOIN book_tags ON tag.id = book_tags.tag_id
WHERE book_tags.book_id IN (1, 2, 3, ..., 50);
```

### 🎯 The Solution: `prefetch_related()`

Unlike `select_related()` which uses a JOIN, `prefetch_related()` executes **two separate queries** and joins them in Python. This is more flexible and works with ManyToMany.

```python
# Syntax
Book.objects.prefetch_related('tags')              # 1 level
Book.objects.prefetch_related('tags', 'reviews')   # Multiple
Book.objects.prefetch_related('tags__category')    # Nested
```

### ⚖️ select_related vs prefetch_related

| Feature | `select_related()` | `prefetch_related()` |
|---------|-------------------|---------------------|
| SQL Operation | JOIN | Two queries + Python join |
| Queries | **1** | **2** |
| Use for | ForeignKey, OneToOne | ManyToMany, Reverse FK |
| Can filter related | ❌ Limited | ✅ Yes (with `Prefetch`) |
| Nested support | ✅ Yes | ✅ Yes |

### 📝 When to Use Each

```python
# ForeignKey → select_related
Book.objects.select_related('author')

# ManyToMany → prefetch_related
Book.objects.prefetch_related('tags')

# Both in same query
Book.objects.select_related('author').prefetch_related('tags')

# Reverse ForeignKey → prefetch_related
Author.objects.prefetch_related('books')
```

______________________________________________________________________

## Problem 3: SELECT * Unnecessary

### 🔍 What Is It?

Django's `all()` method generates `SELECT *`, which retrieves **all columns** from the database table, including large text fields and binary data you don't need.

### 🧠 Why Does It Matter?

Consider this scenario:

```python
class Book(models.Model):
    title = models.CharField(max_length=200)        # ~200 bytes
    synopsis = models.TextField()                    # Can be 10KB+
    cover_image = models.ImageField(upload_to='covers')  # Can be 1MB+!
    pdf_file = models.FileField(upload_to='pdfs')    # Can be 10MB+!
    published_year = models.IntegerField()
    price = models.DecimalField(max_digits=6, decimal_places=2)
```

If you only need `title` and `price` but query `Book.objects.all()`, you're fetching 10MB+ of data per book unnecessarily!

### 💻 The Code

**❌ Slow Version:**

```python
def fields_slow(request):
    """Loads ALL fields including large ones!"""
    books = Book.objects.all()[:50]
    data = [{'title': b.title, 'price': b.price} for b in books]
    return render(request, 'fields.html', {'data': data})
```

**SQL Generated:**

```sql
SELECT id, title, synopsis, cover_image, pdf_file, published_year, price
FROM book
LIMIT 50;
```

This fetches all columns, including potentially massive `synopsis`, `cover_image`, and `pdf_file` fields.

**✅ Fast Version:**

```python
def fields_fast(request):
    """Loads ONLY the fields we need!"""
    books = Book.objects.only('title', 'price')[:50]
    data = [{'title': b.title, 'price': b.price} for b in books]
    return render(request, 'fields.html', {'data': data})
```

**SQL Generated:**

```sql
SELECT id, title, price
FROM book
LIMIT 50;
```

### 🎯 The Solution: `only()` and `defer()`

```python
# only(): Load ONLY these fields (plus id)
Book.objects.only('title', 'price')

# defer(): Load everything EXCEPT these fields
Book.objects.defer('synopsis', 'cover_image', 'pdf_file')

# Nested relationships
Book.objects.select_related('author').only('title', 'author__name')
```

### ⚠️ Important Warning

After using `only()` or `defer()`, the object is **incomplete**. Accessing a deferred field triggers a new query!

```python
books = Book.objects.only('title', 'price')

book = books[0]
print(book.title)     # ✅ OK - already loaded
print(book.price)    # ✅ OK - already loaded
print(book.synopsis) # ❌ NEW QUERY! - deferred field
```

### 📊 When to Use

| Situation | Method |
|----------|--------|
| Know exactly which fields you need | `only('field1', 'field2')` |
| Know which fields you DON'T need | `defer('big_field', 'file_field')` |
| Use with `select_related()` | `only('title', 'author__name')` |

______________________________________________________________________

## Problem 4: len() vs count()

### 🔍 What Is It?

Using `len()` on a QuerySet **evaluates the entire queryset** and loads all records into memory just to count them. Using `.count()` performs a `SELECT COUNT(*)` at the database level.

### 🧠 Why Does It Matter?

```python
# These look similar but are VERY different:
len(Book.objects.all())      # Loads ALL 1 million books into memory!
Book.objects.count()         # Just asks: "how many rows?"
```

### 💻 The Code

**❌ Slow Version:**

```python
def count_slow(request):
    """Loads ALL books into memory just to count them!"""
    books = Book.objects.all()      # Loads 1 million records
    total = len(books)              # Python counts them in memory
    return render(request, 'count.html', {'total': total})
```

**SQL Generated:**

```sql
SELECT * FROM book;  -- Returns 1 million rows!
```

Then Python counts them in memory. This is **extremely slow and memory-intensive**.

**✅ Fast Version:**

```python
def count_fast(request):
    """Counts at database level - super fast!"""
    total = Book.objects.count()    # Database counts the rows
    return render(request, 'count.html', {'total': total})
```

**SQL Generated:**

```sql
SELECT COUNT(*) FROM book;  -- Returns just the number: 1000000
```

### 📊 Performance Comparison

| Records | `len()` | `.count()` | Speedup |
|---------|---------|------------|---------|
| 100 | 5ms | 1ms | 5x |
| 10,000 | 500ms | 1ms | 500x |
| 1,000,000 | 50s | 2ms | 25,000x |

### 📝 When to Use Each

| Situation | Method | Why |
|----------|--------|-----|
| Need ONLY the count | `.count()` | ✅ Efficient |
| Need count AND the list | `len(queryset)` or `.count()` | Either works |
| Already have cached data | `len(list_of_objects)` | ✅ Fine |
| Checking if queryset is empty | `.exists()` | ✅ More efficient |

### 💡 Pro Tip: The `.exists()` Method

```python
# ❌ Wrong - loads all records
if len(Book.objects.filter(published_year=2024)):
    pass

# ✅ Correct - just checks existence
if Book.objects.filter(published_year=2024).exists():
    pass
```

______________________________________________________________________

## Problem 5: Missing Pagination

### 🔍 What Is It?

Loading all records at once overwhelms the database, server memory, and browser. With large datasets, this causes timeouts, crashes, and terrible user experience.

### 🧠 Why Does It Matter?

```python
# Loading 100,000 books at once:
books = Book.objects.all()  # 100,000 records

# Memory usage:
# - Python: ~500MB
# - Database: 100,000 rows sent
# - Network: 50MB+ transferred
# - Browser: Freezes while rendering 100,000 rows
```

### 💻 The Code

**❌ Slow Version:**

```python
def paginate_slow(request):
    """Loads ALL books - can crash with large datasets!"""
    books = Book.objects.select_related('author').all()[:200]
    return render(request, 'books.html', {'books': books})
```

**✅ Fast Version:**

```python
from django.core.paginator import Paginator

def paginate_fast(request):
    """Loads only 20 books at a time - efficient!"""
    qs = Book.objects.select_related('author').only(
        'title', 'published_year', 'author__name'
    )
    paginator = Paginator(qs, per_page=20)
    page = paginator.get_page(request.GET.get('page', 1))
    return render(request, 'books.html', {'page': page})
```

**SQL Generated (Page 1):**

```sql
-- Query 1: Total count
SELECT COUNT(*) FROM book;

-- Query 2: Current page
SELECT book.id, book.title, book.published_year, author.name
FROM book
INNER JOIN author ON book.author_id = author.id
LIMIT 20 OFFSET 0;
```

### 📝 Template with Pagination

```html
<h1>Books - Page {{ page.number }} of {{ page.paginator.num_pages }}</h1>

<ul>
{% for book in page.object_list %}
    <li>{{ book.title }} by {{ book.author.name }}</li>
{% endfor %}
</ul>

<nav>
    {% if page.has_previous %}
        <a href="?page={{ page.previous_page_number }}">Previous</a>
    {% endif %}
    
    <span>Page {{ page.number }}</span>
    
    {% if page.has_next %}
        <a href="?page={{ page.next_page_number }}">Next</a>
    {% endif %}
</nav>
```

### 📊 Performance Impact

| Books | No Pagination | With Pagination (20/page) |
|-------|--------------|--------------------------|
| 100 | 100 loaded | 20 loaded |
| 1,000 | 1,000 loaded | 20 loaded |
| 100,000 | 💥 Crash | 20 loaded |

### 🎯 Pagination Best Practices

```python
# Always combine with select_related and only
qs = Book.objects.select_related('author').only(
    'title', 'author__name'
)
paginator = Paginator(qs, per_page=20)

# Handle invalid page numbers
page = paginator.get_page(request.GET.get('page', 1))
# Automatically handles page=0, page=9999, etc.
```

______________________________________________________________________

## How to Identify These Problems

### 1. Django Debug Toolbar

Install the Django Debug Toolbar to see query counts per request:

```bash
pip install django-debug-toolbar
```

Add to `settings.py`:

```python
INSTALLED_APPS = [
    'debug_toolbar',
    # ...
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    # ...
]
```

Visit `/__debug__/` in development to see queries.

### 2. Django Silk

For production profiling:

```bash
pip install django-silk
```

Visit `/silk/` to see request profiling.

### 3. The Query Count Middleware

Create custom middleware to count queries:

```python
# middleware.py
class QueryCountMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        from django.db import connection
        initial = len(connection.queries)
        response = self.get_response(request)
        query_count = len(connection.queries) - initial
        response['X-Query-Count'] = query_count
        return response
```

______________________________________________________________________

## Summary: Quick Reference

```python
# ═══════════════════════════════════════════════════════════
# OPTIMIZATION CHEAT SHEET
# ═══════════════════════════════════════════════════════════

# 1. ForeignKey → select_related (JOIN)
Book.objects.select_related('author')

# 2. ManyToMany → prefetch_related (2 queries)
Book.objects.prefetch_related('tags')

# 3. Load only needed fields
Book.objects.only('title', 'price')

# 4. Count efficiently (not len!)
Book.objects.count()

# 5. Always paginate
from django.core.paginator import Paginator
Paginator(qs, per_page=20)

# ═══════════════════════════════════════════════════════════
# COMBINING OPTIMIZATIONS
# ═══════════════════════════════════════════════════════════

# Best: select_related + only + paginate
books = Book.objects.select_related('author').only(
    'title', 'published_year', 'author__name'
)
paginator = Paginator(books, per_page=20)
page = paginator.get_page(request.GET.get('page', 1))
```

______________________________________________________________________

## Try It Yourself: Demo Application

🚀 **Explore the live demo application and test all optimizations hands-on!**

The demo application includes:

- **5 interactive demonstrations** - One for each performance problem
- **Side-by-side comparison** - Slow vs Fast implementations
- **Real-time query counting** - See exactly how many queries each approach generates
- **Playground mode** - Test custom scenarios

### Live Demo Links

| Problem | Slow Version | Fast Version |
|---------|-------------|--------------|
| N+1 ForeignKey | [`/books/slow/`](/books/slow/) | [`/books/fast/`](/books/fast/) |
| N+1 ManyToMany | [`/tags/slow/`](/tags/slow/) | [`/tags/fast/`](/tags/fast/) |
| SELECT * | [`/fields/slow/`](/fields/slow/) | [`/fields/fast/`](/fields/fast/) |
| len() vs count() | [`/count/slow/`](/count/slow/) | [`/count/fast/`](/count/fast/) |
| Pagination | [`/paginate/slow/`](/paginate/slow/) | [`/paginate/fast/`](/paginate/fast/) |

### Interactive Playground

Test all problems interactively at: [**`/playground/`**](/playground/)

______________________________________________________________________

## Conclusion

These 5 performance problems are the **most common culprits** behind slow Django applications. The good news? They're all **easy to fix** once you know what to look for:

1. **N+1 ForeignKey** → Use `select_related()`
1. **N+1 ManyToMany** → Use `prefetch_related()`
1. \*\*SELECT \*\*\* → Use `only()` or `defer()`
1. **len()** → Use `.count()`
1. **All records** → Use pagination

Start applying these optimizations today, and watch your application performance soar! 🚀

______________________________________________________________________

## References

- [Django ORM Documentation](https://docs.djangoproject.com/en/stable/topics/db/queries/)
- [select_related()](https://docs.djangoproject.com/en/stable/ref/models/querysets/#select-related)
- [prefetch_related()](https://docs.djangoproject.com/en/stable/ref/models/querysets/#prefetch-related)
- [only() and defer()](https://docs.djangoproject.com/en/stable/ref/models/querysets/#only-and-defer)
- [Django Paginator](https://docs.djangoproject.com/en/stable/topics/pagination/)
- [Django Debug Toolbar](https://django-debug-toolbar.readthedocs.io/)
- [Django Silk](https://github.com/jazzband/django-silk)
- [Demo Application on GitHub](https://github.com/alisonamerico/django-perf-demo)
