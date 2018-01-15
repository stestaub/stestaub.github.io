---
layout: post
title:  "Pg Inheritance with Rails"
date:   2018-01-15 12:00:00 +0100
categories: rails postgres inheritance sti
---

I just recently stumbled upon a feature of postgres I didn't know about yet, its called **pg_inheritance**.
It is basically the possibility of creating child tables which inherit the columns of the base table, although
other things such as constraints. The main use case for this is table partitioning, but as I read the [docs](https://www.postgresql.org/docs/current/static/ddl-inherit.html)
I was thinking it could might be useful for Model inheritance in rails as well. 

So lets start with a small sample rails app to test this out:

```bash
rails new myapp --database=postgresql
```

The first thing we need to do is change Schema type to sql in config/application.rb as we will be adding some native sql in the migrations

```ruby
config.active_record.schema_format = :sql
```

Now just scaffold a sample model (based on the example in the postgres docs):

```bash
rails g scaffold model City name:string population:float altitude:integer
```

And a Capital model with some additional information

```bash
rails g model Capital state:string
```

In order to use table inheritance of postgres, we need to change the migration to something like this:

```ruby
class CreateCapitals < ActiveRecord::Migration[5.1]
  def up
    execute 'CREATE TABLE capitals (state char(2)) INHERITS (cities);'
  end

  def down
    execute 'DROP TABLE capitals;'
  end
end
```

And we want to inherit the model from City:

```ruby
class Capital < City  
end
```

After running the migration we now fire up the rails console. But if we try to call any AR related method on Capital we get an error:

```
irb(main):003:0> Capital.all
  Capital Load (0.7ms)  SELECT  "cities".* FROM "cities" WHERE "cities"."type" IN ('Capital') LIMIT $1  [["LIMIT", 11]]
ActiveRecord::StatementInvalid: PG::UndefinedColumn: ERROR:  column cities.type does not exist
LINE 1: SELECT  "cities".* FROM "cities" WHERE "cities"."type" IN ('...
                                               ^
: SELECT  "cities".* FROM "cities" WHERE "cities"."type" IN ('Capital') LIMIT $1

```

This is because the inheritance in the model automatically expects it to be STI, which is using the cities table and expecting it to have
 a column named type.
 
So we need to find a way to work around the default behaviour of rails.
 
If we look into the implementation of [ActiveRecord::Inheritance](http://api.rubyonrails.org/classes/ActiveRecord/Inheritance/ClassMethods.html) we find a method called base_class. This is used
for sti to find the base class to inherit from. By overriding this method we can tell ActiveRecord to use the current
class as the base class, which will keep Rails from using STI for this model:

```ruby
class Capital < City
  def self.base_class
    self
  end
end
```

So we have now STI disabled but still inheriting from City. And as our database table also inherited the columns from
its parent table, we should now be able to see these attributes on the Capital model. We test this with a small spec:

```ruby
require 'rails_helper'

RSpec.describe Capital, type: :model do

  it 'should inherit attributes from citites' do
    expect(subject).to respond_to :name
    expect(subject).to respond_to :population
    expect(subject).to respond_to :altitude
  end

  it 'should have the custom attributes' do
    expect(subject).to respond_to :state
  end

end
```

When we run this tests we see that those tests are green and we have successfully set up Model inheritance on multiple tables
with postgres pg_inheritance.

Now lets do some additional tests by adding some validation to the base class:

```ruby
class City < ApplicationRecord
  validates :name, length: { minimum: 1, maximum: 50 }, presence: true
  validates :population, numericality: { greater_than: 0 }, presence: true
end
```

In our Capital model, we also should have these validations now. Lets test that

```ruby
  it 'should inherit validations' do
    expect(subject).to validate_presence_of(:name)
    expect(subject).to validate_presence_of(:population)
  end
```

And as expected due to the class inheritance, this works as it should.

Now what about retrieving records? Lets start the rails console and create a Capitol.

```
cap = Capital.create(name: "Bern", population: 20000, state: "CH")
=> #<Capital id: nil, name: "Bern", population: 20000.0, altitude: nil, created_at: "2018-01-15 22:10:12", updated_at: "2018-01-15 22:10:12", state: "CH">
cap.persisted?
=> true
```

As this record is inherited from City, it should also be possible to retrieve it using the City model:

```
City.first
=> #<City id: 1, name: "Bern", population: 20000.0, altitude: nil, created_at: "2018-01-15 22:07:50", updated_at: "2018-01-15 22:07:50">
```

But retrieving it by id gives us an error:

```
Capital.find 1
=> ActiveRecord::UnknownPrimaryKey: Unknown primary key for table capitals in model Capital.
```

Why is that? pg_inheritance does only inherit columns, but not constraints. The generation of the id works, as the cities table
uses a sequence as default value for the id column (which is inherited), but as the constraint is missing, rails has no way to identify it
as primary key.

To solve this we create a new migration and specify add a constraint to the id column:

```ruby
class AddPrimaryKeyConstraintToCapital < ActiveRecord::Migration[5.1]
  def up
    execute 'ALTER TABLE capitals ADD CONSTRAINT capitals_pkey PRIMARY KEY(id);'
  end

  def down
    execute 'ALTER TABLE capitals DROP CONSTRAINT capitals_pkey;'
  end
end
```

Now we also can retrieve capitals by id and update their attributes.

# Getting back STI functionality
At this point, we have a working multitable inheritance but features that STI provides, such as automatically cast to the correct
subclass, are not available.

Getting those features is actually easier than I thought. We just add an other migration and add a type column
to the base table:

```ruby
class AddTypeToCity < ActiveRecord::Migration[5.1]
  def change
    add_column :cities, :type, :string
    add_index :cities, :type
    add_index :capitals, :type
  end
end
```

With can now test this with a spec:

```ruby
it 'should create capital through base class' do
    expect {
      City.create name: 'test', population: 1, type: 'Capital', state: 'CH'
    }.to change(Capital, :count).by 1
  end

  it 'should cast to captal if retrieved through base class' do
    Capital.create name: 'test', population: 1, type: 'Capital', state: 'CH'
    expect(City.last).to be_a Capital
  end
```

# Conclusion
This method seems actually pretty interesting to me. It allows to create multitable inheritance with the
goodness of rails STI. Of course there are some downsides as well:

* It is database dependent (postgres in this specific case)
* SQL schema_type is required as we need native sql in the migrations
* Database Uniqueness checks are only performed on a per table basis (although you could solve that with a trigger)
* Caveats described here: [https://www.postgresql.org/docs/current/static/ddl-inherit.html#DDL-INHERIT-CAVEATS](https://www.postgresql.org/docs/current/static/ddl-inherit.html#DDL-INHERIT-CAVEATS)

And there are maybe more which I didn't think about yet.