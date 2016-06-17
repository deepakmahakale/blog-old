---
layout:   post
title:    "Include scoped associations"
date:     2016-06-16 14:00:58 +0530
comments: true
---

Rails provide different ways to load associations from the database to avoid [N + 1 query][eager-loading-associations] being fired.
In this post, I will show you how to `include` associations with scope applied on them.

I have two models `User` and `Post` with following associations:

```ruby
# app/models/user.rb
class User < ActiveRecord::Base
  has_many :posts, dependent: :destroy
end

# app/models/post.rb
class Post < ActiveRecord::Base
  belongs_to :user
  scope :published, -> { where(published: true) }
end
```

Now if you loop over the association you will see N + 1 queries being fired

```ruby
User.all.map do |user|
  [user.name, user.posts.map(&:title).join(', ')]
end
User Load (0.3ms)  SELECT "users".* FROM "users"
Post Load (0.4ms)  SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = ?  [["user_id", 1]]
Post Load (0.2ms)  SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = ?  [["user_id", 2]]
Post Load (0.1ms)  SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = ?  [["user_id", 3]]
#=> [["Jack", "post-1, post-2"], ["Adam", "post-3, post-3"], ["John", "post-5, post-6"]]
```

Now let's try including posts

```ruby
User.includes(:posts).map do |user|
  [user.name, user.posts.map(&:title).join(', ')]
end
User Load (0.2ms)  SELECT "users".* FROM "users"
Post Load (0.3ms)  SELECT "posts".* FROM "posts" WHERE "posts"."user_id" IN (1, 2, 3)
#=> [["Jack", "post-1, post-2"], ["Adam", "post-3, post-3"], ["John", "post-5, post-6"]]
```
Well it worked. But what if I want to apply published scope on posts.

```ruby
User.includes(:posts).map do |user|
  [user.name, user.posts.published.map(&:title).join(', ')]
end
User Load (0.2ms)  SELECT "users".* FROM "users"
Post Load (0.3ms)  SELECT "posts".* FROM "posts" WHERE "posts"."user_id" IN (1, 2, 3)
Post Load (0.1ms)  SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = ? AND "posts"."published" = 't'  [["user_id", 1]]
Post Load (0.1ms)  SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = ? AND "posts"."published" = 't'  [["user_id", 2]]
Post Load (0.1ms)  SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = ? AND "posts"."published" = 't'  [["user_id", 3]]
#=> [["Jack", "post-1, post-2"], ["Adam", "post-3, post-3"], ["John", "post-5, post-6"]]
```

It fired N + 1 queries.
Let's fix this by creating a new association with a condition.

```ruby
# app/models/user.rb
class User < ActiveRecord::Base
  has_many :posts, dependent: :destroy
  has_many :published_posts, -> { where(published: true) }, class_name: 'Post'
end

# app/models/post.rb
class Post < ActiveRecord::Base
  belongs_to :user
  scope :published, -> { where(published: true) }
end
```

Now we will include `published_posts` instead `posts` and it will load posts with `published` scope applied to them.

```ruby
User.includes(:published_posts).map do |user|
  [user.name, user.published_posts.map(&:title).join(', ')]
end
User Load (0.3ms)  SELECT "users".* FROM "users"
Post Load (0.3ms)  SELECT "posts".* FROM "posts" WHERE (published = 't') AND "posts"."user_id" IN (1, 2, 3)
#=> [["Jack", "post-1, post-2"], ["Adam", "post-3, post-3"], ["John", "post-5, post-6"]]

```

You can use scope instead of condition so that you don't have to change it twice in case you change the scope.

```ruby
# app/models/user.rb
class User < ActiveRecord::Base
  has_many :posts, dependent: :destroy
  has_many :published_posts, -> { published }, class_name: 'Post'
end

# app/models/post.rb
class Post < ActiveRecord::Base
  belongs_to :user
  scope :published, -> { where(published: true) }
end
```

Voila!

[eager-loading-associations]: http://guides.rubyonrails.org/active_record_querying.html#eager-loading-associations
