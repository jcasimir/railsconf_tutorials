---
layout: page
title: Running background processes
section: TDD Tools, Techniques, and Discipline
sidebar: false
---

## Synchronous vs. Asynchronous jobs

When your web application is processing a request, it may do a number of things
before serving up a page. It may read from the database, send an email, and
render a template before returning a response to the user.

Each of these sub-tasks is run in series, so if one is slow, then the response
time for the whole app can be slowed.

If some work being done by the application doesn't need to be done in real-time,
you can consider doing the work in a background process. These are worker
processes which handle a [FIFO][1] stack of 'jobs' at their own pace, seconds or
minutes after the job is created.

## Good candidate tasks for background jobs

* Sending email
* External API requests
* Complicated calculations
* Large database writes
* Image manipulation
* Uploads / Downloads
* Enqueuing other jobs

## Delayed::Job

Our tool of choice is [Delayed::Job][2], which works on top of ActiveRecord and
your database of choice (Postgres). Also, it integrates nicely with Heroku's default
stack.

Let's install it first in the `Gemfile`:

```ruby
source 'https://rubygems.org'

gem 'delayed_job_active_record'
gem 'jquery-rails'
gem 'rails', '3.2.13'
gem 'simple_form'
gem 'sqlite3'
...
```

Next run `bundle install` to install Delayed::Job (DJ).

The Active Record backend requires a jobs table. You can create it using a
generator, then running the migration it creates:

{% terminal %}
$ rails generate delayed_job:active_record

      create  script/delayed_job
       chmod  script/delayed_job
      create  db/migrate/20130428031829_create_delayed_jobs.rb

$ rake db:migrate
==  CreateDelayedJobs: migrating ==============================================
-- create_table(:delayed_jobs, {:force=>true})
   -> 0.0191s
-- add_index(:delayed_jobs, [:priority, :run_at],
{:name=>"delayed_jobs_priority"})
   -> 0.0005s
==  CreateDelayedJobs: migrated (0.0198s) =====================================
$
{% endterminal %}

## Delaying an email

We never got around to sending that email we created before. Let's do that and
run it in the background:

```ruby
# spec/models/user_spec.rb
require 'spec_helper'

describe User, 'callbacks' do
  it 'sends a welcome email after creation' do
    mailer_stub = stub(:deliver)
    Mailer.stubs(:hello).returns(mailer_stub)

    user = create(:user)

    expect(Mailer).to have_received(:hello).with(user)
    expect(mailer_stub).to have_received(:deliver)
  end
end
```
So this test stubs a method on `Mailer` called `hello`, which returns another
stub which responds to `deliver` as we saw previously. Then it creates a user
(using FactoryGirl), and checks to see if the mailer has received the message
with the `user` as a parameter, and then `deliver` is called on that.

The code looks like this:
```ruby
# app/models/user.rb
class User < ActiveRecord::Base
  attr_accessible :email, :name

  after_create :send_hello_email

  private

  def send_hello_email
    Mailer.hello(self).deliver
  end
end
```

Here we are using a callback, which is a hook into the ActiveRecord lifecycle.
This will watch for any creations of `User` objects, then after creation, will
call the `send_hello_email` method.

Okay, let's delay this email sending the simple way:

## Delayed::Jobs - the simple way

This is the miracle of Delayed::Jobs - to send something to the queue and run
in the background, add `.delay` between the object and method call. For example:

```ruby
# without delayed_job
user.activate(param)

# with delayed_job
user.delay.activate(param)
```

That's it. Before we begin testing DJs, we have to make sure they don't get run
after creation (default behavior). This way we can make assertions about the
jobs before they are run.

```ruby
# spec/spec_helper.rb
RSpec.configure do |config|
  before(:each). do
    Delayed::Worker.delay_jobs = true
  end
...
```

With this setting, DJ will actually delay the jobs, as opposed to just running
them right away. That way we can set up an assertion like this:

```ruby
# spec/models/user_spec.rb
require 'spec_helper'

describe User, 'callbacks' do
  it 'sends a welcome email after creation' do
    mailer_stub = stub(:hello)
    Mailer.stubs(:delay).returns(mailer_stub)

    user = create(:user)

    expect(Mailer).to have_received(:delay)
    expect(mailer_stub).to have_received(:hello).with(user)
  end

  it 'delays a job' do
    user = create(:user)

    expect(Delayed::Job.count).to eq 1
    expect(Delayed::Job.last.handler).to match user.name
    expect(Delayed::Job.last.handler).to match user.email
  end
end
```

We've done a few things here so let's walk through them.

First, we've changed our first test to expect something like
`Mailer.delay.hello(user)` which has a `delay` in the middle *and* no `deliver`
call on the end (DJ knows what to do here). We change our expectations to match.

There are other ways to stub a chain of calls like this like RSpec's
`stub_chain` method. We are taking the simple approach here.

Second, we create a user, which we expect enqueues a DJ, then assert things
about the new job, like it's presence, and that the handler code contains our
user's email and name.

Let's make it pass:
```ruby
# app/models/user.rb
class User < ActiveRecord::Base
  attr_accessible :email, :name

  after_create :send_hello_email

  private

  def send_hello_email
    Mailer.delay.hello(self)
  end
end
```

We add `delay` and pull off the `deliver` as we expected and our tests pass. Now
we send emails in the background instead of clogging up the request cycle.

## Custom Delayed::Jobs

DJ will also allow you to enqueue an instance of a job class as long as it
responds to `perform`. This is a next-level concept, so I'll just introduce it
quickly here.

Something like:

```ruby
class HelloEmailJob < Struct.new(:user)
  def perform
    Mailer.hello(user)
  end
end

Delayed::Job.enqueue(HelloEmailJob.new(user))
```

This custom job allows you to put more logic in the custom job class, specify
job priorities, and add job lifecycle callbacks.

Testing it would look like this:
```ruby
it 'should enqueue a new HelloMail job' do
  Delayed::Job.stubs(:enqueue)

  create(:user)
  hello_email_job = HelloEmailJob.new(user)

  expect(Delayed::Job.to have_received(:enqueue).
    with(hello_email_job
end
```

## Conclusion
There are other job queue handlers out there, notably [Resque][3] which runs on
top of Redis. It's much more performant but for us, seems like it's not worth
using unless you have a ton of traffic, or already use Redis.

[1]: http://en.wikipedia.org/wiki/FIFO
[2]: https://github.com/collectiveidea/delayed_job
[3]: https://github.com/resque/resque
