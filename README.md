# Django ORM Query Optimization, Aggregation, and `annotate()`

A practical reference for writing efficient Django ORM queries using `Q`, `Count`, `Sum`, `Avg`, `Min`, `Max`, `annotate()`, `aggregate()`, conditional aggregation, and related query best practices.

---

## Table of Contents

1. [The Big Picture](#the-big-picture)
2. [`Q()` Expressions](#q-expressions)
3. [`Count()`](#count)
4. [`Sum()`](#sum)
5. [`Avg()`, `Min()`, and `Max()`](#avg-min-and-max)
6. [`annotate()` vs `aggregate()`](#annotate-vs-aggregate)
7. [Combining `Q()` with Aggregation](#combining-q-with-aggregation)
8. [Conditional `Count()`](#conditional-count)
9. [Conditional `Sum()`](#conditional-sum)
10. [Multiple Aggregations and Join Duplication](#multiple-aggregations-and-join-duplication)
11. [Aggregation Across Relationships](#aggregation-across-relationships)
12. [Filtering Before Aggregating](#filtering-before-aggregating)
13. [`values()` with Aggregation](#values-with-aggregation)
14. [Database Query Optimization Best Practices](#database-query-optimization-best-practices)
15. [`select_related()` and `prefetch_related()`](#select_related-and-prefetch_related)
16. [Avoiding N+1 Queries](#avoiding-n1-queries)
17. [Only Fetch What You Need](#only-fetch-what-you-need)
18. [`exists()` and `count()`](#exists-and-count)
19. [Filter in the Database](#filter-in-the-database)
20. [Indexes](#indexes)
21. [Pagination](#pagination)
22. [Bulk Operations](#bulk-operations)
23. [Avoid Database Queries in Templates](#avoid-database-queries-in-templates)
24. [Using `explain()`](#using-explain)
25. [Measuring Queries](#measuring-queries)
26. [Dashboard Example](#dashboard-example)
27. [Quick Mental Model](#quick-mental-model)
28. [Learning Progression](#learning-progression)

---

# The Big Picture

Django's ORM allows you to express database operations using Python instead of writing SQL directly.

For example:

```python
orders = ResearchOrder.objects.filter(
    status="approved"
)
```

Django translates this into SQL and asks the database to perform the operation.

A major performance principle is:

> **Let the database do database work.**

Instead of loading thousands of records into Python and calculating totals, counts, or filters yourself, use Django ORM operations that translate into database operations.

For example, avoid:

```python
total = 0

for task in Task.objects.all():
    total += task.hours
```

Prefer:

```python
from django.db.models import Sum

result = Task.objects.aggregate(
    total_hours=Sum("hours")
)
```

---

# `Q()` Expressions

`Q()` allows you to construct complex query conditions.

Import it:

```python
from django.db.models import Q
```

## Simple OR

Suppose we want orders whose status is either `approved` or `pending`.

```python
orders = ResearchOrder.objects.filter(
    Q(status="approved") |
    Q(status="pending")
)
```

The `|` operator means **OR**.

Conceptually:

```sql
WHERE status = 'approved'
   OR status = 'pending'
```

## AND

The `&` operator means **AND**.

```python
orders = ResearchOrder.objects.filter(
    Q(status="approved") &
    Q(client=user)
)
```

For simple AND conditions, normal Django syntax is usually cleaner:

```python
orders = ResearchOrder.objects.filter(
    status="approved",
    client=user,
)
```

Use `Q()` when the logic becomes more complex.

## NOT

The `~` operator means **NOT**.

```python
orders = ResearchOrder.objects.filter(
    ~Q(status="rejected")
)
```

This means:

> Return orders whose status is not rejected.

## Combining AND and OR

Suppose we want:

> Orders belonging to a particular client where status is either approved or pending.

```python
orders = ResearchOrder.objects.filter(
    Q(client=user) &
    (
        Q(status="approved") |
        Q(status="pending")
    )
)
```

Parentheses are important because they make the intended logical grouping explicit.

## Search across multiple fields

A common real-world use is a search box.

```python
search = "employee"

orders = ResearchOrder.objects.filter(
    Q(title__icontains=search) |
    Q(description__icontains=search)
)
```

This means:

> Find records where either title or description contains "employee", ignoring case.

---

# `Count()`

`Count()` answers:

> **How many?**

Import:

```python
from django.db.models import Count
```

Suppose:

```text
ResearchOrder
    |
    +-- Phase
    +-- Phase
    +-- Phase
```

We want the number of phases for each order.

Use `annotate()`:

```python
orders = ResearchOrder.objects.annotate(
    phase_count=Count("phases")
)
```

Then:

```python
for order in orders:
    print(order.title)
    print(order.phase_count)
```

Each `ResearchOrder` now has a calculated `phase_count` attribute.

## Counting all records

If you simply need the number of orders:

```python
total_orders = ResearchOrder.objects.count()
```

## Counting with `aggregate()`

```python
result = ResearchOrder.objects.aggregate(
    total=Count("id")
)
```

Result:

```python
{
    "total": 25
}
```

---

# `Sum()`

`Sum()` answers:

> **What is the total?**

Import:

```python
from django.db.models import Sum
```

Suppose:

```python
class Task(models.Model):
    phase = models.ForeignKey(
        Phase,
        on_delete=models.CASCADE
    )

    hours = models.DecimalField(
        max_digits=10,
        decimal_places=2
    )
```

Calculate total hours:

```python
result = Task.objects.aggregate(
    total_hours=Sum("hours")
)
```

Result might look like:

```python
{
    "total_hours": Decimal("245.50")
}
```

## Sum per related object

Suppose each phase has tasks.

```python
phases = Phase.objects.annotate(
    total_hours=Sum("tasks__hours")
)
```

Now:

```python
phase.total_hours
```

contains the total hours of that phase's tasks.

---

# `Avg()`, `Min()`, and `Max()`

These work similarly to `Sum()`.

```python
from django.db.models import (
    Avg,
    Min,
    Max,
)
```

Example:

```python
result = Task.objects.aggregate(
    average_hours=Avg("hours"),
    minimum_hours=Min("hours"),
    maximum_hours=Max("hours"),
)
```

You can combine them with `Count()` and `Sum()`:

```python
from django.db.models import (
    Count,
    Sum,
    Avg,
    Min,
    Max,
)

result = Task.objects.aggregate(
    count=Count("id"),
    total_hours=Sum("hours"),
    average_hours=Avg("hours"),
    minimum_hours=Min("hours"),
    maximum_hours=Max("hours"),
)
```

---

# `annotate()` vs `aggregate()`

This is one of the most important distinctions.

## `annotate()`

`annotate()` adds calculated values to **each object**.

Example:

```python
orders = ResearchOrder.objects.annotate(
    phase_count=Count("phases")
)
```

Conceptually:

```text
Order A -> 3 phases
Order B -> 7 phases
Order C -> 2 phases
```

You can access:

```python
order.phase_count
```

Think:

> **Calculate something for every row/object.**

## `aggregate()`

`aggregate()` calculates one or more values for the **whole queryset**.

```python
result = ResearchOrder.objects.aggregate(
    total=Count("id")
)
```

Result:

```python
{
    "total": 25
}
```

Think:

> **Calculate something for the entire queryset.**

### Easy memory trick

```text
annotate()  -> one result attached to each object
aggregate() -> one overall result
```

---

# Combining `Q()` with Aggregation

This is where the ORM becomes particularly powerful.

Suppose tasks have a `status` field:

```text
pending
in_progress
completed
cancelled
```

We want to count only completed tasks.

```python
from django.db.models import Count, Q

orders = ResearchOrder.objects.annotate(
    completed_tasks=Count(
        "phases__tasks",
        filter=Q(
            phases__tasks__status="completed"
        ),
        distinct=True,
    )
)
```

Now:

```python
order.completed_tasks
```

gives the number of completed tasks for that order.

---

# Conditional `Count()`

Suppose we want several dashboard metrics:

```text
Total tasks
Completed tasks
Pending tasks
```

We can calculate all three in one queryset.

```python
from django.db.models import Count, Q

orders = ResearchOrder.objects.annotate(

    total_tasks=Count(
        "phases__tasks",
        distinct=True,
    ),

    completed_tasks=Count(
        "phases__tasks",
        filter=Q(
            phases__tasks__status="completed"
        ),
        distinct=True,
    ),

    pending_tasks=Count(
        "phases__tasks",
        filter=Q(
            phases__tasks__status="pending"
        ),
        distinct=True,
    ),
)
```

Then:

```python
order.total_tasks
order.completed_tasks
order.pending_tasks
```

This is very useful for dashboards.

---

# Conditional `Sum()`

`Sum()` can also use a filter.

Suppose tasks contain:

```python
hours
status
```

We want:

> Total hours of completed tasks.

```python
from django.db.models import Sum, Q

orders = ResearchOrder.objects.annotate(
    completed_hours=Sum(
        "phases__tasks__hours",
        filter=Q(
            phases__tasks__status="completed"
        ),
    )
)
```

Now:

```python
order.completed_hours
```

contains the total hours for completed tasks.

---

# Multiple Aggregations and Join Duplication

This is a critical concept.

Suppose:

```text
Order
 |
 +-- Phase 1
 |    +-- Task 1
 |    +-- Task 2
 |
 +-- Phase 2
      +-- Task 3
      +-- Task 4
```

We expect:

```text
phase_count = 2
task_count = 4
```

But when several related tables are joined, SQL can create duplicate combinations.

For counts, `distinct=True` can often prevent inflated counts:

```python
orders = ResearchOrder.objects.annotate(
    phase_count=Count(
        "phases",
        distinct=True,
    ),

    task_count=Count(
        "phases__tasks",
        distinct=True,
    ),
)
```

This is especially important when combining multiple aggregations across relationships.

## Important caution

`distinct=True` is useful, but it is not a universal fix for every aggregation problem.

For more complicated queries, you may need:

- separate querysets
- subqueries
- `Subquery()`
- `OuterRef()`
- database-specific approaches

Always inspect the generated SQL and verify the result against known data.

---

# Aggregation Across Relationships

Django allows you to traverse relationships using `__`.

Suppose:

```text
ResearchOrder
    |
    +-- Phase
          |
          +-- Activity
                |
                +-- Task
```

You can traverse the relationships:

```python
Count("phases__activities__tasks")
```

Or:

```python
Sum("phases__activities__budget")
```

This allows you to build powerful reporting queries without manually writing SQL joins.

---

# Filtering Before Aggregating

The order of queryset operations can matter.

Suppose we only want approved orders before calculating totals:

```python
orders = (
    ResearchOrder.objects
    .filter(status="approved")
    .annotate(
        phase_count=Count(
            "phases",
            distinct=True,
        )
    )
)
```

The filter limits the records being considered.

This can be more efficient than retrieving everything and filtering in Python.

Always be conscious of what records you are asking the database to process.

---

# `values()` with Aggregation

`values()` is useful when you want grouped results rather than model objects.

Suppose you want the number of orders by status:

```python
from django.db.models import Count

result = (
    ResearchOrder.objects
    .values("status")
    .annotate(
        total=Count("id")
    )
)
```

Conceptually:

```text
approved  -> 20
pending   -> 12
rejected  -> 5
```

This is similar to SQL:

```sql
SELECT status, COUNT(id)
FROM research_order
GROUP BY status;
```

This pattern is excellent for reports and dashboard summaries.

---

# Database Query Optimization Best Practices

Aggregation is only one part of query optimization.

The general goal is:

> Retrieve only the data you need, with as few unnecessary database operations as possible.

---

# `select_related()` and `prefetch_related()`

These solve many relationship-related query problems.

## `select_related()`

Use for:

- `ForeignKey`
- `OneToOneField`

Example:

```python
orders = ResearchOrder.objects.select_related(
    "client"
)
```

For nested relationships:

```python
orders = ResearchOrder.objects.select_related(
    "client__user"
)
```

## `prefetch_related()`

Use for:

- reverse ForeignKey
- ManyToMany relationships

Example:

```python
orders = ResearchOrder.objects.prefetch_related(
    "phases",
    "collaborators",
)
```

### Quick rule

```text
ForeignKey / OneToOne
        ↓
select_related()

ManyToMany / reverse FK
        ↓
prefetch_related()
```

---

# Avoiding N+1 Queries

N+1 is one of the most common Django ORM performance problems.

Bad:

```python
orders = ResearchOrder.objects.all()

for order in orders:
    print(order.client.email)
```

This can result in:

```text
1 query -> retrieve orders
N queries -> retrieve each client's data
```

Instead:

```python
orders = ResearchOrder.objects.select_related(
    "client"
)
```

Now Django can retrieve the related client data efficiently.

Another example:

```python
orders = ResearchOrder.objects.prefetch_related(
    "phases"
)

for order in orders:
    for phase in order.phases.all():
        print(phase.name)
```

---

# Only Fetch What You Need

If you only need selected fields, consider `values()`:

```python
users = User.objects.values(
    "id",
    "first_name",
    "email",
)
```

Instead of retrieving complete model objects when you don't need them.

You can also use:

```python
User.objects.only(
    "id",
    "first_name",
    "email",
)
```

### Caution with `only()`

If you later access a deferred field:

```python
user.phone
```

Django may issue another database query.

Use `only()` deliberately rather than automatically.

---

# `exists()` and `count()`

## Use `exists()` when you only need to know whether something exists

Avoid:

```python
if Order.objects.filter(client=user):
    ...
```

Prefer:

```python
if Order.objects.filter(client=user).exists():
    ...
```

## Use `count()` when you need a count

```python
total = Order.objects.filter(
    client=user
).count()
```

Avoid retrieving all records simply to count them.

---

# Filter in the Database

Avoid:

```python
orders = ResearchOrder.objects.all()

active_orders = [
    order
    for order in orders
    if order.status == "active"
]
```

Prefer:

```python
active_orders = ResearchOrder.objects.filter(
    status="active"
)
```

The database is optimized for filtering data.

---

# Indexes

Indexes can make frequently used filters much faster.

Example:

```python
class ResearchOrder(models.Model):
    status = models.CharField(
        max_length=30,
        db_index=True,
    )
```

Or:

```python
created_at = models.DateTimeField(
    db_index=True
)
```

Indexes can be particularly useful for fields frequently used in:

```python
.filter()
.exclude()
.get()
.order_by()
```

## Composite indexes

Suppose you frequently query:

```python
ResearchOrder.objects.filter(
    client=client,
    status="active",
)
```

A composite index may help:

```python
class Meta:
    indexes = [
        models.Index(
            fields=["client", "status"]
        ),
    ]
```

The correct indexes depend on your actual query patterns.

### Don't index everything

Indexes have costs:

- storage
- additional work during inserts
- additional work during updates
- additional work during deletes

Index fields because there is a query-performance reason to do so.

---

# Pagination

Don't retrieve thousands of records for one page.

Instead of:

```python
orders = ResearchOrder.objects.all()
```

for a huge table, paginate:

```python
from django.core.paginator import Paginator

orders = ResearchOrder.objects.all()

paginator = Paginator(
    orders,
    25,
)

page = paginator.get_page(
    page_number
)
```

Now the UI only handles a manageable number of records.

---

# Bulk Operations

Avoid repeatedly calling `save()` in a loop when a bulk database operation is appropriate.

Instead of:

```python
for task in tasks:
    task.status = "completed"
    task.save()
```

You can often do:

```python
Task.objects.filter(
    id__in=task_ids
).update(
    status="completed"
)
```

For creating many objects:

```python
Task.objects.bulk_create(
    tasks
)
```

Be aware that bulk operations have different behavior from normal model `.save()` calls, including around signals and custom save logic.

---

# Avoid Database Queries in Templates

Avoid repeatedly calculating relationships inside templates.

For example:

```django
{% for order in orders %}
    {{ order.phases.count }}
{% endfor %}
```

Depending on the relationship and how the queryset was prepared, this can create repeated queries.

Instead, prepare the calculation in the view:

```python
orders = ResearchOrder.objects.annotate(
    phase_count=Count("phases")
)
```

Then:

```django
{% for order in orders %}
    {{ order.phase_count }}
{% endfor %}
```

The template should primarily display prepared data.

---

# Using `explain()`

When a query is slow, inspect how the database plans to execute it.

Example:

```python
queryset = ResearchOrder.objects.filter(
    status="active"
)

print(queryset.explain())
```

This can help identify:

- sequential/table scans
- index usage
- joins
- sorting
- query plans

For PostgreSQL, more detailed analysis can be requested:

```python
print(
    queryset.explain(
        analyze=True,
        buffers=True,
    )
)
```

### Important

`analyze=True` actually executes the query.

Be careful when using it against production data.

---

# Measuring Queries

Don't optimize based only on intuition.

Use tools such as Django Debug Toolbar during development.

For example, if a page produces:

```text
SQL Queries: 157
```

investigate why.

You might discover:

```text
1 query -> orders
50 queries -> clients
50 queries -> phases
56 queries -> other related objects
```

After optimization, the page might use a much smaller number of well-designed queries.

The goal is not simply:

> "Make the number of queries as small as possible."

The goal is:

> "Make the database work appropriate, efficient, and predictable."

---

# Dashboard Example

Consider a simplified RaaS structure:

```text
ResearchOrder
│
├── Client
│
├── Collaborators
│
└── Phase
    │
    └── Activity
        │
        └── Task
```

Suppose:

```python
class Task(models.Model):
    activity = models.ForeignKey(
        Activity,
        on_delete=models.CASCADE,
        related_name="tasks",
    )

    status = models.CharField(
        max_length=30,
    )

    hours = models.DecimalField(
        max_digits=10,
        decimal_places=2,
    )
```

Suppose `Activity` has:

```python
budget = models.DecimalField(
    max_digits=12,
    decimal_places=2,
)
```

A dashboard might need:

- number of phases
- total tasks
- completed tasks
- pending tasks
- completed hours
- total activity budget

One possible queryset:

```python
from django.db.models import (
    Count,
    Sum,
    Q,
)

orders = ResearchOrder.objects.annotate(

    phase_count=Count(
        "phases",
        distinct=True,
    ),

    task_count=Count(
        "phases__activities__tasks",
        distinct=True,
    ),

    completed_tasks=Count(
        "phases__activities__tasks",
        filter=Q(
            phases__activities__tasks__status="completed"
        ),
        distinct=True,
    ),

    pending_tasks=Count(
        "phases__activities__tasks",
        filter=Q(
            phases__activities__tasks__status="pending"
        ),
        distinct=True,
    ),

    completed_hours=Sum(
        "phases__activities__tasks__hours",
        filter=Q(
            phases__activities__tasks__status="completed"
        ),
    ),

    total_budget=Sum(
        "phases__activities__budget"
    ),
)
```

Then the template can use:

```django
{{ order.phase_count }}
{{ order.task_count }}
{{ order.completed_tasks }}
{{ order.pending_tasks }}
{{ order.completed_hours }}
{{ order.total_budget }}
```

This is much cleaner than querying the database repeatedly while rendering the page.

---

# Quick Mental Model

Use this mental model when writing Django ORM queries.

## `filter()`

> Which records do I want?

```python
Task.objects.filter(
    status="completed"
)
```

## `Q()`

> How complicated is my filtering logic?

```python
Task.objects.filter(
    Q(status="completed") |
    Q(status="in_progress")
)
```

## `Count()`

> How many?

```python
Count("tasks")
```

## `Sum()`

> What is the total?

```python
Sum("tasks__hours")
```

## `Avg()`

> What is the average?

```python
Avg("tasks__hours")
```

## `Min()` / `Max()`

> What is the smallest/largest?

```python
Min("tasks__hours")
Max("tasks__hours")
```

## `annotate()`

> Give me a calculation for each object.

```python
Phase.objects.annotate(
    task_count=Count("tasks")
)
```

## `aggregate()`

> Give me an overall calculation.

```python
Task.objects.aggregate(
    total_hours=Sum("hours")
)
```

## `select_related()`

> Fetch ForeignKey/OneToOne relationships efficiently.

## `prefetch_related()`

> Fetch ManyToMany/reverse relationships efficiently.

## `distinct=True`

> Avoid duplicate related objects in counts when joins can multiply rows.

---

# Learning Progression

A useful sequence for mastering Django ORM analytics is:

### Lesson 1 — `Q()`

Learn:

- OR
- AND
- NOT
- nested conditions
- search across fields

### Lesson 2 — `Count()`

Learn:

- counting model records
- counting relationships
- `distinct=True`
- counting by groups

### Lesson 3 — `Sum()`

Learn:

- totals
- related-field sums
- financial calculations
- conditional sums

### Lesson 4 — `annotate()` vs `aggregate()`

Learn:

```text
annotate()  -> per object
aggregate() -> whole queryset
```

### Lesson 5 — Conditional Aggregation

Combine:

```python
Q()
Count()
Sum()
```

For example:

```python
Count(
    "tasks",
    filter=Q(tasks__status="completed")
)
```

### Lesson 6 — `F()` Expressions

Learn to compare database fields:

```python
from django.db.models import F

Task.objects.filter(
    actual_hours__gt=F("estimated_hours")
)
```

### Lesson 7 — `Case()` and `When()`

Learn conditional calculated fields.

### Lesson 8 — Joins and Duplicate Aggregations

Understand why counts can become inflated and when `distinct=True` is appropriate.

### Lesson 9 — `Subquery()` and `OuterRef()`

Use these for more advanced database-side calculations.

### Lesson 10 — Real Dashboard Querysets

Combine everything into practical Django analytics.

---

# Final Best-Practice Checklist

Before shipping a Django view involving database queries, ask:

- [ ] Am I querying inside a loop?
- [ ] Could this be an N+1 query?
- [ ] Should I use `select_related()`?
- [ ] Should I use `prefetch_related()`?
- [ ] Am I retrieving fields I don't need?
- [ ] Could `values()` be appropriate?
- [ ] Should I use `exists()` instead of loading objects?
- [ ] Should I use `count()` instead of loading objects?
- [ ] Can the database perform this filtering?
- [ ] Can `annotate()` replace calculations in Python?
- [ ] Can `aggregate()` calculate the overall result?
- [ ] Can `Count()` replace a Python loop?
- [ ] Can `Sum()` replace a Python loop?
- [ ] Do I need conditional aggregation with `Q()`?
- [ ] Could joins cause duplicate counts?
- [ ] Do I need `distinct=True`?
- [ ] Is pagination needed?
- [ ] Are frequently filtered fields indexed?
- [ ] Would a composite index help?
- [ ] Have I inspected the query with `explain()`?
- [ ] Have I actually measured the number and cost of queries?

---

## Core Principle

The most important lesson is not to memorize every Django ORM function.

Think in terms of the database:

```text
Python asks:
    "What data do I need?"

Django ORM translates that into:
    SQL

Database performs:
    filtering
    joining
    counting
    summing
    grouping
    sorting

Django returns:
    only the result your application needs
```

The more effectively you can express your requirements through the ORM, the less unnecessary work your Django application has to perform in Python.
