---
layout:   post
title:    "Generating migrations with proper commands"
date:     2016-12-05 00:00:30 +0530
comments: true
---

### Creating a table

> If the migration name is of the form "CreateXXX" and is followed by a list of column names and types then a migration creating the table XXX with the columns listed will be generated.

```bash
$ rails g migration CreateUsers email:string:index
```

_db/migrate/20161201203703_create_users.rb_

```ruby
class CreateUsers < ActiveRecord::Migration
  def change
    create_table :users do |t|
      t.string :email
    end
    add_index :users, :email
  end
end
```

### Adding a new column to table

> If the migration name is of the form "AddXXXToYYY" or "RemoveXXXFromYYY" and is followed by a list of column names and types then a migration containing the appropriate add_column and remove_column statements will be created.

```sh
rails generate migration AddPartNumberToProducts part_number:string
```

_db/migrate/20161201204626_add_part_number_to_products.rb_

```ruby
class AddPartNumberToProducts < ActiveRecord::Migration
  def change
    add_column :products, :part_number, :string
  end
end
```

### Removing a column from table

```sh
rails generate migration RemovePartNumberFromProducts part_number:string
```

_db/migrate/20161201204909_remove_part_number_from_products.rb_

```ruby
class RemovePartNumberFromProducts < ActiveRecord::Migration
  def change
    remove_column :products, :part_number, :string
  end
end
```

### Adding a reference to table

```sh
rails generate migration AddUserRefToProducts user:references
```

_db/migrate/20161201205056_add_user_ref_to_products.rb_

```ruby
class AddUserRefToProducts < ActiveRecord::Migration
  def change
    add_reference :products, :user, index: true, foreign_key: true
  end
end
```

### Creating a join table

```sh
rails g migration CreateJoinTableCustomerProduct customer product
```

_db/migrate/20161201205222_create_join_table_customer_product.rb_

```ruby
class CreateJoinTableCustomerProduct < ActiveRecord::Migration
  def change
    create_join_table :customers, :products do |t|
      # t.index [:customer_id, :product_id]
      # t.index [:product_id, :customer_id]
    end
  end
end
```

### Adding a new column to table with modifiers

```sh
rails generate migration AddDetailsToProducts 'price:decimal{5,2}' supplier:references{polymorphic}
```

_db/migrate/20161201205408_add_details_to_products.rb_

```ruby
class AddDetailsToProducts < ActiveRecord::Migration
  def change
    add_column :products, :price, :decimal, precision: 5, scale: 2
    add_reference :products, :supplier, polymorphic: true, index: true
  end
end
```

#### List of column modifiers

|     Modifier    |  Description |
|-----------------|---|
| `limit`       | Sets the maximum size of the string/text/binary/integer fields. |
| `precision`   | Defines the precision for the decimal fields, representing the total number of digits in the number. |
| `scale`       | Defines the scale for the decimal fields, representing the number of digits after the decimal point. |
| `polymorphic` | Adds a type column for belongs_to associations. |
| `null`        | Allows or disallows NULL values in the column. |
| `default`     | Allows to set a default value on the column. Note that if you are using a dynamic value (such as a date), the default will only be calculated the first time (i.e. on the date the migration is applied). |
| `index`       | Adds an index for the column. |
| `comment`     | Adds a comment for the column. |
