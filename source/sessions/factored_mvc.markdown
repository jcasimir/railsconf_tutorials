---
layout: page
title: Properly Factored MVC
sidebar: true
---

## Intro

We will be using a long-lived open source rails application named [tracks](http://codeclimate.com/github/jumpstartlab/tracks) for this workshop.

It is a todo list application inspired by David Allen's [Getting Things Done](http://www.amazon.com/Getting-Things-Done-Stress-Free-Productivity/dp/0142000280).

The first commit on the application was made in 2006. It currently has well over 200 forks, and is still under active development.

## Setup

### Prerequisites

* ruby 1.9.3
* bundler
* firefox (cucumber features)
  I have a .dmg that can be passed around if necessary


### Code Base

0. git clone git://github.com/jumpstartlab/tracks.git
1. cp config/site.yml.tmpl config/site.yml
2. cp config/database.yml.tmpl config/database.yml
3. bundle install
4. rake db:create db:migrate db:test:prepare
5. bundle exec rake wip

If the tests run correctly when you run `rake wip`, then you're all set.


