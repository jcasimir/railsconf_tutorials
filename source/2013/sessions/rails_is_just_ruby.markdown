---
layout: page
title: Rails is Just Ruby
sidebar: true
---

Rails: the result of magical incantations, voodoo, and wizardry? Or: a
collection of patterns from the most awesomest language in the world (Ruby)?

<div style="width:800px; margin-bottom: 20px;">
<script async class="speakerdeck-embed" data-id="c45531409280013062881e5646d94223" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>
</div>

## Update

The repo was fixed for the problem where you would need to restart the
server. The
[fix](https://github.com/jwo/railsconftutorial-beeper/commit/777988657abd3472ebec9250ff731836a7ef84c8)
removes Rails auto-loading for the lib directory. It's probably easiest for you
to delete your directory and start over, though if you pull in or replicate the
commit you'll be all set.

## Workshop

Yay! Let's write some code. We'll start from a small Rails 3.2.13 application
that logs alarms and notifies you. Think of it as a honeybadger/airbrake system.

### What we need to do

Section 1: 20 minutes

* Send notifications via SMS Text
* Get setup on Twilio

Section 2: 20 minutes

* use a model `before_validation` callback to set severity depending on the
  service. If it's redis, treat it as critical, else warning
* Either send by SMS or send by Email depending on severity
* Add a controller `before_filter` to easily have some security

## Let's get started

{% terminal %}
$ git clone git@github.com:jwo/railsconftutorial-beeper.git
$ cd railsconftutorial-beeper
$ bundle
$ rake db:create db:migrate
{% endterminal %}

### We need to setup Twilio to send texts

1. Go to [https://www.twilio.com/try-twilio](https://www.twilio.com/try-twilio)
2. Enter your mobile phone number
3. You'll receive a text instantly, enter your confirmation number
4. Fill out the form (it's free)
5. Get the two values (SID and Token):

![empty-alarms](/images/rails-is-just-ruby-1.png)

Edit the config/initializers/sms.rb and fill out the Beeper.config

#### TIP: If Twilio takes too long (5 minutes max), use these values below:

```ruby
Beeper.config do |config|
  config.twilio_phone_number  = '(281) 940-8409'
  config.twilio_sid           = 'AC98c1274bcab7b6beb8f4164f61e00836'
  config.twilio_auth_token    = '01d11a589086716b617708f8c88385b5'
  config.your_phone_number    = 'XXX'
  config.your_email_address   = 'XXX'
end
```

#### Let's test out and make sure you can send texts

{% terminal %}
$ bundle exec rake beeper:test:sms
{% endterminal %}

You should receive a text instantly. If you don't, you'll need to check the
config values.

Let's start the Rails server and get some errors reported to us!

{% terminal %}
$ rails s
{% endterminal %}

Open your browser to [http://0.0.0.0:3000](http://0.0.0.0:3000). It should look
like this:

![empty-alarms](/images/rails-is-just-ruby-2.png)

And Open a new Terminal Window and we'll report two errors, one on email and one
on redis

{% terminal %}
$ rake alarm:report service=email
$ rake alarm:report service=redis
{% endterminal %}

If we refresh our browser, we'll see the alarms:

![we-have-alarms](/images/rails-is-just-ruby-3.png)

But!!! We weren't notified. Let's put the Beeper in this Beeper app!

## Section 1: Notification via SMS

Open up the Alarms Controller.  The two methods we care about here are the

* HINT: notice the create action has a before filter that creates the alarm

```ruby
  def create
    @alarm.start_alarm
    respond_with @alarm
  end

  def destroy
    @alarm = Alarm.find_by_service! params[:id]
    @alarm.stop_alarm
    respond_with @alarm
  end
```

We want to send SMS text when `start_alarm` is run, so let's send in the code we want @alarm to run. That code looks like:

```ruby
Beeper::SMS.notify "WARNING: Alarm has been opened for #{alarm.service}"
```

To send that in, let's use a block:

```ruby
  def create
    @alarm.start_alarm do |alarm|
      Beeper::SMS.notify "WARNING: Alarm has been opened for #{alarm.service}"
    end
    respond_with @alarm
  end
```

And, let's report an error.
{% terminal %}
$ rake alarm:report service=email
{% endterminal %}

DING! (you should have received a text)

And, let's do the same when an alarm closes (it stops being a problem)

```ruby
  def destroy
    @alarm = Alarm.find_by_service! params[:id]
    @alarm.stop_alarm do |alarm|
      Beeper::SMS.notify "CLEARED: Alarm #{alarm.service} has been cleared"
    end
    respond_with @alarm
  end
```

And, let's close the error log:

{% terminal %}
$ rake alarm:close service=email
{% endterminal %}

Congrats, that's section 1!

## Section 2: SMS OR Email


OK, so right now, Beeper will send texts if the service is critical or if
it's just a warning. Let's only send SMS if it's critical

First, let's add a `critical?` method to our alarm to user. Open up
app/models/alarm.rb and add:

```ruby
def critical?
  severity == 'critical'
end
```

Now, let's add the email notifications to the Alarms controller:

```ruby
  def create
    @alarm.start_alarm do |alarm|
      if alarm.critical?
        Beeper::SMS.notify "WARNING: Alarm has been opened for #{alarm.service}"
      else
        Beeper::Email.notify "WARNING: Alarm has been opened for #{alarm.service}"
      end
    end
    respond_with @alarm
  end

  def destroy
    @alarm = Alarm.find_by_service! params[:id]
    @alarm.stop_alarm do |alarm|
      if alarm.critical?
        Beeper::SMS.notify "CLEARED: Alarm #{alarm.service} has been cleared"
      else
        Beeper::Email.notify "CLEARED: Alarm #{alarm.service} has been cleared"
      end
    end
    respond_with @alarm
  end
```

And, let's report a non-critical error, and then clear it
{% terminal %}
$ rake alarm:report service=memcached
$ rake alarm:clear service=memcached
{% endterminal %}

If we look at the development log, we'll see two emails were sent:
{% terminal %}
$ tail -n 20 log/development.log
{% endterminal %}

(if you're on Windows, you can open the log/development.log and scroll to the
bottom)

You'll see something like:

```
Sent mail to jesse@comalproductions.com (10ms)
Date: Sun, 28 Apr 2013 11:47:52 -0500
From: beeper@example.com
To: jesse@comalproductions.com
Message-ID: <517d52b8cd8b1_e9e23fd5a5c28b24341f9@Jesses-MacBook-Air.local.mail>
Subject: Beeper Notification
Mime-Version: 1.0
Content-Type: text/plain;
 charset=UTF-8
Content-Transfer-Encoding: 7bit
```

But now, our critical email service won't ever actually be SMS'd. Let's fix that
by adding a `before_validation` filter in the Alarm Model.

```ruby
before_validation do |alarm|
  if %w(email rails postgres).include? alarm.service
    alarm.severity = 'critical' 
  end
end
```

OK, so now if we send a postgres alarm, we'll get a text. But LOLCATS will give
us an email.


{% terminal %}
$ rake alarm:clear service=postgres
$ rake alarm:clear service=lolcats
{% endterminal %}

You should see:
1. Email in the log file
2. SMS Text about posgtres

YAY

Last thing we want to do: secure the website real quick. We've already setup
authentication, and just need to use the methods in the Alarms controller.

```ruby
  before_filter :only => [:index] do |controller|
    redirect_to login_url unless controller.authenticated?
  end
```

So, we're telling the controller to redirect away unless we're authenticated.
Where is that authenticated method? It's in ApplicationController and available
to all controllers.

Now, when you refresh your browser, you'll be redirected to a login page.

![we-have-alarms](/images/rails-is-just-ruby-4.png)

Enter: `supersecret` and you'll magically gain access again

## Advanced / Homework

* prevent more than 1 notification from being sent
* Rather than sending a block, you could use a lambda and send in actions:
```ruby
  {
    :warning => ->(alarm) {Beeper::Email.notifiy("hi!")},
    :critical => ->(alarm) {Beeper::SMS.notify("hi!")}
   }
```

## About Speaker

* Jesse Wolgamott
* [Freelance Ruby Dev in Houston, Texas](http://comalproductions.com)
* [Ruby Off Rails](http://rubyoffrails.com)
* Twitter: [@jwo](http://twitter.com/jwo)
* GitHub: [@jwo](http://github.com/jwo)
* [More Talks](https://speakerdeck.com/jwo/rails-is-just-ruby)
