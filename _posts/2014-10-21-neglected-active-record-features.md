---
layout:   post
title:    "Neglected ActiveRecord Features"
date:     2014-10-21
summary:  Some ActiveRecord features not every Rails developers knows, but could come in handy.
tags:     rails activerecord neglected
---

As Rails developers, we're constantly using ActiveRecord, but are mostly using the same subset of methods and features it provides. Some of its lesser known and lesser used features could help you write cleaner code in the future and delight your co-workers. So let's dive right in!

#### Pluck

{% highlight ruby %}

> user_ids = User.limit(5).pluck(:id)
  (0.5ms) SELECT "users"."id" FROM "users" ORDER BY "users"."name" ASC LIMIT 5
=> [856, 857, 858, 850, 852]

{% endhighlight %}

Instead of calling `map` on ActiveRecord objects, use `pluck` if you only need a list of attributes. It avoids building ActiveRecord model objects and is a lot more efficient for larger collections.

#### Array Arguments

{% highlight ruby %}

> Post.where(:user_id => user_ids).count
  (5.2ms) SELECT COUNT(* ) FROM "posts" WHERE "posts"."user_id" IN (856, 857, 858, 850, 852)
=> 286

{% endhighlight %}

You can provide an array argument for the `where` clause, which is translated to an SQL `IN`.

#### Where and Not

{% highlight ruby %}

> Post.count
  (0.6ms)  SELECT COUNT(* ) FROM "posts"
=> 1120
> Post.where(:category => 'rails').count
  (0.8ms)  SELECT COUNT(* ) FROM "posts"  WHERE "posts"."category" = 'rails'
=> 295
> Post.where.not(:category => 'rails').count
  (0.9ms)  SELECT COUNT(* ) FROM "posts"  WHERE ("posts"."category" != 'rails')
=> 825

{% endhighlight %}

Using `where.not` will test for inequality, where you might have used `where('category != ?', 'rails')` in the past. This was introduced with Rails 4.0.

#### Ranges

{% highlight ruby %}

> date_start = Date.new(2013, 1, 1)
=> Tue, 01 Jan 2013
> date_end = Date.new(2013, 2, 1)
=> Fri, 01 Feb 2013
> puts Post.where(:published_on => date_start..date_end).to_sql
SELECT "posts".* FROM "posts"  WHERE ("posts"."published_on" BETWEEN '2013-01-01' AND '2013-02-01')

{% endhighlight %}

Where clauses also support ranges. In the example above we see a ruby date range that gets elegantly translated
to SQL `BETWEEN` syntax.

#### Order

{% highlight ruby %}

> puts Post.unscoped.order(:published_on => :asc).to_sql
SELECT "posts".* FROM "posts"   ORDER BY "posts"."published_on" ASC
=> nil
> puts Post.unscoped.order(:published_on => :desc).to_sql
SELECT "posts".* FROM "posts"   ORDER BY "posts"."published_on" DESC
=> nil

{% endhighlight %}

Instead of the using the `order` argument with a string as `order("published_on DESC")`, you can simply pass a hash.

#### Discarding From The Chain

{% highlight ruby %}

> puts Post.unscoped.order(:published_on => :desc).except(:order).to_sql
SELECT "posts".* FROM "posts"

{% endhighlight %}

With the `expect` syntax you can get rid of clauses to the ActiveRecord relation that were added before. This could
come in handy if you're given a base scope that is valid for all cases except yours and avoids throwing away any other modifications to the relation.

#### The Null Relation

{% highlight ruby %}

> Post.none.where(:category => "rails")
=> []

{% endhighlight %}

This feature might look like non-sense at first. The `none` clause makes sure that whatever clause is added to the relation, the result will always be empty. Again, this might make sense if you are provided with a relation and have to return one again. Say now we have a case where the current query is not valid due to permission constraints. Adding the `none` clause makes sure that whatever relation modification follows, the final query will not hit the database and will be empty.

#### Readonly

{% highlight ruby %}

post = Post.readonly.first
post.title = "Don't change me!"
post.save!

{% endhighlight %}

The `readonly` clause makes sure that a relation cannot be modified. This might make sense if you pass an object to any code that is not yours (e.g. a library) and want to make sure that the object is not modified.

#### Summary

So that's it. I hope you can use some of these tricks to not only make your code faster and more elegant, but also more readable.
