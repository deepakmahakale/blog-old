---
layout:   post
title:    Rails and TimeZone issue with query
date:     2018-04-21 13:08:58 +0530
comments: true
---
# Rails and TimeZone Issue with query

You must have faced timezone issue if you work with the application in different timezone.
ActiveRecord does take care of the timezone while creating, updating or querying the records.
The DateTime is converted to UTC before the query is executed.

You will face issue if you are using raw queries or passing `String` instead of `DateTime`.

# DateTime conversion while creating the records

Let's create a few records in database to see how ActiveRecord converts the DateTime while creating the record.

We are going to create following records:
1. Today: beginning of day
2. Today: end of day
3. Tomorrow: beginning of day

```ruby
today_beginning = '2018-04-21 00:00:00'.to_time
#=> 2018-04-21 00:00:00 +0530

Message.create(text: 'Today: beginning of day', created_at: today_beginning, updated_at: today_beginning)
  SQL (1.0ms)  INSERT INTO "messages" ("text", "created_at", "updated_at") VALUES (?, ?, ?)  [["text", "Today: beginning of day"], ["created_at", "2018-04-20 18:30:00"], ["updated_at", "2018-04-20 18:30:00"]]
#<Message:0x000000000404b5d8> {
            :id => 1,
          :text => "Today: beginning of day",
    :created_at => Sat, 21 Apr 2018 00:00:00 IST +05:30,
    :updated_at => Sat, 21 Apr 2018 00:00:00 IST +05:30
}

today_end = '2018-04-21 23:59:59'.to_time
#=> 2018-04-21 23:59:59 +0530

Message.create(text: 'Today: end of day', created_at: today_end, updated_at: today_end)
  SQL (8.0ms)  INSERT INTO "messages" ("text", "created_at", "updated_at") VALUES (?, ?, ?)  [["text", "Today: end of day"], ["created_at", "2018-04-21 18:29:59"], ["updated_at", "2018-04-21 18:29:59"]]
#<Message:0x00000000027d2588> {
            :id => 2,
          :text => "Today: end of day",
    :created_at => Sat, 21 Apr 2018 23:59:59 IST +05:30,
    :updated_at => Sat, 21 Apr 2018 23:59:59 IST +05:30
}

tomorrow_beginning = '2018-04-22 00:00:00'.to_time
#=> 2018-04-22 00:00:00 +0530

Message.create(text: 'Tomorrow: beginning of day', created_at: tomorrow_beginning, updated_at: tomorrow_beginning)
  SQL (1.4ms)  INSERT INTO "messages" ("text", "created_at", "updated_at") VALUES (?, ?, ?)  [["text", "Tomorrow: beginning of day"], ["created_at", "2018-04-21 18:30:00"], ["updated_at", "2018-04-21 18:30:00"]]
#<Message:0x0000000005092cc0> {
            :id => 3,
          :text => "Tomorrow: beginning of day",
    :created_at => Sun, 22 Apr 2018 00:00:00 IST +05:30,
    :updated_at => Sun, 22 Apr 2018 00:00:00 IST +05:30
}
```

Notice that ActiveRecord changed the time from `Sat, 21 Apr 2018 00:00:00 IST +05:30` to `2018-04-20 18:30:00` before executing the query.

# DateTime conversion while fetching the records

Let's query on the records we have just created.

