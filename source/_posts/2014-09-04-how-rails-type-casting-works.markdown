---
layout: post
title: "How Rails' Type Casting Works"
date: 2013-12-04 00:04:20 +0800
comments: true
categories: ["Ruby", "Rails"]
has_image: false
---

Have you ever noticed that when you assign a property to an Active Record model and read it back, the value isn’t always the same? Here’s an example:

```ruby
class StoreListing < ActiveRecord::Base
  connection.create_table :store_listings, force: true do |t|
    t.integer :price_in_cents
  end
end

store_listing = StoreListing.new
store_listing.price_in_cents = "100" # Note, this is a string
store_listing.price_in_cents # => 100
```

This is because Active Record automatically type casts all input so that it matches the database schema. Depending on the type, this may be incredibly simple, or extremely complex. Let’s take a look into how the internals work in 4.1.

The first method we need to understand in the above code is where the price_in_cents method is defined. In older versions of Rails, your models would go look up the database schema and define the attribute methods as soon as it was loaded. However, this caused problems on platforms like Heroku, where you might want to load the application when you don’t have a real database connection.

Today, the loading is lazy, and happens in a call to `method_missing` (source). The important line here is the call to `define_attribute_methods`.

```ruby
def method_missing(method, *args, &block) # :nodoc:
  self.class.define_attribute_methods
  if respond_to_without_attributes?(method)
    # make sure to invoke the correct attribute method, as we might have gotten here via a `super`
    # call in a overwritten attribute method
    if attribute_method = self.class.find_generated_attribute_method(method)
      # this is probably horribly slow, but should only happen at most once for a given AR class
      attribute_method.bind(self).call(*args, &block)
    else
      return super unless respond_to_missing?(method, true)
      send(method, *args, &block)
    end
  else
    super
  end
end
```

Active Record’s definition of `define_attribute_methods` does little of note, other than call super with `column_names` (source).

```ruby
def define_attribute_methods # :nodoc:
  return false if @attribute_methods_generated
  # Use a mutex; we don't want two thread simultaneously trying to define
  # attribute methods.
  generated_attribute_methods.synchronize do
    return false if @attribute_methods_generated
    superclass.define_attribute_methods unless self == base_class
    super(column_names)
    @attribute_methods_generated = true
  end
  true
end
```

We won’t look into how `column_names` gets determined today, but that method call is what causes Rails to go perform the SQL query that loads information about the model’s schema. Inside of Active Model, we’ll do some metaprogramming magic and ultimately end up calling `define_method_attribute` (source). Finally, in the body of `define_method_attribute`, we can see the method that gets called is `read_attribute` (source). Quite a bit of legwork!

If you decide to read along with us, make sure you’re on the 4-1-stable branch. A lot of this code has changed significantly on master. One of the most important changes to keep in mind is that `@attributes_cache` has been renamed to @attributes, and @attributes has been renamed to `@raw_attributes`.

The body of `read_attribute` looks like this:

```ruby
def read_attribute(attr_name)
  # If it's cached, just return it
  # We use #[] first as a perf optimization for non-nil values. See https://gist.github.com/jonleighton/3552829.
  name = attr_name.to_s
  @attributes_cache[name] || @attributes_cache.fetch(name) {
    column = @column_types_override[name] if @column_types_override
    column ||= @column_types[name]

    return @attributes.fetch(name) {
      if name == 'id' && self.class.primary_key != name
        read_attribute(self.class.primary_key)
      end
    } unless column

    value = @attributes.fetch(name) {
      return block_given? ? yield(name) : nil
    }

    if self.class.cache_attribute?(name)
      @attributes_cache[name] = column.type_cast(value)
    else
      column.type_cast value
    end
  }
end
```

Let’s go through each segment and understand what it’s doing. First we call `to_s` on the argument, as it’s possible we were passed a symbol (this method is part of the public API). Next we check to see if we’ve already type cast this attribute, as we cache the results. The next line is not always obvious.

```ruby
column = @column_types_override[name] if @column_types_override
column ||= @column_types[name]
```

`@column_types_override` is sometimes given to us when the model in question was built as part of the result of a SQL query. If you’ve done something like

```ruby
Post
  .joins(:comments)
  .select('posts.*, COUNT(comments.*) AS comments_count')
  .group('comments.post_id')
```

then we sometimes have to do additional leg work to type cast the count to an integer. If you ran that code while using the MySQL adapter or PostgreSQL adapter (keep in mind that most MySQL users are using the MySQL2 adapter), then @column_types_override would look like: `{ 'comments_count' => an_object_that_type_casts_to_int }`. Continuing to the next line, `@column_types` will contain the column object that is crucial to this behavior, except for a few special cases (which we will have to cover another time).

The next block of code causes model.id to return the primary key, even if the primary key for the table is a column other than id.

```ruby
return @attributes.fetch(name) {
  if name == 'id' && self.class.primary_key != name
    read_attribute(self.class.primary_key)
  end
} unless column
```

Next we need to grab the raw, un-typecast version of the attribute, which came either from user input, or from the database (“user” in this case refers to you, the programmer using Rails). However, there’s an interesting fork in behavior here.

```ruby
value = @attributes.fetch(name) {
  return block_given? ? yield(name) : nil
}
```

The first question is whether or not a block was given. This is based on how read_attribute ended up being called. If you called it as post.title, no block would have been given. If you called it as post[:title], then a block would have been given to raise an exception. The reason that the title method would exist in this case, even if we don’t have a 'title' key in our attributes hash is: you performed a custom select statement. (This is an excellent example of how one feature can cause a surprising amount of complexity if not sufficiently isolated).

The conditional around caching attributes is actually bugged, and will always return true for most users, so we’ll ignore it. This leaves us with the line of importance:

```ruby
@attributes_cache[name] = column.type_cast(value)
```

column in this case, will be an instance of ActiveRecord::ConnectionAdapters::Column, or one of its adapter specific subclasses. The behavior in question lives on the type_cast method (source).

```ruby
# Casts value (which is a String) to an appropriate instance.
def type_cast(value)
  return nil if value.nil?
  return coder.load(value) if encoded?

  klass = self.class

  case type
  when :string, :text        then value
  when :integer              then klass.value_to_integer(value)
  when :float                then value.to_f
  when :decimal              then klass.value_to_decimal(value)
  when :datetime, :timestamp then klass.string_to_time(value)
  when :time                 then klass.string_to_dummy_time(value)
  when :date                 then klass.value_to_date(value)
  when :binary               then klass.binary_to_string(value)
  when :boolean              then klass.value_to_boolean(value)
  else value
  end
end
```

As any good method should, we start with an extremely misleading comment (the value will be anything you passed to the writer, not a string). Like many methods in Column, we also have a case statement based on type, and will call one of many helper methods based on which type it is. type will have been set back in the constructor, by using a regex from the sql_type source. sql_type will be the raw type string from the database schema, such as varchar(255).

At this point, the behavior is linear. All of the helper methods called exist in the class, and most are no more than a few lines long.

Also of note is the method type_cast_for_write, which gets called during the writer, before we store the attributes for type casting later. (Note: Anything that happens in this method will be applied to the _before_type_cast version of the attribute as well.)

If you’ve been cringing looking through the Column class, you’re justified. Luckily, it’s gotten much better. In preparation for adding a public API to hook into the type casting behavior in 4.2, this class has been heavily refactored to focus on polymorphism, rather than conditionals and regular expressions. In part 2, we’ll dig into some of the refactoring that’s been done, and the decisions behind it.
