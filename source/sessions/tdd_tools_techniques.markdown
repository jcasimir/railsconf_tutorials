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

For newcomers to Rails, we recommend checking out the learning resources at our
[Rails trail-map on our Learn Portal][1].

## Smoke Test

Clone [the repository][2] and make sure you can run `rspec` successfully from
the root of the project:

{% terminal %}
$ git clone git@github.com:harlow/railsconf_intro_track.git
$ cd railsconf_intro_track
$ bundle install
$ bundle exec rspec
.

Finished in 0.08985 seconds
1 example, 0 failures

Randomized with seed 52557
{% endterminal %}

## Workshop 1

Outward-in devleopment, unit tests, and fixture data.

- [Introduction][3]
- [Outside-In and Integration Testing][4]
- [Testing Associations and Validations][5]
- [Generating test data][6]
- [Sending email][7]

## Workshop 2

Mocking, stubbing, and faking external services.

- [Limiting test scope][9]
- [Running background processes][10]
- [Working with 3rd party external services][11]

## Contact Us

Don't hesitate to reach out to us if you have problems getting your system set
up [harlow@thoughtbot.com][12] or [adarsh@thoughtbot.com][13]

[1]: https://learn.thoughtbot.com/rails
[2]: https://github.com/harlow/railsconf_intro_track
[3]: {% page_url 01_introduction %}
[4]: {% page_url 02_integration_testing %}
[5]: {% page_url 03_validations_and_associations %}
[6]: {% page_url 04_generating_test_data %}
[7]: {% page_url 05_sending_email %}
[8]: {% page_url 06_intro_second_session %}
[9]: {% page_url 07_limiting_test_scope %}
[10]: {% page_url 08_background_jobs %}
[11]: {% page_url 09_external_services %}
[12]: mailto://harlow@thoughtbot.com
[13]: mailto://adarsh@thoughtbot.com
