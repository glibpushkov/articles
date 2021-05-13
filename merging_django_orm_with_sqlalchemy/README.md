![Cover image](images/a0ngpwycmxttdxqvyyz4.png)
# Merging  Django ORM with SQLAlchemy for Easier Data Analysis
Development of products with Django framework is usually easy and straightforward; great documentation, many tools out of the box, plenty of open source libraries and big community. Django ORM takes full control about SQL layer protecting you from mistakes, and underlying details of queries so you can spend more time on designing and building your application structure in Python code. However, sometimes such behavior may hurt - for example, when you’re building a project related to data analysis. Building advanced queries with Django is not very easy; it’s hard to read (in Python) and hard to understand what’s going on in SQL-level without logging or printing generated SQL queries somewhere. Moreover, such queries could not be efficient enough, so this will hit you back when you load more data into DB to play with. In one moment, you can find yourself doing too much raw SQL through Django cursor, and this is the moment when you should do a break and take a look on another interesting tool, which is placed right between ORM layer and the layer of raw SQL queries.

As you can see from the title of the article, we successfully mixed Django ORM and SQLAlchemy Core together, and we’re very satisfied with results. We built an application which helps to analyze data produced by EMR systems by aggregating data into charts and tables, scoring by throughput/efficiency/staff cost, and highlighting outliers which allows to optimize business processes for clinics and save money.

## What is the point of mixing Django ORM with SQLAlchemy?

There are a few reasons why we stepped out from Django ORM for this task:

* For ORM world, one object is one record in the database, but here we deal only with aggregated data.

* Some aggregations are very tricky, and Django ORM functionality is not enough to fulfill the needs. To be honest, sometimes in some simple cases it’s hard (or even impossible) to make ORM produce the SQL query exactly the way you want, and when you’re dealing with a **big data**, it will affect performance a lot.

* If you’re building advanced queries via Django ORM, it’s hard to read and understand such queries in Python, and hard to predict which SQL query will be generated and treated to the database.

