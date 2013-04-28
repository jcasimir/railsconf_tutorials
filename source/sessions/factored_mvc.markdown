---
layout: page
title: Properly Factored MVC
sidebar: true
---

## Intro

We will be using a long-lived open source rails application named
[tracks](http://codeclimate.com/github/jumpstartlab/tracks) for
this workshop.

It is a todo list application inspired by David Allen's [Getting Things Done](http://www.amazon.com/Getting-Things-Done-Stress-Free-Productivity/dp/0142000280).

The first commit on the application was made in 2006. It currently has
well over 200 forks, and is still under active development.

## Setup

### Prerequisites

* ruby 1.9.3
* bundler
* firefox (cucumber features)
  I have a .dmg that can be passed around if necessary


### Code Base

0. Fork the repository at https://github.com/jumpstartLab/tracks
1. Clone your fork: `git clone git://github.com/<you>/tracks.git`
2. Add your repository to [Code Climate](https://codeclimate.com/github/signup).
3. `cp config/site.yml.tmpl config/site.yml`
4. `cp config/database.yml.tmpl config/database.yml`
5. `bundle install`
6. `bundle exec rake db:create db:migrate db:test:prepare`
7. `bundle exec rake wip`

**NOTE**: `wip` stands for _work in progress_.

If the tests run correctly when you run `bundle exec rake wip`, then
you're all set.

## I0: Exploration

Take a look at the [Code Climate report](http://codeclimate.com/github/JumpstartLab/tracks) for this code base.

* Which parts of the code base have the worst ratings?
* What seems the scariest to you (you don't need to be able to
  articulate why it seems scary)?
* What does code climate tell us about the code quality in the views?

We will be working on a small portion of the `StatsController`.

* What does Code Climate say is the `complexity` metric for this file?
* How about the `duplication` metric?
* What do these metrics actually mean?
* How long is the `index` action in the `StatsController`?
* What methods are being called?
* Where are they defined?
* How long are they?
* How many instance variables get assigned?
* Do all the assigned instance variables get used in the view and
  partials that get rendered?

## I1: Test Coverage

Run the `bundle exec rake wip` task and then open up `coverage/index.html`
 in your browser.

On a mac you can do this by saying `open coverage/index.html` on the
command line.

This will not represent the full coverage of the entire project, only
the coverage when running the handful of tests in the `wip` task.

Find the `StatsController` in this list.

* What is the code coverage for this file?
* How good do you think the coverage is for the `get_stats_tags` method?

Now open up the `app/views/stats/index.html.erb` file and delete the line
that looks like this:

```erb
<%= render :partial => 'tags' -%>
```

Run `bundle exec rake wip` again.

* How many tests fail?

Run the command `git checkout .` to put the deleted code back.

### A lockdown test

The tag cloud appears to have 100% code coverage, but it turns out that
there are no assertions regarding this part of the code, it just runs
without raising any exceptions.

We need a test that will fail if we change anything in the `get_stats_tags` method or the corresponding `tags` partial.

Run the following command:

```bash
git checkout -b iteration1 cloud.i1
```

This creates a new branch named `iteration1` based off of the git tag
named `cloud.i1`.

In the tag `cloud.i1` we've added a new test file `test/functional/lockdown_test.rb`.
An additional rake task in `lib/tasks/wip.rake` is set up to run this test.

```bash
bundle exec rake test:lockdown
```

Go ahead and run the test, which will fail.

If you look inside the `.lockdown/` directory which has been created at
the root of the tracks project, you will see two files:

* `.lockdown/received.html`
* `.lockdown/approved.html`

The lockdown test is an `approval` test, meaning that whatever you
previously approved is considered the correct result.

The `approved.html` file is empty, because this is the first time the
test has been run on this system. The `received.html` file contains the
entire output of the `StatsController#index` action.


For the purposes of this refactoring, we're going to assume that the
`StatsController#index` action is rendering exactly what it should.

Approve the test by copying the received file over the approved file:

```bash
cp .lockdown/received.html .lockdown/approved.html
```

If you run `bundle exec rake test:lockdown` again, the test should pass.

Open up the `app/views/stats/index.html.erb` file and delete the line that
looks like this again:

```erb
<%= render :partial => 'tags' -%>
```

Run `bundle exec rake test:lockdown` again. It should fail.

Reset the code with `git checkout .`. We're ready to refactor.

Commit your changes.

## I2: Cordonning Off Some Code

Check out a new branch based on the current state of your code:

```bash
git checkout -b iteration2
```

We're going to be refactoring the `get_stats_tags` method in the
`StatsController`.

* How long is the method?
* How many instance variables does it assign?
* How many SQL queries does it execute?
* Where does it get called from? (HINT: `git grep get_stats_tags`)

This logic belongs in the model layer. We're going to create a ruby class that
doesn't inherit from `ActiveRecord::Base`.

Create a directory inside of `app/models/` named `stats`. Then, create a file
`app/models/stats/tag_cloud.rb`

```ruby
class TagCloud
  def compute
  end
end
```

Now _copy_ (do NOT _cut_) the contents of the `get_stats_tags` method from the
controller into the compute method of the new `TagCloud` class.

Go back to the controller, and add this line of code to the top of the
`get_stats_tags` method:

```ruby
TagCloud.new.compute
```

Notice that we aren't using the line of code yet. We're just calling it. All
the old code is still in the `get_stats_tags` method.

Run the `bundle exec rake test:lockdown` task again.

We get the following error:

```text
1) Error:
  test_page_does_not_change_while_refactoring(LockdownTest):
    NameError: undefined local variable or method `current_user' for #<TagCloud:0x007f9dfd70ea20>
```

We can fix that by giving the tag cloud object a `current_user`:

```ruby
class TagCloud

  attr_reader :current_user
  def initialize(current_user)
    @current_user = current_user
  end

  def compute
    # ...
  end
end
```

And then pass the `current_user` to the tag cloud in the `StatsController#get_stats_tags` method:

```ruby
TagCloud.new(current_user).compute
```

Rerun the `test:lockdown` rake task. The test should be passing.

Commit your changes.

## I3: Using the TagCloud

If you've gotten confused and want a clean slate, go ahead and checkout a new branch based on the `cloud.i2` tag.

```bash
git checkout -b iteration3 cloud.i2
```

Otherwise just create a new branch based on the current state of your code:

```bash
git checkout -b iteration3
```

First, let's assign the `TagCloud` object that we're creating in the `StatsController#get_stats_tags` method:

```ruby
def get_stats_tags
  cloud = TagCloud.new(current_user)
  cloud.compute
  # ...
end
```

Then find the first instance variable assignment in `get_stats_tags` method.

The first one I find is `@tags_for_cloud`.

We want to expose this instance variable in the TagCloud object. Add an `attr_reader` for it.

```ruby
class TagCloud

   attr_reader :current_user, :tags_for_cloud

   # ...

end
```

Then, back in the controller, find the place where `@tags_for_cloud` is being assigned, and add a new line underneath it:

```ruby
@tags_for_cloud = Tag.find_by_sql(query).sort_by { |tag| tag.name.downcase }
@tags_for_cloud = cloud.tags_for_cloud
```

Run the `lockdown` test. It should be passing.

Now you can delete the original `@tags_for_cloud` assignment, along with the big SQL `query` string, because it is only used in that line of code.

Run the `lockdown` test again to make sure you didn't delete too much.

The next assignment is `@tags_min`, which is first assigned and then changed:

```ruby
max, @tags_min = 0, 0
@tags_for_cloud.each { |t|
  max = [t.count.to_i, max].max
  @tags_min = [t.count.to_i, @tags_min].min
}
```

Create an `attr_reader` in the `TagCloud` model for `:tags_min`, and then go back to the controller and add a new assignment below the section that deals with `@tags_min`:

```ruby
@tags_min = cloud.tags_min
```

Run the `lockdown` test again. It should still be passing.

Delete the code that we've just replaced and re-run the `lockdown` test to make sure that you didn't delete too much.

Ouch! That failed. We're missing a local variable named `max`. Let's put back the code we just deleted. The `@tags_divisor` assignment needs `max`.

Let's go ahead and expose `:tags_divisor` in the `TagCloud` object as well, and assign it in the controller:

```ruby
@tags_divisor = ((max - @tags_min) / levels) + 1
@tags_divisor = cloud.tags_divisor
```

Run the lockdown test. It should be passing.

Now we can delete the old code for `@tags_min` and `@tags_divisor`. Do that, and then run the `lockdown` test again.

The `get_stats_tags` method should now look something like this:

```ruby
def get_stats_tags
  cloud = TagCloud.new(current_user)
  cloud.compute
  # tag cloud code inspired by this article
  #  http://www.juixe.com/techknow/index.php/2006/07/15/acts-as-taggable-tag-cloud/

  levels=10
  # TODO: parameterize limit

  @tags_for_cloud = cloud.tags_for_cloud
  @tags_min = cloud.tags_min
  @tags_divisor = cloud.tags_divisor

  # A lot more code here...
end
```

The next instance variable that gets assigned is `@tags_for_cloud_90_days`. Expose this in the `TagCloud` using an attr_reader, and then put a new declaration below it using the exposed value:

```ruby
@tags_for_cloud_90days = Tag.find_by_sql(
  [query, current_user.id, @cut_off_3months, @cut_off_3months]
).sort_by { |tag| tag.name.downcase }
@tags_for_cloud_90days = cloud.tags_for_cloud_90_days
```

The tests are failing. What happened?

Let's look at the diff:

```text
diff .lockdown/approved.html .lockdown/received.html
```

It looks like the whole tag cloud disappeared. What the heck?

If you take a good look at the query for `@tags_for_cloud_90_days` it references an instance variable named `@cut_off_3months`. Where is that instance variable defined?

It looks like it comes from a helper method in the controller called `init`. Let's pass it to the `TagCloud` object when we initialize it:

```ruby
cloud = TagCloud.new(current_user, @cut_off_3months)
```

We need to make sure that we have support for this new variable in the `TagCloud` as well:

```ruby
class TagCloud

  attr_reader :current_user, :tags_for_cloud, :tags_min, :tags_divisor,
    :tags_for_cloud_90days
  def initialize(current_user, cut_off_3months)
    @current_user = current_user
    @cut_off_3months = cut_off_3months
  end

  def compute
    # ...
  end
end
```

Run the `lockdown` test again. It should be passing.

Now we can delete that second big sql `query` string, along with the old `@tags_for_cloud_90days` assignment.

The test still passes.

The next instance variable that is being assigned is `@tags_min_90days`.

Expose the variable in the `TagCloud`, and add the new assignment below the old one in the controller.

Run the `lockdown` test. It should be passing.

We won't try to delete this, because it looks eerily familiar. The `@tags_divisor_90days` needs the `max_90days` local variable that is in there.

Expose the `@tags_divisor_90days` and add the new assignment below the old one.

This section of the controller should now look like this:

```ruby
max_90days, @tags_min_90days = 0, 0
@tags_for_cloud_90days.each { |t|
  max_90days = [t.count.to_i, max_90days].max
  @tags_min_90days = [t.count.to_i, @tags_min_90days].min
}
@tags_min_90days = cloud.tags_min_90days

@tags_divisor_90days = ((max_90days - @tags_min_90days) / levels) + 1
@tags_divisor_90days = cloud.tags_divisor_90days
```

The `lockdown` test should be passing.

Go ahead and delete the old code and rerun the `lockdown` test.

The controller method should look like this:

```ruby
def get_stats_tags
  cloud = TagCloud.new(current_user, @cut_off_3months)
  cloud.compute
  # tag cloud code inspired by this article
  #  http://www.juixe.com/techknow/index.php/2006/07/15/acts-as-taggable-tag-cloud/

  levels=10
  # TODO: parameterize limit

  @tags_for_cloud = cloud.tags_for_cloud
  @tags_min = cloud.tags_min
  @tags_divisor = cloud.tags_divisor

  @tags_for_cloud_90days = cloud.tags_for_cloud_90days
  @tags_min_90days = cloud.tags_min_90days
  @tags_divisor_90days = cloud.tags_divisor_90days
end
```

We can delete the comments, as well as the local variable named `levels`.
It isn't used anymore. The controller method now looks like this:

```ruby
def get_stats_tags
  cloud = TagCloud.new(current_user, @cut_off_3months)
  cloud.compute

  @tags_for_cloud = cloud.tags_for_cloud
  @tags_min = cloud.tags_min
  @tags_divisor = cloud.tags_divisor

  @tags_for_cloud_90days = cloud.tags_for_cloud_90days
  @tags_min_90days = cloud.tags_min_90days
  @tags_divisor_90days = cloud.tags_divisor_90days
end
```

The `lockdown` test should be passing.

Commit your changes.

## I4: Cleaning up the TagCloud

If you've gotten confused and want a clean slate, go ahead and checkout a new branch based on the `cloud.i3` tag.

```bash
git checkout -b iteration4 cloud.i3
```

Otherwise just create a new branch based on the current state of your code:

```bash
git checkout -b iteration4
```

### Current User is not Current

In `app/models/tag_cloud.rb` we're referring to a `current_user`, but users in
the model aren't really `current`, that's a controller/view level bit of
logic.

Rename `current_user` to `user`, and run the `lockdown` test to make sure you
did it correctly.

### Comments

Take a look at the comment that goes like this:

```ruby
# tag cloud code inspired by this article
# http://www.juixe.com/techknow/index.php/2006/07/15/acts-as-taggable-tag-cloud/
```

That seems like it is relevant to the whole file, not just the compute method.
Move it up to the top of the file.

The `TODO` comment is still relevant, but let's make it a comment for the
whole `compute` method.

The remaining comments are just trying to explain the code. Let's go ahead and
get rid of them.

### Redundant Variable Names

The variable names `tags_for_cloud` and `tags_for_cloud_90days` contain some
redundancy in them now, because these are being called on an object that *is*
a tag cloud. Let's get rid of the `for_cloud` bit and just call them `tags`
and `tags_90days`.

Once you've made the change in the `TagCloud` class, you'll need to update the
controller as well.

Be sure to only update the calls to `cloud`, not the instance variables that
you're assigning, because the views are still using the old names.

```ruby
@tags_for_cloud = cloud.tags
@tags_for_cloud_90days = cloud.tags_90days
```

I think the `tags_min` and `tags_divisor` variables also have some redundancy
in them. It's all about tags. Let's call them `min`, `divisor`, `min_90days`,
and `divisor_90days`.

Remember to update the controller as well.

Run the `lockdown` test, and it should still be passing.

## Overspecified Variable Name

The `@cut_off_3months` variable name has too much information in it. It will
work no matter what cut off we operate with. Let's rename it to be simply
`cut_off`.

Run the `lockdown` test, and commit your changes.

## I5: Extracting the SQL queries

If you've gotten confused and want a clean slate, go ahead and checkout a new branch based on the `cloud.i4` tag.

```bash
git checkout -b iteration5 cloud.i4
```

Otherwise just create a new branch based on the current state of your code:

```bash
git checkout -b iteration5
```

Let's start dealing with that big `compute` method.

The first thing I want to do is get those big SQL strings out of there.
They're very similar, but not identical.

Let's start with the second one, because that is the longest one.

First add a `private` declaration at the bottom of the class.

Then, below it, add an empty method called `sql_90days`.

```ruby
def sql_90days
end
```
Now _copy_ the big query string that gets used for the `@tags_90days` assignment.

```ruby
def sql_90days
  query = "SELECT tags.id, tags.name AS name, count(*) AS count"
  query << " FROM taggings, tags, todos"
  query << " WHERE tags.id = tag_id"
  query << " AND todos.user_id=? "
  query << " AND taggings.taggable_type='Todo' "
  query << " AND taggings.taggable_id=todos.id "
  query << " AND (todos.created_at > ? OR "
  query << "      todos.completed_at > ?) "
  query << " GROUP BY tags.id, tags.name"
  query << " ORDER BY count DESC, name"
  query << " LIMIT 100"
end
```

Find the line where the local variable `query` gets used:

```ruby
@tags_90days = Tag.find_by_sql(
  [query, user.id, @cut_off, @cut_off]
).sort_by { |tag| tag.name.downcase }
```

Replace `query` with `sql_90days`, and then run the `lockdown` test.

It should be passing.

Delete the old `query` definition, and rerun the `lockdown` test. It should
still be passing.

Commit your changes so you can roll back to this point easily if things go
south in the next step.

### Do It Again

Let's do the same thing for the first SQL query string.

Create a method called `sql`, and copy the query string into it.

Verify that the test is passing, delete the old query, and check that the test
is still passing.

Commit your changes.

### Comparing the two SQL queries

Now that the SQL queries are all on their own, let's take a look at how they
compare to each other.

Take a look at the first lines:

```ruby
# sql_90days
"SELECT tags.id, tags.name AS name, count(*) AS count"
```

```ruby
# sql
"SELECT tags.id, name, count(*) AS count"
```

These are equivalent.

Copy the first line of `sql_90days` into `sql`, replacing the first line there.

Lines 2 and 3 of both queries are identical.

Lines 4, 5, and 6 are not. Let's look at them:

```ruby
# sql_90days
query << " AND todos.user_id=? "
query << " AND taggings.taggable_type='Todo' "
query << " AND taggings.taggable_id=todos.id "
```

```ruby
# sql
query << " AND taggings.taggable_id = todos.id"
query << " AND todos.user_id="+user.id.to_s+" "
query << " AND taggings.taggable_type='Todo' "
```

We can reorder those lines in `sql`, putting line 4 after line 6:

```ruby
# sql
query << " AND todos.user_id="+user.id.to_s+" "
query << " AND taggings.taggable_type='Todo' "
query << " AND taggings.taggable_id = todos.id"
```

Now lines 5 and 6 are identical in both queries, and we're stuck with line 4:

```ruby
# sql_90days
query << " AND todos.user_id=? "
```

```ruby
# sql
query << " AND todos.user_id="+user.id.to_s+" "
`

These are essentially the same thing, but in one we're interpolating the
`user.id` id directly into the string, and in the other we're passing the
`user.id` to the `find_by_sql` method as a parameter.

Before we can make these two lines identical we need to tweak the
`find_by_sql` calls in the `compute` method.


The two `find_by_sql` calls look like this:

```ruby
@tags_90days = Tag.find_by_sql(
  [sql_90days, user.id, @cut_off, @cut_off]
).sort_by { |tag| tag.name.downcase }
```

```ruby
@tags = Tag.find_by_sql(sql).sort_by { |tag| tag.name.downcase }
```

First, let's reformat the one-liner to be on 3 lines:

```ruby
@tags = Tag.find_by_sql(
  sql
).sort_by { |tag| tag.name.downcase }
```

The only difference is the second line. Let's assign that line to a variable
named `params`:

```ruby
params = sql
@tags = Tag.find_by_sql(params).sort_by { |tag| tag.name.downcase }
```

```ruby
params = [sql_90days, user.id, @cut_off, @cut_off]
@tags_90days = Tag.find_by_sql(params).sort_by { |tag| tag.name.downcase }
```

Notice how the second `params` is an array? Let's make the first one an array
as well:

```ruby
params = [sql]
@tags = Tag.find_by_sql(params).sort_by { |tag| tag.name.downcase }
```

If you run the `lockdown` test, it should still be passing.

Next, let's go ahead and add the `user.id` to the array, even though it
doesn't get used yet.

```ruby
params = [sql, user.id]
@tags = Tag.find_by_sql(params).sort_by { |tag| tag.name.downcase }
```

The `lockdown` test is still passing`.

And now, finally, we can change the line of code in the `sql` method to be
identical to the one in the `sql_90days` method:

```ruby
query << " AND todos.user_id=? "
```

At this point the last 3 lines of the query are identical as well, but the
`sql_90days` method has 2 extra lines in it:

```ruby
query << " AND (todos.created_at > ? OR "
query << "      todos.completed_at > ?) "
```

Notice the question marks in those two lines? This is where the
`@cut_off` gets used in the call to `find_by_sql`:

```ruby
params = [sql_90days, user.id, @cut_off, @cut_off]
```

So we can pass the cut off to the method and add an if statement around
those lines of code, making the `sql_90days` method look like this:

```ruby
def sql_90days(cut_off = nil)
  query = "SELECT tags.id, tags.name AS name, count(*) AS count"
  query << " FROM taggings, tags, todos"
  query << " WHERE tags.id = tag_id"
  query << " AND todos.user_id=? "
  query << " AND taggings.taggable_type='Todo' "
  query << " AND taggings.taggable_id=todos.id "
  if cut_off
    query << " AND (todos.created_at > ? OR "
    query << "      todos.completed_at > ?) "
  end
  query << " GROUP BY tags.id, tags.name"
  query << " ORDER BY count DESC, name"
  query << " LIMIT 100"
end
```

Update the params to pass in the cutoff to the `sql_90days` method:

```ruby
params = [sql_90days(@cut_off), user.id, @cut_off, @cut_off]
```

The `lockdown` test should still be passing.

### Code Reuse

Let's update the first query to use the `sql_90days` method instead of the
`sql` method.

Find this line of code:

```ruby
params = [sql, user.id]
```

and update it to be like this:

```ruby
params = [sql_90days, user.id]
```

The `lockdown` test should still be passing.

The `sql` method is no longer in use. Go ahead and delete it.

Finally, rename `sql_90days` to just be `sql`.

Then run your test and commit your changes.

## I6: Two `TagCloud` Objects

If you've gotten confused and want a clean slate, go ahead and checkout a new branch based on the `cloud.i5` tag.

```bash
git checkout -b iteration6 cloud.i5
```

Otherwise just create a new branch based on the current state of your code:

```bash
git checkout -b iteration6
```

Notice how the `TagCloud` essentially does the same thing twice, once with no
restrictions, and then a second time with the `90_days` cut off?

We've managed to reduce the duplication of the SQL query by using the
`cut_off` to determine whether or not the result set should actually be
restricted.

Let's expand on that idea so that the whole tag cloud object is either
unrestricted, or for a certain time period.

### Optional Cut Off

In the `TagCloud` initializer, make the `cut_off` parameter optional, with a
default of `nil`:

```ruby
def initialize(user, cut_off = nil)
  @user = user
  @cut_off = cut_off
end
```

Then create two tag clouds in the controller instead of one.

The first does not take a `cut_off`, but the second still does.

The controller method now looks like this:

```ruby
def get_stats_tags
  cloud = TagCloud.new(current_user)
  cloud.compute
  @tags_for_cloud = cloud.tags
  @tags_min = cloud.min
  @tags_divisor = cloud.divisor

  cloud = TagCloud.new(current_user, @cut_off_3months)
  cloud.compute
  @tags_for_cloud_90days = cloud.tags_90days
  @tags_min_90days = cloud.min_90days
  @tags_divisor_90days = cloud.divisor_90days
end
```

This causes the `lockdown` test to fail because we have the wrong number of
`bind` variables for the SQL queries.

We now need to pass in the correct number of bind variables based on whether
or not we actually have a value for `@cut_off` at all.

Change both of the `params` assignments to look like this:

```ruby
params = [sql(@cut_off), user.id]
if @cut_off
  params += [@cut_off, @cut_off]
end
```

The test should be passing again.

In the controller method the first tag cloud is using the first half of the
compute method, whereas the second tag cloud is using the second half of the
compute method.

Change the controller so that none of the calls to `cloud` call methods with
`_90days` in them:

```ruby
def get_stats_tags
  cloud = TagCloud.new(current_user)
  cloud.compute
  @tags_for_cloud = cloud.tags
  @tags_min = cloud.min
  @tags_divisor = cloud.divisor

  cloud = TagCloud.new(current_user, @cut_off_3months)
  cloud.compute
  @tags_for_cloud_90days = cloud.tags
  @tags_min_90days = cloud.min
  @tags_divisor_90days = cloud.divisor
end
```

The `lockdown` test should still be passing.

Having done this we can delete anything in the `TagCloud` referring to
`90days`.

Run the `lockdown` test and commit your changes.

## I7: Sending the tag clouds to the view

If you've gotten confused and want a clean slate, go ahead and checkout a new branch based on the `cloud.i6` tag.

```bash
git checkout -b iteration7 cloud.i6
```

Otherwise just create a new branch based on the current state of your code:

```bash
git checkout -b iteration7
```

Open up the `app/views/stats/_tags.html.erb` file. This is the template for
our tag cloud.

Now that we have these handy tag cloud objects in the controller, let's assign
them to instance variables and use them in the view.

Start out by just adding extra lines with the assignments in them so that we
don't have to change all the code in the controller method:

```ruby
def get_stats_tags
  cloud = TagCloud.new(current_user)
  cloud.compute
  @cloud = cloud # <-- this line is new
  @tags_for_cloud = cloud.tags
  @tags_min = cloud.min
  @tags_divisor = cloud.divisor

  cloud = TagCloud.new(current_user, @cut_off_3months)
  cloud.compute
  @cloud_90days = cloud # <-- this line is new
  @tags_for_cloud_90days = cloud.tags
  @tags_min_90days = cloud.min
  @tags_divisor_90days = cloud.divisor
end
```

Assigning extra variables won't change the output, so our test is still
passing.

Let's go back to the view and replace each instance variable one at a time
with a call to the cloud object that we now have available to us.

First, replace `@tags_for_cloud` with `@cloud.tags`.

The tests failed when I did this.

Take a look at the diff of the files, though:

```bash
diff .lockdown/received.html .lockdown/approved.html
```

The only difference is a single line of extra whitespace. I'm OK with that, so
I'm going to approve the new file:

```bash
```

The only difference is a single line of extra whitespace. I'm OK with that, so
I'm going to approve the new file:

```bash
cp .lockdown/received.html .lockdown/approved.html
```

Next replace `@tags_min` with `@cloud.min` and run the test.

It's still passing.

Replace `@tags_divisor` with `@cloud.divisor`.

Run the test.

Replace `@tags_for_cloud_90days` with `@cloud_90days.tags`, and run the
tests.

Replace `@tags_min_90days` with `@cloud_90days.min` and `@tags_divisor_90days`
with `@cloud_90days.divisor`.

Run the tests.

Now we can go back and clean up the controller, because we're not using some
of those instance variables:

```ruby
def get_stats_tags
  @cloud = TagCloud.new(current_user)
  @cloud.compute
  @cloud_90days = TagCloud.new(current_user, @cut_off_3months)
  @cloud_90days.compute
end
```

Run the test and commit your changes.

## I8: Collapsing duplication in the view

If you've gotten confused and want a clean slate, go ahead and checkout a new branch based on the `cloud.i7` tag.

```bash
git checkout -b iteration8 cloud.i7
```

Otherwise just create a new branch based on the current state of your code:

```bash
git checkout -b iteration8
```

Let's get those instance variables all the way out of the partial.

Open up `app/views/stats/index.html.erb` and find the line that renders the
`tags` partial:

```erb
<%= render :partial => 'tags' -%>
```

Let's pass the tag cloud objects as local variables to the partial.

```erb
<%= render :partial => 'tags', locals: { cloud: @cloud, cloud_90days:
@cloud_90days } -%>
```

The test is still passing, but we're not using those local variables.

Go into the `app/views/stats/_tags.html.erb` file and delete the `@`
characters.

When I did this I got another failure, but again it was just a bit of
whitespace. I just went ahead and approved the new version of the output:

```bash
cp .lockdown/received.html .lockdown/approved.html
```

Now we can call the partial twice from the `index.html.erb` file, once for each tag cloud:

```erb
<%= render :partial => 'tags', locals: { cloud: @cloud } -%>
<%= render :partial => 'tags', locals: { cloud: @cloud_90days } -%>
```

Before we run the tests, we need to delete the second half of the partial.

Run the tests.

Woah! We have more than just whitespace changes this time.

It looks like we changed the following lines:

```html
<h3>Tag cloud actions in past 90 days</h3>
<p>This tag cloud includes tags of actions that were created or completed in
the past 90 days.</p>
```

```html
<h3>Tag cloud for all actions</h3>
<p>This tag cloud includes tags of all actions (completed, not completed,
visible and/or hidden)</p>
```

We missed a spot.

Two of the lines that we deleted looked like this:

```erb
<h3><%= t('stats.tag_cloud_90days_title') %></h3>
<p><%= t('stats.tag_cloud_90days_description') %></p>
```

and the code that gets called instead looks like this:

```erb
<h3><%= t('stats.tag_cloud_title') %></h3>
<p><%= t('stats.tag_cloud_description') %></p>
```

We need to update the translation key to include the `_90days` bit.

Change the calls to render the partials like this:

```erb
<%= render :partial => 'tags', locals: { cloud: @cloud, :key => '' } -%>
<%= render :partial => 'tags', locals: { cloud: @cloud_90days, :key => '_90days' } -%>
```

Also, update the translation keys in the `_tags.html.erb` partial:

```erb
<h3><%= t("stats.tag_cloud#{key}_title") %></h3>
<p><%= t("stats.tag_cloud#{key}_description") %></p>
```

Running the tests again leaves us with only whitespace changes. Approve the
new version of the output:

```bash
cp .lockdown/received.html .lockdown/approved.html
```

Commit your changes.

## I9: Polishing up the TagCloud

If you've gotten confused and want a clean slate, go ahead and checkout a new branch based on the `cloud.i8` tag.

```bash
git checkout -b iteration9 cloud.i8
```

Otherwise just create a new branch based on the current state of your code:

```bash
git checkout -b iteration9
```

### Status

So far we've managed to reduce the amount of code in the controller method
from 50 lines to 4 lines.

We've added a model, `TagCloud`, where we deleted about half of the original
code that was in the controller.

We've also deleted about half of the code in the view layer.

What could possibly be left to do here?

Well, I'm not very happy with how the `TagCloud` class turned out. We can do
better.

There's a lot of bits and pieces here.

First, let's move the `TODO` comment down to the big `sql` method.

Next, create a private method called `levels` and have it return the value 10.

Delete the `levels = 10` line from the compute method.

The `lockdown` test should still be passing.

Let's create a public method called `tags`:

```ruby
def tags
  params = [sql(@cut_off), user.id]
  if @cut_off
    params += [@cut_off, @cut_off]
  end
  @tags = Tag.find_by_sql(params).sort_by { |tag| tag.name.downcase }
end
```

Notice that we've copied the first 5 lines of the `compute` method into it.

Let's get rid of the assignment on the last line:

```ruby
def tags
  params = [sql(@cut_off), user.id]
  if @cut_off
    params += [@cut_off, @cut_off]
  end
  Tag.find_by_sql(params).sort_by { |tag| tag.name.downcase }
end
```

And now in `compute` after the line with the assignment in it, create a new
assignment that calls the tags method:

```ruby
def compute
  params = [sql(@cut_off), user.id]
  if @cut_off
    params += [@cut_off, @cut_off]
  end
  @tags = Tag.find_by_sql(params).sort_by { |tag| tag.name.downcase }
  @tags = tags
  # ...
end
```

Run the test. It passes, so we can delete the 5 lines above our new
assignment.

The tests are passing, but we have some weirdness. We have an `attr_reader`
for `:tags` and we also have a method for `tags`.

We can change the method for `tags` to assign the results of the database
call the first time that it gets called:

```ruby
def tags
  unless @tags
    params = [sql(@cut_off), user.id]
    if @cut_off
      params += [@cut_off, @cut_off]
    end
    @tags = Tag.find_by_sql(params).sort_by { |tag| tag.name.downcase }
  end
  @tags
end
```

Delete the `@tags = tags` assignment in `compute`, and change the remaining
reference to `@tags` in `compute` to `tags`.

Delete the `attr_reader` for `:tags` as well.

Run the test. It should be passing.

Commit your changes.

Take a look at the code that calculates that `min` and `max` tag counts:

```ruby
max, @min = 0, 0
tags.each { |t|
  max = [t.count.to_i, max].max
  @min = [t.count.to_i, @min].min
}
```

First of all, min is always 0, because we never get negative counts out of our
sql query.

So we can simplify:

```ruby
max, @min = 0, 0
tags.each { |t|
  max = [t.count.to_i, max].max
}
```

Next, nstead of checking at every step of the way in the iteration, let's
create a variable called `tag_counts` below this section, which is a simple
array of the counts:

```ruby
tag_counts = tags.map {|t| t.count.to_i}
```

Then, we'll get the max from that array:

```ruby
max = tag_counts.max
```

The test is still passing, so we can delete the old code, leaving us with
this `compute` method:

```ruby
def compute
  @min = 0
  tag_counts = tags.map {|t| t.count.to_i}
  max = tag_counts.max

  @divisor = ((max - @min) / levels) + 1
end
```

We can create a public method called min that just returns `0`, and delete
the `attr_reader` for it. Make sure to change `@min` to `min` in the computation
for `@divisor`.

Run the test.

Next, let's extract the `tag_counts` to a private method:

```ruby
def tag_counts
 @tag_counts ||= tags.map {|t| t.count.to_i}
end
```

Run the test.

Extract `max` to a private method:

```ruby
def max
  tag_counts.max
end
```

Run the test.

Extract divisor to a public method:

```ruby
def divisor
  @divisor ||= ((max - min) / levels) + 1
end
```

Run the test.

This leaves us with an empty compute method. Delete it. Remember to delete the
call to compute from the controller as well.

Run the `lockdown` test, as well as the `wip` rake task, and then commit your
changes.

## I10: Polishing Up the View

If you've gotten confused and want a clean slate, go ahead and checkout a new branch based on the `cloud.i9` tag.

```bash
git checkout -b iteration10 cloud.i9
```

Otherwise just create a new branch based on the current state of your code:

```bash
git checkout -b iteration10
```

Open up the tag cloud partial `app/views/stats/_tags.html.erb`. There's a big
calculation here:

```erb
(9 + 2*(t.count.to_i-cloud.min)/cloud.divisor)
```

This calculates the relative font size for a tag. A font size is definitely a
view concern, but it seems like part of that calculation wants to live in the tag
cloud itself.

Let's create a method in the `TagCloud` called `relative_size` which takes a tag, and put the non-font part of the calculation into it:

```ruby
def relative_size(tag)
  (t.count.to_i - cloud.min) / cloud.divisor
end
```

`t` is now called `tag`:

```ruby
def relative_size(tag)
  (tag.count.to_i - cloud.min) / cloud.divisor
end
```

And we're inside the cloud object, so we don't need to refer to the `cloud`
variable:

```ruby
def relative_size(tag)
  (tag.count.to_i - min) / divisor
end
```

Update the view to use this:

```erb
(9 + 2*cloud.relative_size(t))
```

This is better, but we still probably shouldn't have the calculation in the
view. Let's create a helper method for it.

Open up the `app/helpers/stats_helper.rb` and add the following:

```ruby
def font_size(cloud, tag)
  9 + 2 * cloud.relative_size(tag)
end
```

Update the view again:

```erb
:style => "font-size: #{font_size(cloud, t)}pt"
```

The `lockdown` test is passing.

`min` and `divisor` are no longer referenced outside of the `TagCloud` so we
can make those methods private.

The `lockdown` test is passing.

Let's run the `rake wip` task to be sure that everything is still good. The
tests are all green.

Pat your self on the back, and commit your code.

## I11: Retrospective

Push your changes up to github.

Go to your page on Code Climate and look at the changes that were introduced.

* Did the overall GPA change?
* What is the complexity metric for the `StatsController`?
* What is the duplication metric for the `StatsController`?
* What is the score of the new `TagCloud` class?