```ruby
Time.current.beginning_of_day
#=> Sat, 21 Apr 2018 00:00:00 IST +05:30

Time.current.end_of_day
#=> Sat, 21 Apr 2018 23:59:59 IST +05:30

Message.where(created_at: Time.current.beginning_of_day..Time.current.end_of_day)
  Message Load (0.2ms)  SELECT "messages".* FROM "messages" WHERE ("messages"."created_at" BETWEEN ? AND ?)  [["created_at", "2018-04-20 18:30:00"], ["created_at", "2018-04-21 18:29:59.999999"]]
[
    [0] #<Message:0x0000000004efb380> {
                :id => 1,
              :text => "Today: beginning of day",
        :created_at => Sat, 21 Apr 2018 00:00:00 IST +05:30,
        :updated_at => Sat, 21 Apr 2018 00:00:00 IST +05:30
    },
    [1] #<Message:0x0000000004efb1f0> {
                :id => 2,
              :text => "Today: end of day",
        :created_at => Sat, 21 Apr 2018 23:59:59 IST +05:30,
        :updated_at => Sat, 21 Apr 2018 23:59:59 IST +05:30
    }
]

Message.where("created_at BETWEEN ? AND ?", Time.current.beginning_of_day, Time.current.end_of_day)
  Message Load (0.4ms)  SELECT "messages".* FROM "messages" WHERE (created_at BETWEEN '2018-04-20 18:30:00' AND '2018-04-21 18:29:59.999999')
[
    [0] #<Message:0x0000000004eb6028> {
                :id => 1,
              :text => "Today: beginning of day",
        :created_at => Sat, 21 Apr 2018 00:00:00 IST +05:30,
        :updated_at => Sat, 21 Apr 2018 00:00:00 IST +05:30
    },
    [1] #<Message:0x0000000004eb5e48> {
                :id => 2,
              :text => "Today: end of day",
        :created_at => Sat, 21 Apr 2018 23:59:59 IST +05:30,
        :updated_at => Sat, 21 Apr 2018 23:59:59 IST +05:30
    }
]
```

This works if you are passing the `DateTime` object.
But you will get the incorrect result when you pass `String`. You can see the TimeZone in the string but ActiveRecord will not convert the time to UTC.

```ruby
Time.current.beginning_of_day.to_s
#=> "2018-04-21 00:00:00 +0530"

Time.current.end_of_day.to_s
#=> "2018-04-21 23:59:59 +0530"

Message.where("created_at BETWEEN ? AND ?", Time.current.beginning_of_day.to_s, Time.current.end_of_day.to_s)
  Message Load (0.2ms)  SELECT "messages".* FROM "messages" WHERE (created_at BETWEEN '2018-04-21 00:00:00 +0530' AND '2018-04-21 23:59:59 +0530')
[
    [0] #<Message:0x0000000004e10df8> {
                :id => 2,
              :text => "Today: end of day",
        :created_at => Sat, 21 Apr 2018 23:59:59 IST +05:30,
        :updated_at => Sat, 21 Apr 2018 23:59:59 IST +05:30
    },
    [1] #<Message:0x0000000004e10c40> {
                :id => 3,
              :text => "Tomorrow: beginning of day",
        :created_at => Sun, 22 Apr 2018 00:00:00 IST +05:30,
        :updated_at => Sun, 22 Apr 2018 00:00:00 IST +05:30
    }
]
```

ActiveRecord assumes that the time is already in UTC and executes the query without changing the time.


# How to avoid timezone issue in raw SQL query.

### Solution 1: If possible use `find_by_sql`

To avoid SQL injection you can use `find_by_sql`. Pass the values and it will be converted to proper timezone.

```ruby
Message.find_by_sql(["SELECT * FROM messages where created_at BETWEEN ? AND ?", start_time, end_time])
  Message Load (1.0ms)  SELECT * FROM messages where created_at BETWEEN '2018-04-20 18:30:00' AND '2018-04-21 18:29:59'
[
    [0] #<Message:0x00000000045abb90> {
                :id => 1,
              :text => "Today: beginning of day",
        :created_at => Sat, 21 Apr 2018 00:00:00 IST +05:30,
        :updated_at => Sat, 21 Apr 2018 00:00:00 IST +05:30
    },
    [1] #<Message:0x00000000045aba50> {
                :id => 2,
              :text => "Today: end of day",
        :created_at => Sat, 21 Apr 2018 23:59:59 IST +05:30,
        :updated_at => Sat, 21 Apr 2018 23:59:59 IST +05:30
    }
]

```

### Solution 2: Convert the time to UTC before querying

If you have no other option but to use raw query and interpolate the values, convert the time to utc.