It’s worth saying that we also set up a second database, which is handled by Django ORM to cover other web application related tasks and business-logic needs, which it perfectly does. Django ORM is evolving from version to version, giving more and more features. For example, in recent releases, a bunch of neat features were added like support of [Subquery expressions](https://docs.djangoproject.com/en/1.11/ref/models/expressions/#subquery-expressions) or [Window functions](https://docs.djangoproject.com/en/2.1/ref/models/database-functions/#window-functions) and many others, which you should definitely try before doing raw SQL or looking at the tools like SQLAlchemy if your problem is more complex than fixing a few queries.

So that’s why we decided to take a look at SQLAlchemy. It consists of two parts - ORM and Core. SQLAlchemy ORM is similar to Django ORM, but at the same time, they differ. SQLAlchemy ORM uses a different concept, Data Mapper, compared to Django's Active Record approach. As far as you’re building projects on Django, you definitely should not switch ORM (if you don’t have very special reasons to do so), as you want to use Django REST framework, Django-admin, and other neat stuff which is tied to Django models.

The second part of SQLAlchemy is called Core. It’s placed right between high-level ORM and low-level SQL. The Core is very powerful and flexible; it gives you the ability to build any SQL-queries you wish, and when you see such queries in Python, it’s easy to understand what’s going on. For example, take a look into a sample query from the documentation:

```  python
q = session.query(User).filter(User.name.like('e%')).\
    limit(5).from_self().\
    join(User.addresses).filter(Address.email.like('q%')).\
    order_by(User.name)
```

Which will result into 

```python
SELECT anon_1.user_id AS anon_1_user_id,
       anon_1.user_name AS anon_1_user_name
FROM (SELECT "user".id AS user_id, "user".name AS user_name
FROM "user"
WHERE "user".name LIKE :name_1
 LIMIT :param_1) AS anon_1
JOIN address ON anon_1.user_id = address.user_id
WHERE address.email LIKE :email_1 ORDER BY anon_1.user_name
```

Note: with such tricks, we don’t fall into `N+1 problem`: `from_select` makes an additional `SELECT` wrapper around the query, so we reduce the amount of rows at first (via `LIKE` and `LIMIT`) and only then we join the address information.

## How to mix Django application and SQLALchemy

So if you’re interested and want to try to mix SQLAlchemy with Django application, here are some hints which could help you.

First of all, you need to [create](http://docs.sqlalchemy.org/en/latest/core/engines.html#engine-configuration) a global variable with Engine, but the actual connection with DB will be established on first `connect` or `execute` call.

```python
sa_engine = create_engine(settings.DB_CONNECTION_URL, pool_recycle=settings.POOL_RECYCLE)
```

Create_engine accepts additional configuration for connection. MySQL/MariaDB/AWS Aurora(MySQL compatible) have an interactive_timeout setting which is 8h by default, so without pool_recycle extra parameter you will get annoying `SQLError: (OperationalError) (2006, ‘MySQL server has gone away’)`. So the `POOL_RECYCLE` should be smaller than `interactive_timeout`. For example a half of it: `POOL_RECYCLE = 4 * 60 * 60`

Next step is to build your queries. Depending on your application architecture, you can declare tables and fields [using](http://docs.sqlalchemy.org/en/latest/core/metadata.html) `Table` and `Column` classes (which also can be used with ORM), or if your application already stores tables and columns names in another way, you can do it in place, via table (`table_name`) and column (`col_name`) functions (as shown [here](http://docs.sqlalchemy.org/en/latest/core/selectable.html#sqlalchemy.sql.expression.table)).

In our app, we picked the second option, as we stored information about aggregations, formulas, and formatting in our own declarative syntax. Then we built a layer which read such structures and executed queries based on the provided instructions.

When your query is ready, simply call `sa_engine.execute(query)`. The cursor would be opened until you read all the data or if you close it explicitly.

There is one very annoying thing worth mentioning. As a [documentation says](http://docs.sqlalchemy.org/en/latest/faq/sqlexpressions.html#how-do-i-render-sql-expressions-as-strings-possibly-with-bound-parameters-inlined), SQLAlchemy has limited ability to do query stringification, so it’s not so easy to get a final query which will be executed. You can print query by itself:

```python
print(query)

SELECT 
role_group_id, role_group_name, nr_patients 
FROM "StaffSummary"
WHERE day >= :day_1 AND day <= :day_2 AND location_id = :location_id_1 AND service_id = :service_id_1
```

(This one looks not so scary, but with more complex queries it could be about 20+ placeholders, which are very annoying and time-expensive to fill manually to play later in SQL console.)

If you have only strings and numbers to be inserted into query, this will work for you 

```python
print(s.compile(compile_kwargs={"literal_binds": True}))
```

For dates, such a trick will not work. There is [a discussion](https://stackoverflow.com/questions/5631078/sqlalchemy-print-the-actual-query) on StackOverflow on how to achieve the desired results, but solutions look unattractive. 

Another option is to enable queries logging into the file via database configuration, but in this case, you could face another issue; it becomes hard to find a query you want to debug if Django ORM connected to this database too.

## Testing  

Note: Pytest [multidb note](http://pytest-django.readthedocs.io/en/latest/database.html#tests-requiring-multiple-databases) says “Currently pytest-django does not specifically support Django’s multi-database support. You can however use normal Django TestCase instances to use it’s multi_db support.”

So what does it mean - not support? By default, Django will create and remove (at the end of all tests) a test-database for each db listed in `DATABASES` definition. This feature works perfectly with `pytests` also.

Django `TestCase` and `TransactionTestCase` with `multi_db=True` enables erasing of data in multiple databases between tests. Also it enables data loading into second database via `django-fixtures`, but it’s much better to use [model_mommy](https://model-mommy.readthedocs.io/en/latest/basic_usage.html) or [factory_boy](https://model-mommy.readthedocs.io/en/latest/basic_usage.html)  instead, which are not affected by this attribute. 

There are a few hacks suggested in [pytest-django discussion](https://github.com/pytest-dev/pytest-django/issues/76) how to work around the issue and enable `multi_db` to continue pytesting. 

There is one important advice - for tables that have Django-models, you should save data to DB via Django-ORM. Otherwise, you will face issues during writing tests. `TestCase` will not be able to rollback other transactions which happened outside from Django DB connection. If you have such a situation, you may use `TransactionalTestCase` with `multi_db=True` for tests which trigger functionality, which produces DB writes through SQLAlchemy connection, but remember that such tests are slower than regular `TestCase`.

Also, another scenario is possible - you have Django-models only in one database and you’re working with the second database via SQLAlchemy. In this case, `multi_db` doesn’t affect you at all. In such cases, you need to write a pytest-fixture (or do it as a mixin and trigger logic in `setUp` if you’re using unittests) which will create DB structure from SQL file. Such a file should contain `DROP TABLE IF EXISTS` statements before `CREATE TABLE`. This fixture should be applied to each test case which manipulates with this database. Other fixture could load data into created tables. 

Note: Such tests will be slower as tables will be recreated for each test. Ideally, tables should be created once (declared as `@pytest.fixture(scope='session', autouse=True)`), and each transaction should rollback data for each test. It’s not easy to achieve because of different connections: Django & SQLAlchemy or different connections of SQLAlchemy connection-pool, e.g in your tests you start the transaction, fill DB with test data, then run test and rollback transaction (it wasn’t committed). But during the test, your application code may do queries to DB like connection.execute(query) which performed outside of transaction which created test data. So with default transaction isolation level, the application will not see any data, only empty tables. It’s possible to change transaction isolation level to `READ UNCOMMITTED` for SQLAlchemy connection, and everything will work as expected, but it’s definitely not a solution at all. 

## Conclusion 

To sum up everything above, SQLAlchemy Core is a great tool which brings you closer to SQL and gives you understanding and full control over the queries. If you’re building the application (or a part of it) which requires advanced aggregations, it is worth it to check out SQLAlchemy Core capabilities as an alternative to Django ORM tools. 

Read on to learn how to make building advanced queries for data analysis projects easier. Find out how we managed to mix Django ORM and SQLAlchemy Core and what we got from it.

Originally published on [Django Stars blog](https://djangostars.com/blog/merging-django-orm-with-sqlalchemy-for-easier-data-analysis/). 
