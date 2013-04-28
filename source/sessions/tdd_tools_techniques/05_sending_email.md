---
layout: page
title: Sending Email
section: TDD Tools, Techniques, and Discipline
sidebar: false
---

## Email!

Email is an integral part of every modern web app and sending it from Rails is simple.

Email requires a Mailer class, which you can think of as a controller. Much like
a controller, the actual emails look and behave much like view templates you are
used to.

Let's get into it!

## Add the mailer

Let's send an email to users after they sign up to say hello and thanks.

First let's send an email to the right person:

```ruby
# spec/mailers/mailer_spec.rb
require 'spec_helper'

describe Mailer do
  it 'sends to the correct user' do
    user = User.new(name: 'Adarsh Pandit', email: 'adarsh@thoughtbot.com')

    mail = Mailer.hello(user)

    expect(mail.to).to eq [user.email]
  end
end
```

We run the test and it fails because we have no `Mailer` class:

{% terminal %}
$ rspec spec/mailers/mailer_spec.rb
/railsconf_intro_track/spec/mailers/mailer_spec:3:in `<top (required)>': uninitialized constant Mailer (NameError)
...
{% endterminal %}

Let's make the mailer class. It should live in app/mailers and inherit from
ActiveMailer:

```ruby
# app/mailers/mailer.rb
class Mailer < ActionMailer::Base
end
```

We run the tests again and see we now lack a `hello` action:

{% terminal %}
$ rspec spec/mailers/mailer_spec.rb
F

Failures:

  1) Mailer uses the correct send-to address
     Failure/Error: mail = Mailer.hello(user)
     NoMethodError:
       undefined method `hello' for Mailer:Class
     # ./spec/mailers/mailer_spec.rb:7:in `block (2 levels) in <top (required)>'

Finished in 0.02249 seconds
1 example, 1 failure

Failed examples:

rspec ./spec/mailers/mailer_spec.rb:4 # Mailer uses the correct send-to address
{% endterminal %}

## Add an action

Let's add a `hello` action which takes a user as a parameter:

```ruby
# app/mailers/mailer.rb
class Mailer < ActionMailer::Base
  def hello(user)
  end
end
```

We run our tests again and see the desired failure:
{% terminal %}
$ rspec spec/mailers/mailer_spec.rb
F

Failures:

  1) Mailer.hello sends to the correct user
     Failure/Error: expect(mail.to).to eq user.email

       expected: "adarsh@thoughtbot.com"
            got: nil

       (compared using ==)
     # ./spec/mailers/mailer_spec.rb:19:in `block (2 levels) in <top
(required)>'

Finished in 0.02991 seconds
1 example, 1 failure

Failed examples:

rspec ./spec/mailers/mailer_spec.rb:14 # Mailer.hello sends to the correct user
{% endterminal %}

Great! Now let's add the correct address:

```ruby
# app/mailers/mailer.rb=
class Mailer < ActionMailer::Base
  def hello(user)
    mail(to: user.email)
  end
end
```

Nice, our test is now passing. But email needs a subject, a body, and a 'from'
address. Let's write those tests:

## Add a default from

```ruby
# spec/mailers/mailer_spec.rb
describe Mailer do
  it 'uses the correct send-to address' do
    user = User.new(name: 'Adarsh Pandit', email: 'adarsh@thoughtbot.com')

    mail = Mailer.hello(user)

    expect(mail.from).to eq ['ralph@thoughtbot.com']
  end
end
```

So we get a nice failure:

{% terminal %}
$ rspec spec/mailers/mailer_spec.rb
F

Failures:

  1) Mailer uses the correct send-to address
     Failure/Error: expect(mail.from).to eq ['ralph@thoughtbot.com']

       expected: ["ralph@thoughtbot.com"]
            got: nil

       (compared using ==)
     # ./spec/mailers/mailer_spec.rb:9:in `block (2 levels) in <top (required)>'

Finished in 0.04654 seconds
1 example, 1 failure

Failed examples:

rspec ./spec/mailers/mailer_spec.rb:4 # Mailer uses the correct send-to address
{% endterminal %}

And add the default from to the Mailer class to make it pass:

```ruby
class Mailer < ActionMailer::Base
  default from: 'ralph@thoughtbot.com'

  def hello(user)
    mail(to: user.email)
  end
end
```

Yay! All set. Now let's actually make an email we can send.

## Adding a mail template

Let's write a test which expects the user's name to be included in the text:

```ruby
# spec/mailers/mailer.rb
describe Mailer, '.hello' do
  it 'sends to the correct user' do
    user = User.new(name: 'Adarsh Pandit', email: 'adarsh@thoughtbot.com')

    mail = Mailer.hello(user)

    expect(mail.to).to eq [user.email]
  end

  it 'sends the correct email body' do
    user = User.new(name: 'Adarsh Pandit', email: 'adarsh@thoughtbot.com')

    mail = Mailer.hello(user)

    expect(mail.body.encoded).to match user.name
  end
end
```

