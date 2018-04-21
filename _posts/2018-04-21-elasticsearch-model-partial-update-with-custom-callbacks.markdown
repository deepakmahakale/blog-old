---
layout:   post
title:    Elasticsearch::Model - Partial update with custom callbacks
date:     2018-04-21 22:28:35 +0530
comments: true
---

If you want to partially update the documents on Elasticsearch you can include `Elasticsearch::Model::Callbacks` module.

You can also perform partial update with custom callbacks instead of indexing the whole document.

> Elasticsearch Version - 5.3

# Partially update a document

There is [`update_document_attributes`][update_document_attributes]{:target='_blank'} method in `Elasticsearch::Model` which is used for partial update but it's not documented.
You can still use the method to perform the partial update.

```ruby
# https://github.com/elastic/elasticsearch-rails/blob/7815039c0f78ed0b9b896936875ee4d01855390e/elasticsearch-model/lib/elasticsearch/model/indexing.rb#L432-L439
def update_document_attributes(attributes, options={})
  client.update(
    { index: index_name,
      type:  document_type,
      id:    self.id,
      body:  { doc: attributes } }.merge(options)
  )
end
```

You can use this method in a callback to update the document manually.

```ruby
class User < ActiveRecord::Base
  include Elasticsearch::Model
  after_update :update_first_name, if: -> { first_name_changed? }

  #...

  def update_first_name
    __elasticsearch__.update_document_attributes({first_name: 'John'})
  end
end

```

This will made a update call only with the changed field

```ruby
2018-04-21 22:18:25 +0530: POST http://localhost:9200/test_users/user/1/_update [status:200, request:0.005s, query:n/a]
2018-04-21 22:18:25 +0530: > {"doc":{"first_name": "John"}}
2018-04-21 22:18:25 +0530: < {"_index":"test_users","_type":"user","_id":"1","_version":3,"result":"noop","_shards":{"total":0,"successful":0,"failed":0}}
```

as opposed to the one with `index_document`

```ruby
2018-04-21 22:37:51 +0530: PUT http://localhost:9200/test_users/user/1 [status:200, request:0.104s, query:n/a]
2018-04-21 22:37:51 +0530: > {"id":1,"first_name":"John","last_name":"Snow","email":"john@winterfell.com"}
2018-04-21 22:37:51 +0530: < {"_index":"test_users","_type":"user","_id":"1","_version":4,"result":"updated","_shards":{"total":1,"successful":1,"failed":0},"created":false}
```

This method allows you to partially update the document even if you are using custom callbacks.

## References

  - [Elasticsearch official documentation][elastic-documentation]{:target='_blank'}
  - [Elasticsearch::Model][elasticsearch-model]{:target='_blank'}

[elastic-documentation]: https://www.elastic.co/guide/en/elasticsearch/reference/5.3/index.html
[elasticsearch-model]: https://github.com/elastic/elasticsearch-rails/tree/master/elasticsearch-model
[update_document_attributes]: https://github.com/elastic/elasticsearch-rails/blob/7815039c0f78ed0b9b896936875ee4d01855390e/elasticsearch-model/lib/elasticsearch/model/indexing.rb#L432-L439
