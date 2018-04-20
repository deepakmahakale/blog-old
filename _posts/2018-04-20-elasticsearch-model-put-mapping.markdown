---
layout:   post
title:    Elasticsearch::Model - Put Mapping
date:     2018-04-20 13:38:58 +0530
comments: true
---
# Elasticsearch::Model - [Put Mapping][put-mapping]{:target='_blank'}

  The PUT mapping API allows you to add fields to an existing index.
  This method is available with Elasticsearch::Model but it's not documented.

  > Elasticsearch Version - 5.3

## Create a User model

```bash
> rails g model user first_name:string last_name:string email:string phone_number:string

invoke  active_record
  create    db/migrate/20180420162454_create_users.rb
  create    app/models/user.rb

> rake db:migrate
```

Seed the data

```ruby
10.times do
  name = Faker::Name.first_name
  User.create!(
    first_name: name,
    last_name: Faker::Name.last_name,
    email: Faker::Internet.email(name),
    phone_number: Faker::PhoneNumber.phone_number
  )
end
```

## Add mapping

```ruby
# app/models/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  included do
    include Elasticsearch::Model
    mapping do
      indexes :id, type: :integer
      indexes :first_name, type: :text
      indexes :last_name, type: :text
      indexes :email, type: :text
    end
  end
end

#app/models/user.rb
class User < ActiveRecord::Base
  include Searchable

  def as_indexed_json(_options = {})
    as_json(only: %i[id first_name last_name email phone_number])
  end
end
```

## Create index and import all users

```ruby
> User.__elasticsearch__.create_index!
{
           "acknowledged" => true,
    "shards_acknowledged" => true,
                  "index" => "users"
}

> User.import
```

**You can search users now**

```ruby
> response = User.search 'vesta'
> response.results.as_json

[
  {
     "_index" => "users",
      "_type" => "user",
        "_id" => "10",
     "_score" => 1.6348188,
    "_source" => {
                "id" => 10,
        "first_name" => "Vesta",
         "last_name" => "Toy",
             "email" => "vesta@bradtkeconnelly.biz"
    }
  }
]
```
## Add new fields

Now to change the mapping just add new field and update the mapping

```ruby
mapping do
  #...
  indexes :phone_number, type: :text
end
```

## Update the mapping using Put Mapping API

```ruby
class User < ActiveRecord::Base
  # ...
  def self.put_mapping
    __elasticsearch__.client.indices.put_mapping(
      index: index_name,
      type:  document_type,
      body:  mapping.to_hash
    )
  end
end
```

## Bulk Update

Now you need to update all the documents. we can use the bulk update API.

```ruby
def self.bulk_update
  find_in_batches do |users|
    __elasticsearch__.client.bulk(
      index: index_name,
      type: __elasticsearch__.document_type,
      body: processed_records(users)
    )
  end
end

def self.processed_records(users)
  users.map do |user|
    {
      update: {
        _id: user.id,
        data: {
          doc: {
            id: user.id,
            phone_number: user.phone_number
          }.as_json
        }
      }
    }
  end
end
```

## References

  - [Elasticsearch official documentation][elastic-documentation]{:target='_blank'}
  - [Elasticsearch::Model][elasticsearch-model]{:target='_blank'}


[elastic-documentation]: https://www.elastic.co/guide/en/elasticsearch/reference/5.3/index.html
[elasticsearch-model]: https://github.com/elastic/elasticsearch-rails/tree/master/elasticsearch-model
[put-mapping]: https://www.elastic.co/guide/en/elasticsearch/reference/5.3/indices-put-mapping.html
