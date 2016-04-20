# ActiveRecord Associations Review

ActiveRecord associations are an iconic Rails feature. They allow developers to work with complex networks of related models without having to write a single line of SQL--as long as all of the names line up!

## Objectives

After this lesson, you will be able to...

1. Create the correct foreign keys for associations.
2. Use the correct association macros to apply an ActiveRecord association.
3. List the methods that are added by the association macros.
4. Use `build` and `create` correctly to instantiate associated data.
   `post.build_author` and `author.posts.build`
5. Add associated data to a collection association (push objects onto a `has_many`
   association, `@category.posts << post`).
6. Identify the SQL generated by various association methods.
7. Construct a join model and table.
8. Use a `has_many :through` with a join model.
9. Manipulate associated data from a `has_many :through` including creating the
   association from the join model directly `PostTag.new(:post => post, :tag =>
   tag)`

# Foreign Keys

It all starts in the database. **Foreign keys** are columns that refer to the primary key of another table. Conventionally, foreign keys in ActiveRecord are comprised of the name of the model you're referencing, and `_id`. So for example if the foreign key was for a `posts` table it would be `post_id`.

Like any other column, foreign keys are accessible through instance methods of the same name. For example, a migration that looks like this:

```ruby
class AddAuthorIdToPosts < ActiveRecord::Migration
  def change
    change_table :posts do |t|
      t.integer :author_id
    end
  end
end
```

Would mean you could find a post's author with the following ActiveRecord query:

```ruby
Author.find(@post.author_id)
```

Which is equivalent to the SQL:

```sql
SELECT * FROM authors WHERE id = #{@post.author_id}
```

And you could lookup an author's posts like this:

```ruby
Post.where("author_id = ?", @author.id)
```

Which is equivalent to the SQL:

```sql
SELECT * FROM posts WHERE author_id = #{@author.id}
```

This is all great, but Rails is always looking for ways to save us keystrokes.

# Many-to-One Relationships

By using ActiveRecord's macro-style association class methods, we can add some convenient instance methods to our models.

The most common relationship is **many-to-one**, and it is declared in ActiveRecord with `belongs_to` and `has_many`.

## `belongs_to`

Each `Post` is associated with **one** `Author`.

```ruby
class Post < ActiveRecord::Base
  belongs_to :author
end
```

We now have access to some new instance methods, like `author`. This will return the actual `author` object that is attached to that `@post`.

```ruby
@post.author_id = 5
@post.author #=> #<Author id=5>
```

## `has_many`

In the opposite direction, each `Author` might have zero, one, or many `Post`s. We haven't changed the schema of the `authors` table at all; ActiveRecord is just going to use `posts.author_id` to do all of the lookups.

```ruby
class Author < ActiveRecord::Base
  has_many :posts
end
```

Now we can look up an author's posts just as easily:

```ruby
@author.posts #=> [#<Post id=3>, #<Post id=8>]
```

Remember, ActiveRecord uses its [Inflector][api_inflector] to switch between the singular and plural forms of your models.

Name | Data
---- | -----
Model | `Author`
Table | `authors`
Foreign Key | `author_id`
`belongs_to` | `:author`
`has_many` | `:authors`

Like many other ActiveRecord class methods, the symbol you pass determines the name of the instance method that will be defined. So `belongs_to :author` will give you `@post.author`, and `has_many :posts` will give you `@author.posts`.

# Convenience Builders

## Building a new item in a collection

If you want to add a new post for an author, you might start this way:

```ruby
new_post = Post.new(author_id: @author.id, title: "Web Development for Cats")
new_post.save
```

But the association macros save the day again, allowing this instead:

```ruby
new_post = @author.posts.build(title: "Web Development for Cats")
new_post.save
```

This will return a new `post` object with the `author_id` already set for you! We use this one as much as possible because it's just easier. `build` works just like `new`. So the instance that is returned isn't quite saved to the database just yet. You'll need to `#save` the instance when you want it to be persisted to the database.

## Setting a singular association

It's a little bit less intuitive for singular associations. Remember a post `belongs_to` an author. The verbose way of doing this would be like so:

