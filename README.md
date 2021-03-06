# Squeel

Squeel is a rewrite of [MetaWhere](http://metautonomo.us/projects/metawhere).

## Getting started

In your Gemfile:

    gem "squeel"  # Last officially released gem
    # gem "squeel", :git => "git://github.com/ernie/squeel.git" # Track git repo

In an intitializer:

    Squeel.configure do |config|
      # To load hash extensions (to allow for AND (&), OR (|), and NOT (-) against
      # hashes of conditions)
      config.load_core_extensions :hash

      # To load symbol extensions (for a subset of the old MetaWhere functionality,
      # via ARel predicate methods on Symbols: :name.matches, etc)
      # config.load_core_extensions :symbol

      # To load both hash and symbol extensions
      # config.load_core_extensions :hash, :symbol
    end

## The Squeel Query DSL

Squeel enhances the normal ActiveRecord query methods by enabling them to accept
blocks. Inside a block, the Squeel query DSL can be used. Note the use of curly braces
in these examples instead of parentheses. `{}` denotes a Squeel DSL query.

Stubs and keypaths are the two primary building blocks used in a Squeel DSL query, so we'll
start by taking a look at them. Most of the other examples that follow will be based on
this "symbol-less" block syntax.

*An important gotcha, before we begin:* The Squeel DSL works its magic using `instance_eval`.
If you've been working with Ruby for a while, you'll know immediately that this means that
_inside_ a Squeel DSL block, `self` isn't the same thing that it is _outside_ the block.

This carries with it an important implication: <strong>Instance variables and instance methods
inside the block won't refer to your object's variables/methods.</strong>

Don't worry, Squeel's got you covered. Use one of the following methods to get access
to your object's methods and variables:

  1. Assign the variable locally before the DSL block, and access it as you would
     normally.
  2. Supply an arity to the DSL block, as in `Person.where{|dsl| dsl.name == @my_name}`
     Downside: You'll need to prefix stubs, keypaths, and functions (explained below)
     with the DSL object.
  3. Wrap the method or instance variable inside the block with `my{}`.
     `Person.where{name == my{some_method_to_return_a_name}}`

### Stubs

Stubs are, for most intents and purposes, just like Symbols in a normal call to
`Relation#where` (note the need for doubling up on the curly braces here, the first ones
start the block, the second are the hash braces):

    Person.where{{name => 'Ernie'}}
    => SELECT "people".* FROM "people"  WHERE "people"."name" = 'Ernie'

You normally wouldn't bother using the DSL in this case, as a simple hash would
suffice. However, stubs serve as a building block for keypaths, and keypaths are
very handy.

### KeyPaths

A Squeel keypath is essentially a more concise and readable alternative to a
deeply nested hash. For instance, in standard ActiveRecord, you might join several
associations like this to perform a query:

    Person.joins(:articles => {:comments => :person})
    => SELECT "people".* FROM "people"
         INNER JOIN "articles" ON "articles"."person_id" = "people"."id"
         INNER JOIN "comments" ON "comments"."article_id" = "articles"."id"
         INNER JOIN "people" "people_comments" ON "people_comments"."id" = "comments"."person_id"

With a keypath, this would look like:

    Person.joins{articles.comments.person}

A keypath can exist in the context of a hash, and is normally interpreted relative to
the current level of nesting. It can be forced into an "absolute" path by anchoring it with
a ~, like:

    ~articles.comments.person

This isn't quite so useful in the typical hash context, but can be very useful when it comes
to interpreting functions and the like. We'll cover those later.

### Predicates

All of the ARel "predication" methods can be accessed inside the Squeel DSL, via
their method name, an alias, or an an operator, to create ARel predicates, which are
used in `WHERE` or `HAVING` clauses.

<table>
  <tr>
    <th>SQL</th>
    <th>Predication</th>
    <th>Operator</th>
    <th>Alias</th>
  </tr>
  <tr>
    <td>=</td>
    <td>eq</td>
    <td>==</td>
    <td></td>
  </tr>
  <tr>
    <td>!=</td>
    <td>not_eq</td>
    <td>!= (1.9 only), ^ (1.8)</td>
    <td></td>
  </tr>
  <tr>
    <td>LIKE</td>
    <td>matches</td>
    <td>=~</td>
    <td>like</td>
  </tr>
  <tr>
    <td>NOT LIKE</td>
    <td>does_not_match</td>
    <td>!~ (1.9 only)</td>
    <td>not_like</td>
  </tr>
  <tr>
    <td>&lt;</td>
    <td>lt</td>
    <td>&lt;</td>
    <td></td>
  </tr>
  <tr>
    <td>&lt;=</td>
    <td>lteq</td>
    <td>&lt;=</td>
    <td>lte</td>
  </tr>
  <tr>
    <td>></td>
    <td>gt</td>
    <td>></td>
    <td></td>
  </tr>
  <tr>
    <td>>=</td>
    <td>gteq</td>
    <td>>=</td>
    <td>gte</td>
  </tr>
  <tr>
    <td>IN</td>
    <td>in</td>
    <td>>></td>
    <td></td>
  </tr>
  <tr>
    <td>NOT IN</td>
    <td>not_in</td>
    <td>&lt;&lt;</td>
    <td></td>
  </tr>
</table>

Let's say we want to generate this simple query:

    SELECT "people".* FROM people WHERE "people"."name" = 'Joe Blow'

All of the following will generate the above SQL:

    Person.where(:name => 'Joe Blow')
    Person.where{{name => 'Joe Blow'}}
    Person.where{{name.eq => 'Joe Blow'}}
    Person.where{name.eq 'Joe Blow'}
    Person.where{name == 'Joe Blow'}
    
Not a very exciting example since equality is handled just fine via the
first example in standard ActiveRecord. But consider the following query:

    SELECT "people".* FROM people
    WHERE ("people"."name" LIKE 'Ernie%' AND "people"."salary" < 50000)
      OR  ("people"."name" LIKE 'Joe%' AND "people"."salary" > 100000)
      
To do this with standard ActiveRecord, we'd do something like:

    Person.where(
      '(name LIKE ? AND salary < ?) OR (name LIKE ? AND salary > ?)',
      'Ernie%', 50000, 'Joe%', 100000
    )
    
With Squeel:

    Person.where{(name =~ 'Ernie%') & (salary < 50000) | (name =~ 'Joe%') & (salary > 100000)}
    
Here, we're using `&` and `|` to generate `AND` and `OR`, respectively.

There are two obvious but important differences between these two code samples, and
both of them have to do with *context*.

1. To read code with SQL interpolation, the structure of the SQL query must
   first be considered, then we must cross-reference the values to be substituted
   with their placeholders. This carries with it a small but perceptible (and
   annoying!) context shift during which we stop thinking about the comparison being
   performed, and instead play "count the arguments", or, in the case of 
   named/hash interpolations, "find the word". The Squeel syntax places
   both sides of each comparison in proximity to one another, allowing us to
   focus on what our code is doing.

2. In the first example, we're starting off with Ruby, switching context to SQL,
   and then back to Ruby, and while we spend time in SQL-land, we're stuck with
   SQL syntax, whether or not it's the best way to express what we're trying to do.
   With Squeel, we're writing Ruby from start to finish. And with Ruby syntax comes
   flexibility to express the query in the way we see fit.

### Predicate aliases

That last bit is important. We can mix and match predicate methods with operators
and take advantage of Ruby's operator precedence or parenthetical grouping to make
our intentions more clear, on the first read-through. And if we don't like the
way that the existing predications read, we can create our own aliases in a Squeel
configure block:

    Squeel.configure do |config|
      config.alias_predicate :is_less_than, :lt
    end
    
    Person.where{salary.is_less_than 50000}.to_sql
    # => SELECT "people".* FROM "people"  WHERE "people"."salary" < 50000

And while we're on the topic of helping you make your code more expressive...

### Compound conditions

Let's say you want to check if a Person has a name like one of several possibilities.

    names = ['Ernie%', 'Joe%', 'Mary%']
    Person.where('name LIKE ? OR name LIKE ? OR name LIKE ?', *names)

But you're smart, and you know that you might want to check more or less than
3 names, so you make your query flexible:

    Person.where((['name LIKE ?'] * names.size).join(' OR '), *names)

Yeah... that's readable, all right. How about:

    Person.where{name.like_any names}
    # => SELECT "people".* FROM "people"  
         WHERE (("people"."name" LIKE 'Ernie%' OR "people"."name" LIKE 'Joe%' OR "people"."name" LIKE 'Mary%'))
    
I'm not sure about you, but I much prefer the latter. In short, you can add `_any` or
`_all` to any predicate method, and it would do what you expect, when given an array of
possibilities to compare against.

### Subqueries

You can supply an `ActiveRecord::Relation` as a value for a predicate in order to use
a subquery. So, for example:

    awesome_people = Person.where{awesome == true}
    Article.where{author_id.in(awesome_people.select{id})}
    # => SELECT "articles".* FROM "articles"  
         WHERE "articles"."author_id" IN (SELECT "people"."id" FROM "people"  WHERE "people"."awesome" = 't')

### Joins

Squeel adds a couple of enhancements to joins. First, keypaths can be used as shorthand for
nested association joins. Second, you can specify join types (inner and outer), and a class
in the case of a polymorphic belongs_to relationship.

    Person.joins{articles.outer}
    => SELECT "people".* FROM "people"
       LEFT OUTER JOIN "articles" ON "articles"."person_id" = "people"."id"
    Note.joins{notable(Person).outer}
    => SELECT "notes".* FROM "notes"
       LEFT OUTER JOIN "people"
         ON "people"."id" = "notes"."notable_id"
         AND "notes"."notable_type" = 'Person'

These can also be used inside keypaths:

    Note.joins{notable(Person).articles}
    => SELECT "notes".* FROM "notes"
       INNER JOIN "people" ON "people"."id" = "notes"."notable_id"
         AND "notes"."notable_type" = 'Person'
       INNER JOIN "articles" ON "articles"."person_id" = "people"."id"
       
You can refer to these associations when constructing other parts of your query, and
they'll be automatically mapped to the proper table or table alias This is most noticeable
when using self-referential associations:

    Person.joins{children.parent.children}.
           where{
             (children.name.like 'Ernie%') |
             (children.parent.name.like 'Ernie%') |
             (children.parent.children.name.like 'Ernie%')
           }
    => SELECT "people".* FROM "people" 
       INNER JOIN "people" "children_people" ON "children_people"."parent_id" = "people"."id" 
       INNER JOIN "people" "parents_people" ON "parents_people"."id" = "children_people"."parent_id" 
       INNER JOIN "people" "children_people_2" ON "children_people_2"."parent_id" = "parents_people"."id" 
       WHERE ((("children_people"."name" LIKE 'Ernie%' 
             OR "parents_people"."name" LIKE 'Ernie%') 
             OR "children_people_2"."name" LIKE 'Ernie%'))

Keypaths were used here for clarity, but nested hashes would work just as well.

### Functions

You can call SQL functions just like you would call a method in Ruby...

    Person.select{coalesce(name, '<no name given>')}
    => SELECT coalesce("people"."name", '<no name given>') FROM "people"

...and you can easily give it an alias:

    person = Person.select{
      coalesce(name, '<no name given>').as(name_with_default)
    }.first
    person.name_with_default # name or <no name given>, depending on data

When you use a stub, symbol, or keypath inside a function call, it'll be interpreted relative to
its place inside any nested associations:

    Person.joins{articles}.group{articles.title}.having{{articles => {max(id) => id}}}
    => SELECT "people".* FROM "people" 
       INNER JOIN "articles" ON "articles"."person_id" = "people"."id" 
       GROUP BY "articles"."title" 
       HAVING max("articles"."id") = "articles"."id"
       
If you want to use an attribute from a different branch of the hierarchy, use an absolute
keypath (~) as done here:

    Person.joins{articles}.group{articles.title}.having{{articles => {max(~id) => id}}}
    => SELECT "people".* FROM "people" 
       INNER JOIN "articles" ON "articles"."person_id" = "people"."id" 
       GROUP BY "articles"."title" 
       HAVING max("people"."id") = "articles"."id"

### SQL Operators

You can use the standard mathematical operators (`+`, `-`, `*`, `/`) inside the Squeel DSL to
specify operators in the resulting SQL, or the `op` method to specify another
custom operator, such as the standard SQL concatenation operator, `||`:

    p = Person.select{name.op('||', '-diddly').as(flanderized_name)}.first
    p.flanderized_name
    => "Aric Smith-diddly" 

As you can see, just like functions, these operations can be given aliases.

## Legacy compatibility

While the Squeel DSL is the preferred way to access advanced query functionality, you can
still enable methods on symbols to access ARel predications in a similar manner to MetaWhere:

    Squeel.configure do |config|
      config.load_core_extensions :symbol
    end
    
    Person.joins(:articles => :comments).
           where(:articles => {:comments => {:body.matches => 'Hello!'}})
    SELECT "people".* FROM "people" 
    INNER JOIN "articles" ON "articles"."person_id" = "people"."id" 
    INNER JOIN "comments" ON "comments"."article_id" = "articles"."id" 
    WHERE "comments"."body" LIKE 'Hello!'

This should help to smooth over the transition to the new DSL.

## Contributions

If you'd like to support the continued development of Squeel, please consider
[making a donation](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=N7QP5N3UB76ME).

To support the project in other ways:

* Use Squeel in your apps, and let me know if you encounter anything that's broken or missing.
  A failing spec is awesome. A pull request is even better!
* Spread the word on Twitter, Facebook, and elsewhere if Squeel's been useful to you. The more
  people who are using the project, the quicker we can find and fix bugs!

## Copyright

Copyright &copy; 2011 [Ernie Miller](http://twitter.com/erniemiller)