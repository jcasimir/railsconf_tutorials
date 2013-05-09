---
layout: page
title: Integration Testing and Outside-In Development
section: TDD Tools, Techniques, and Discipline
sidebar: false
---

Integration testing is the process of "integrating" your software pieces and
testing them as a whole. Rails builds web applications, so testing the full
stack means starting a web server, opening a browser, and mousing through the
normal functions of your application.

The evolution of Integration Testing looks something like this:

- Early days: Write a script, start a web server, pay some poor soul to click
  through the app in a specific fashion, noting when things break. (Tools:
  mouse, patience)
- Progress: Automation of the above process using a live browser, using DOM
  elements as anchors. (Tools: Gherkin for scripting. Cucumber as the driver)
- More progress: Automated browser scripting using a headless browser, which is
  much faster. (Tools: Cucumber + capybara-webkit)

## Cucumber

Many people use Cucumber for writing Integration tests and it's amazing. You
write in plain English what you want the browser to do _in User Story format_
and it does it. It looks like this:

```cucumber
# spec/feature/user_log_in.feature
Feature: User signs in

Scenario: Valid account information
  Given I am a user
  When I go to the sign-in page
  And I fill in the session email with "ralph@example.com"
  And I fill in the session password with "ralph_was_here"
  And I press the button to sign in
  Then I should be on my account page
```

You run the script and if any of those steps don't work, it'll fail. By the way,
the language of Cucumber is called [Gherkin][1] (get it?).

Now, Cucumber isn't some amazing natural language interpreter - it needs to
match those sentences to some real instructions. They are called 'step
definitions' and look like this:

```cucumber
# spec/feature/step_definitions/user_steps.rb
Given /^I am a user$/ do
  user = create(:user)
end

Given /^When I go to the sign-in page$/ do
  visit sign_in_path
end

And /^I press the button to sign in$/ do
  click_button 'Sign In'
end
```

The RegEx in the step definition allows Cucumber to match Gherkin to a step. The
syntax inside the block is Capybara's syntax, which drives the browser to do
it's bidding.

The big advantage of this approach is anyone can write the Feature Specs,
including non-technical clients. The other plus is that now you can read this
document easily to know what the application should be able to do.

There is a big downside, which is maintainence. The Gherkin script is very
tightly coupled to the step definitions, so if you change one, you have to
change the other one. This is a bummer and in large suites will eat up a lot of
time.


## The Future is Now: RSpec + Capybara

We have taken to writing integration tests using RSpec + Capybara. RSpec is our
preferred language of choice for writing unit tests. This approach is the same
except using the Capybara DSL (e.g. `visit some_page`, `click_button`).

It looks like this:

```ruby
# spec/features/user_signs_in_spec.rb
require 'spec_helper'

feature 'User signs in' do
  scenario 'and sees their profile page' do
    user = create(:user)

    visit '/sign_in'
    fill_in 'Email', with: user.email
    click_button 'Sign In'

    expect(page).not have_content('My Profile Page')
  end
end
```

It doesn't depend on any other code, is nicely encapsulated, easily readable,
and it's in Ruby. More on this approach is [here in our blog post][2].

## Coding Example

Let's add a test to make sure we can see the homepage:

```ruby
# spec/features/view_the_homepage_spec.rb
feature 'View the homepage' do
  scenario 'visitor sees relevant information' do
    visit root_path

    expect(page).to have_css 'h1', text: 'RailsConf 2013 Intro Track'
  end
end
```

Now let's run the test by typing `rspec spec/features/view_the_homepage_spec.rb` at the command line and see it fail.

We see a routing error where Rails doesn't understand what a `root_path` is.

{% terminal %}
  $ rspec spec/features/view_the_homepage_spec.rb

    1) View the homepage visitor sees relevant information
      Failure/Error: visit root_path
      NameError:
       undefined local variable or method `root_path' for
#<RSpec::Core::ExampleGroup::Nested_1:0x007fcf6963e9e0>
     # ./spec/features/view_the_homepage_spec.rb:5:in `block (2 levels) in <top
(required)>'
{% endterminal %}

Let's route root_path requests to `/app/views/pages/home.html.erb`:

```ruby
# config/routes.rb
TddIntro::Application.routes.draw do
  root to: 'pages#home'
end
```

Run the tests again and see the new response.

{% terminal %}
  $ rspec spec/features/view_the_homepage_spec.rb
  1) View the homepage visitor sees relevant information
     Failure/Error: expect(page).to have_css('h1', text: 'RailsConf 2013 Intro
Track')
     Capybara::ExpectationNotMet:
       expected to find css "h1" with text "RailsConf 2013 Intro Track" but
there were no matches
     # ./spec/features/view_the_homepage_spec.rb:7:in `block (2 levels) in <top
(required)>'

Finished in 0.08735 seconds
1 example, 1 failure

Failed examples:

rspec ./spec/features/view_the_homepage_spec.rb:4 # View the homepage visitor
sees relevant information
{% endterminal %}

This error tells us the page doesn't have the information we want - an H1 tag
with our desired copy text. Let's write some HTML to make this pass:

```html+erb
# app/views/pages/home.html.erb
<h1>RailsConf 2013 Intro Track</h1>
```

Run the test again and we should see passing output.

## Refactor

Now is when we would look at our code using something like `git diff` and
refactor or improve the code. This is pretty straightforward, so there's not
much to refactor here.

Let's commit this example and move on.

## Wrapping up

That was one very simple cycle of TDD.

- We wrote a test to set the expected behavior we wanted out of our application.
- Then we wrote only enough code to fix the routing failure, which was the
  current error.
- Finally we added the view template with expected copy to fix the copy error.
- Then the test passed!

This is a very simple example, but it gives you a sense of how TDD is done.

Next step is dive down to an ActiveRecord model and  [test Validations and Associations][3]

[1]: https://github.com/cucumber/cucumber/wiki/Gherkin
[2]: http://robots.thoughtbot.com/post/33771089985/rspec-integration-tests-with-capybara
[3]: {% page_url 03_validations_and_associations %}