```ruby
@post.author = Author.new(name: "Leeroy Jenkins") 
```
In the previous section, `@author.posts` always exists, even if it's an empty array. Here, `@post.author` is `nil` until the author is defined, so ActiveRecord can't give us something like `@post.author.build`. Instead, it prepends the attribute with `build_` and `create_`. The `create_` will persist to the database for you.

```ruby
new_author = @post.build_author(name: "Leeroy Jenkins")
```

Remember! you need to save your new `author` with `#save`.

These methods are also documented in the [Rails Associations Guide][guides_associations].

## Collection Convenience

If you add an existing object to a collection association, ActiveRecord will conveniently take care of setting the foreign key for you:

```ruby
@author = Author.find_by(name: "Leeroy Jenkins")
@author.posts
#=> []
@post = Post.new(title: "Web Development for Cats")
@post.author
#=> nil
@author.posts << @post
@post.author
#=> #<Author @name="Leeroy Jenkins">
```

# One-to-One Relationships

Profiles can get pretty complex, so in large applications, it can be a good idea to give them their own model. In this case:

- Every author would have one, and only one, profile
- Every profile would have one, and only one, author.

`belongs_to` makes another appearance in this relationship, but instead of
`has_many`, the other model is declared with `has_one`.

If you're not sure which model should be declared with which macro, it's usually a safe bet to put `belongs_to` on whichever model has the foreign key column in its database table.


# Many-to-Many Relationships and Join Tables

Each author has many posts, each post has one author.

The universe is in balance. We're programmers, so this really disturbs us. Let's shake things up and think about tags.

- One-to-One doesn't work because a post can have multiple tags.
- Many-to-One doesn't work because a tag can appear on multiple posts.

Because there is no "owner" model in this relationship, there's also no right place to put the foreign key column.

Enter the join table:

`tag_id` | `post_id`
-------- | ---------
1        | 1
2        | 1
1        | 5

This join table depicts two tags (1 and 2) and two posts (1 and 5). Post 1 has both tags, while Post 5 has only one.

Mercifully, ActiveRecord has a migration method for doing exactly this.

```ruby
create_join_table :posts, :tags
```

This will create a table called `posts_tags`.

# `has_many :through`

To work with the join table, both our `Post` and our `Tag` model will `have_many` `post_tags`. But! We still need to associate `Post` and `Tag` themselves. I would like to do something like `@my_post.tags` right? That's where `has_many :through` comes in.

To do this requires a bit of focus. But you can do it! First of all, our `Post` and our `Tag` model will `has_many` `:posts_tags`.

```ruby
class Post
  has_many :posts_tags
end

class PostsTag
  belongs_to :post
  belongs_to :tag
end

class Tag
  has_many :posts_tags
end
```

So now we can run code like `@post.posts_tags` to get all the join entries. This is kinda sorta what we want. What we really want is to be able to do `@post.tags`. So! We need one more `has_many` relationship to make the link between tags and posts. This is the `has_many :through`. In English, my `Post` has many `Tags` through the `posts_tags` model. So let's write that:

```ruby
class Post
  has_many :posts_tags
  has_many :tags, through: :posts_tags
end

class PostsTag
  belongs_to :post
  belongs_to :tag
end

class Tag
  has_many :posts_tags
  has_many :posts, through: :posts_tags
end
```

Now! You can do `@post.tags` and everything should work :)

# Summary

For every relationship, there is a foreign key somewhere. Foreign keys correspond to the `belongs_to` macro on the model.

One-to-one and many-to-one relationships only require a single foreign key, on the "subordinate" or "owned" model. The other model declares its relationship with `has_one` or `has_many`, respectively.

Many-to-many relationships require a join table, with a foreign key for both models. These are joined using `has_many :through`.


You can see the entire [list of class methods][api_associations_class_methods] in the Rails API docs.

[guides_associations]: http://guides.rubyonrails.org/association_basics.html
[guides_has_many_through]: http://guides.rubyonrails.org/association_basics.html#the-has-many-through-association
[api_associations_class_methods]: http://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html
[api_inflector]: http://api.rubyonrails.org/classes/ActiveSupport/Inflector.html

<p data-visibility='hidden'>View <a href='https://learn.co/lessons/activerecord-associations-review'>ActiveRecord Associations Review</a> on Learn.co and start learning to code for free.</p>