```ruby
start_time = "2018-04-21 00:00:00".to_time.utc
#=> 2018-04-20 18:30:00 UTC
end_time = "2018-04-21 23:59:59".to_time.utc
#=> 2018-04-21 18:29:59 UTC

query = <<~SQL
  SELECT *
  FROM messages
  WHERE created_at BETWEEN '#{start_time}' AND '#{end_time}';
SQL

result = ActiveRecord::Base.connection.execute(query)
  (0.8ms)  SELECT *
  FROM messages
  WHERE created_at BETWEEN '2018-04-20 18:30:00 UTC' AND '2018-04-21 18:29:59 UTC';

result.as_json

[
    [0] {
                "id" => 1,
              "text" => "Today: beginning of day",
        "created_at" => "2018-04-20 18:30:00",
        "updated_at" => "2018-04-20 18:30:00"
    },
    [1] {
                "id" => 2,
              "text" => "Today: end of day",
        "created_at" => "2018-04-21 18:29:59",
        "updated_at" => "2018-04-21 18:29:59"
    }
]

```


### Solution 3: [PostgreSQL] Use `TIMESTAMP WITH TIME ZONE`

This is the way to tell postgres that the Time passed is in some timezone and you need to convert it to UTC first.

```ruby
start_time = '2018-04-21 00:00:00'
#=> "2018-04-21 00:00:00"
end_time = '2018-04-21 23:59:59'
#=> "2018-04-21 23:59:59"

query = <<~SQL
  SELECT *
  FROM messages
  WHERE created_at BETWEEN TIMESTAMP WITH TIME ZONE '#{start_time}'
                       AND TIMESTAMP WITH TIME ZONE '#{end_time}';
SQL

result = ActiveRecord::Base.connection.execute(query)
  (1.2ms)  SELECT *
    FROM messages
    WHERE created_at BETWEEN TIMESTAMP WITH TIME ZONE '2018-04-21 00:00:00'
                         AND TIMESTAMP WITH TIME ZONE '2018-04-21 23:59:59';

result.as_json

[
    [0] {
                "id" => 3,
              "text" => "Tomorrow: beginning of day",
        "created_at" => "2018-04-21 18:30:00",
        "updated_at" => "2018-04-21 18:30:00"
    },
    [1] {
                "id" => 2,
              "text" => "Today: end of day",
        "created_at" => "2018-04-21 18:29:59",
        "updated_at" => "2018-04-21 18:29:59"
    }
]
```

# TL;DR

1. Always use [`Time.current`][time-current]{:target='_blank'} or [`Date.current`][date-current]{:target='_blank'} while querying
2. Make sure to use [`beginning_of_day`][beginning_of_day]{:target='_blank'} & [`end_of_day`][end_of_day]{:target='_blank'} if you need the records created for whole day.
3. Use [`find_by_sql`][find_by_sql]{:target='_blank'} if possible.
4. Convert Date to UTC if you are using raw query.
5. Use [`TIMESTAMP WITH TIME ZONE`][datatype-datetime]{:target='_blank'} in raw query if you are using PostgreSQL.


## References

  - [http://api.rubyonrails.org][rubyonrails-doc]{:target='_blank'}
  - [https://www.postgresql.org][postgresql-doc]{:target='_blank'}

[time-current]: http://api.rubyonrails.org/classes/Time.html#method-c-current
[date-current]: http://api.rubyonrails.org/classes/Date.html#method-c-current
[beginning_of_day]: http://api.rubyonrails.org/classes/Time.html#method-i-beginning_of_day
[end_of_day]: http://api.rubyonrails.org/classes/Time.html#method-i-end_of_day
[find_by_sql]: http://api.rubyonrails.org/classes/ActiveRecord/Querying.html#method-i-find_by_sql
[datatype-datetime]: https://www.postgresql.org/docs/9.1/static/datatype-datetime.html
[rubyonrails-doc]: http://api.rubyonrails.org
[postgresql-doc]: https://www.postgresql.org


