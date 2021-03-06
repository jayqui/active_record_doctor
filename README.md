# Active Record Doctor

Active Record Doctor helps to keep the database in a good shape. Currently, it
can:

* index unindexed foreign keys
* detect extraneous indexes
* detect unindexed `deleted_at` columns
* detect missing foreign key constraints
* detect models referencing undefined tables
* detect uniqueness validations not backed by an unique index

More features coming soon!

Want to suggest a feature? Just shoot me [an email](mailto:contact@gregnavis.com).

[<img src="https://travis-ci.org/gregnavis/active_record_doctor.svg?branch=master" alt="Build Status" />](https://travis-ci.org/gregnavis/active_record_doctor)

## Installation

The preferred installation method is adding `active_record_doctor` to your
`Gemfile`:

```ruby
gem 'active_record_doctor', group: :development
```

Then run:

```bash
bundle install
```

## Usage

### Indexing Unindexed Foreign Keys

Foreign keys should be indexed unless it's proven ineffective. However, Rails
makes it easy to create an unindexed foreign key. Active Record Doctor can
automatically generate database migrations that add the missing indexes. It's a
three-step process:

1. Generate a list of unindexed foreign keys by running

  ```bash
  rake active_record_doctor:unindexed_foreign_keys > unindexed_foreign_keys.txt
  ```

2. Remove columns that should _not_ be indexed from `unindexed_foreign_keys.txt`
   as a column can look like a foreign key (i.e. end with `_id`) without being
   one.

3. Generate the migrations

  ```bash
  rails generate active_record_doctor:add_indexes unindexed_foreign_keys.txt
  ```

4. Run the migrations

  ```bash
  rake db:migrate
  ```

### Removing Extraneous Indexes

Let me illustrate with an example. Consider a `users` table with columns
`first_name` and `last_name`. If there are two indexes:

* A two-column index on `last_name, first_name`.
* A single-column index on `last_name`.

Then the latter index can be dropped as the former can play its role. In
general, a multi-column index on `column_1, column_2, ..., column_n` can replace
indexes on:

* `column_1`
* `column_1, column_2`
* ...
* `column_1, column_2, ..., column_(n - 1)`

To discover such indexes automatically just follow these steps:

1. List extraneous indexes by running:

  ```bash
  rake active_record_doctor:extraneous_indexes
  ```

2. Confirm that each of the indexes can be indeed dropped.

3. Create a migration to drop the indexes.

The indexes aren't dropped automatically because there's usually just a few of
them and it's a good idea to double-check that you won't drop something
necessary.

Also, extra indexes on primary keys are considered extraneous too and will be
reported.

Note that a unique index can _never be replaced by a non-unique one_. For
example, if there's a unique index on `users.login` and a non-unique index on
`users.login, users.domain` then the tool will _not_ suggest dropping
`users.login` as it could violate the uniqueness assumption.

### Detecting Unindexed `deleted_at` Columns

If you soft-delete some models (e.g. with `paranoia`) then you need to modify
your indexes to include only non-deleted rows. Otherwise they will include
logically non-existent rows. This will make them larger and slower to use. Most
of the time they should only cover columns satisfying `deleted_at IS NULL`.

`active_record_doctor` can automatically detect indexes on tables with a
`deleted_at` column. Just run:

```
rake active_record_doctor:unindexed_soft_delete
```

This will print a list of indexes that don't have the `deleted_at IS NULL`
clause. Currently, `active_record_doctor` cannot automatically generate
appropriate migrations. You need to do that manually.

### Detecting Missing Foreign Key Constraints

If `users.profile_id` references a row in `profiles` then this can be expressed
at the database level with a foreign key constraint. It _forces_
`users.profile_id` to point to an existing row in `profiles`. The problem is
that in many legacy Rails apps the constraint isn't enforced at the database
level.

`active_record_doctor` can automatically detect foreign keys that could benefit
from a foreign key constraint (a future version will generate a migrations that
add the constraint; for now, it's your job). You can obtain the list of foreign
keys with the following command:

```bash
rake active_record_doctor:missing_foreign_keys
```

The output will look like:

```
users profile_id
comments user_id article_id
```

Tables are listed one per line. Each line starts with a table name followed by
column names that should have a foreign key constraint. In the example above,
`users.profile_id`, `comments.user_id`, and `comments.article_id` lack a foreign
key constraint.

In order to add a foreign key constraint to `users.profile_id` use the following
migration:

```ruby
class AddForeignKeyConstraintToUsersProfileId < ActiveRecord::Migration
  def change
    add_foreign_key :users, :profiles
  end
end
```

### Detecting Models Referencing Undefined Tables

Active Record guesses the table name based on the class name. There are a few
cases where the name can be wrong (e.g. you forgot to commit a migration or
changed the table name). Active Record Doctor can help you identify these cases
before they hit production.

The only  think you need to do is run:

```
rake active_record_doctor:undefined_table_references
```

If there a model references an undefined table then you'll see a message like
this:

```
The following models reference undefined tables:
  Contract (the table contract_records is undefined)
```

On top of that `rake` will exit with status code of 1. This allows you to use
this check as part of your Continuous Integration pipeline.

### Detecting Uniqueness Validations not Backed by an Index

A model-level uniqueness validations should be backed by a database index in
order to be robust. Otherwise you risk inserting duplicate values under heavy
load.

In order to detect such validations run:

```
rake active_record_doctor:missing_unique_indexes
```

If there are such indexes then the command will print:

```
The following indexes should be created to back model-level uniqueness validations:
  users: email
```

This means that you should create a unique index on `users.email`.

## Author

This gem is developed and maintained by [Greg Navis](http://www.gregnavis.com).
