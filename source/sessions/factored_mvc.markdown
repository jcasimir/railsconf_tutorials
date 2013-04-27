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

0. `git clone git://github.com/jumpstartlab/tracks.git`
1. `cp config/site.yml.tmpl config/site.yml`
2. `cp config/database.yml.tmpl config/database.yml`
3. `bundle install`
4. `bundle exec rake db:create db:migrate db:test:prepare`
5. `bundle exec rake wip`

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

If you've gotten confused and want to get a clean slate, go ahead and checkout a new branch based on the `cloud.i2` tag.

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