Note that `match` will try to find your text within the other, using regex. It
will take regex syntax like `\gr[ae]y\` to match all spellings of 'gray'.

Our test fails the way we want:

{% terminal %}
F

Failures:

  1) Mailer.hello sends the correct email body
     Failure/Error: expect(mail.body.encoded).to include user.name
       expected "" to include "Adarsh Pandit"
     # ./spec/mailers/mailer_spec.rb:27:in `block (2 levels) in <top (required)>'

Finished in 0.04672 seconds
1 example, 1 failure

Failed examples:

rspec ./spec/mailers/mailer_spec.rb:22 # Mailer.hello sends the correct email
body
{% endterminal %}

We make it pass by first making the user accessible to the view by setting an
instance variable:
```ruby
# app/mailers/mailer.rb
class Mailer < ActionMailer::Base
  default from: 'ralph@thoughtbot.com'

  def hello(user)
    @user = user
    mail(to: user.email)
  end
end
```

Now let's make a view template which generates the body of the email:

```ruby
# app/views/mailer/hello.html.erb
Hello <%= @user.name %>!
```
Awesome! Our test passes.

## Add a subject line

Start with the expectation:

```ruby
# spec/mailers/mailer_spec.rb
it 'sends the correct subject line' do
  user = User.new(name: 'Adarsh Pandit', email: 'adarsh@thoughtbot.com')

  mail = Mailer.hello(user)

  expect(mail.subject).to match 'Hello from RailsConf'
end
```

The test failure for this one is interesting:

{% terminal %}
F

Failures:

  1) Mailer.hello sends the correct subject line
     Failure/Error: expect(mail.subject).to match 'Hello from RailsConf'
       expected "Hello" to match "Hello from RailsConf"
     # ./spec/mailers/mailer_spec.rb:35:in `block (2 levels) in <top
(required)>'

Finished in 0.10628 seconds
1 example, 1 failure

Failed examples:

rspec ./spec/mailers/mailer_spec.rb:30 # Mailer.hello sends the correct subject
line
{% endterminal %}

Rails will set the default subject line to the same value as the action name.
In this case, it's 'hello', but capitalized.

Let's make it pass:
```ruby
# app/mailers/mailer.rb
class Mailer < ActionMailer::Base
  default from: 'ralph@thoughtbot.com'

  def hello(user)
    @user = user
    mail(to: user.email, subject: 'Hello from RailsConf')
  end
end
```

Nice! Now we have a bunch of green, let's refactor.

## Refactoring

Now we have the code we want, let's look at it and see how we can improve it.

The mailer code looks fine, but our specs could use some work. The same setup
block is used a number of times. Let's put all of the expectations into one
test:

```ruby
# spec/mailers/mailer_spec.rb
require 'spec_helper'

describe Mailer do
  it 'uses the correct send-to address' do
    user = User.new(name: 'Adarsh Pandit', email: 'adarsh@thoughtbot.com')

    mail = Mailer.hello(user)

    expect(mail.from).to eq ['ralph@thoughtbot.com']
  end
end

describe Mailer, '.hello' do
  it 'sends an email with the correct attributes' do
    user = User.new(name: 'Adarsh Pandit', email: 'adarsh@thoughtbot.com')

    mail = Mailer.hello(user)

    expect(mail.to).to eq [user.email]
    expect(mail.body).to match user.name
    expect(mail.subject).to match 'Hello from RailsConf'
  end
end
```

The default "from" address should apply to all emails, so that can stay
separate, but we added all three expectations to one spec. Our preference is to
stay away from `before` blocks, as it's easy to generate objects you won't use,
or worse, forget which objects were generated.

It is easy to get carried away with this approach and put too much into one
spec. When in doubt, have more specs which are more specific.

## Concluding

You can add more features like  HTML-encoding as well, but this illustrates the
basics.

Also consider using the `email_spec` gem (https://github.com/bmabey/email-spec).
It gives you some nice matchers so you can write expectations like this:

```ruby
expect(email).to deliver_to 'adarsh@thoughtbot.com'
expect(email).to have_subject /Account confirmation/
expect(email).to have_body_text /Adarsh Pandit/
```
It's clearer to read, which is always a benefit.

Wherever we want to send the email in our app, we just call:
```ruby
Mailer.hello(user).deliver
```
and Rails will send the email for us. You might want to do this in a background
job, which we will touch on later.

To learn more, check out the [Rails Guide][1].

[1]: http://guides.rubyonrails.org/action_mailer_basics.html
