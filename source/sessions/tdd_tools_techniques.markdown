---
layout: page
title: TDD - Tools, Techniques, and Discipline
sidebar: false
---

We will be building features in a test-driven approach. This is best for
participants who are familiar with Rails and have implemented some basic
features within an application. That being said, all are encouraged to
attend/watch.

For the above reasons, we _strongly_ encourage pairing with someone else during
the workshop. Also, if you see others struggling, please help where possible.

For total beginners, we recommend checking out the learning resources at our
[Rails trail-map on our Learn Portal][4].

## Smoke Test

Clone the repository and make sure you can run `rspec` from the root of the project:

{% terminal %}
$ git clone git@github.com:harlow/railsconf_intro_track.git
$ cd railsconf_intro_track
$ bundle install
$ bundle exec rspec
{% endterminal %}

## Workshop 1

Outward-in devleopment, unit tests, and fixture data.

- [Introduction][5]
- [Outside-In and Integration Testing][6]
- [Testing Associations and Validations][7]
- [Generating test data][8]
- Sending email

## Workshop 2

Mocking, stubbing, and faking external services.

- Introduction and quick recap from Session 1
- Limiting test scope
- Running background processes
- Working with external API services

## Contact Us

Don't hesitate to reach out to us if you have problems getting your system set
up [harlow@thoughtbot.com][2] or [adarsh@thoughtbot.com][3]

[1]: https://github.com/harlow/railsconf_intro_track
[2]: mailto://harlow@thoughtbot.com
[3]: mailto://adarsh@thoughtbot.com
[4]: https://learn.thoughtbot.com/rails
[5]: {% page_url 01_introduction %}
[6]: {% page_url 02_integration_testing %}
[7]: {% page_url 03_validations_and_associations %}
[8]: {% page_url 04_generating_test_data %}
