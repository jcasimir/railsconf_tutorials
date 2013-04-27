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
git checkout -b iteration0 cloud.i0
```

This creates a new branch named `iteration0` based off of the git tag
named `cloud.i0`.

In the tag `cloud.i0` we've added a new test file `test/functional/lockdown\_test.rb`.
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

