---
layout: page
title: Ruby Libraries Important for Rails
sidebar: true
---

## Getting Started

Please open the GitHub repo at https://github.com/mhartl/ruby-libraries

## Important libraries

Here are some important Ruby libraries for Rails (links are to the Ruby&nbsp;2.0 docs, but the results are virtually the same for Ruby&nbsp;1.9.3):

### Classes

* [Array](http://ruby-doc.org/core-2.0/Array.html)
* [Object](http://ruby-doc.org/core-2.0/Object.html)
* [NilClass](http://ruby-doc.org/core-2.0/NilClass.html)
* [Numeric](http://ruby-doc.org/core-2.0/Numeric.html)
* [String](http://ruby-doc.org/core-2.0/String.html)
* [Dir](http://ruby-doc.org/core-2.0/Dir.html)
* [File](http://ruby-doc.org/core-2.0/File.html)
* [Hash](http://ruby-doc.org/core-2.0/Hash.html)
* [Integer](http://ruby-doc.org/core-2.0/Integer.html)
* [Proc](http://ruby-doc.org/core-2.0/Proc.html)
* [Random](http://ruby-doc.org/core-2.0/Random.html)
* [Range](http://ruby-doc.org/core-2.0/Range.html)
* [Regexp](http://ruby-doc.org/core-2.0/Regexp.html)
* [Symbol](http://ruby-doc.org/core-2.0/Symbol.html)
* [Time](http://ruby-doc.org/core-2.0/Time.html)
  
### Modules

* [Kernel](http://ruby-doc.org/core-2.0/Kernel.html)
* [Enumerable](http://ruby-doc.org/core-2.0/Enumerable.html)
* [Math](http://ruby-doc.org/core-2.0/Math.html)

## Workshop

The details of the workshop depend on your background.

### Option 1: Use irb

I suggest that beginning students pick a class or module to learn and then use irb to work through the documentation examples. Post questions to the [RailsConf tutorial chat room](http://railsconftutorials.com/chat).

### Option 2: Use RSpec

If you are comfortable with writing tests, I recommend that you clone the `ruby-libraries` Git repository and use the sample spec file to write documentation of your own.

First, clone the repo:

    git clone https://github.com/mhartl/ruby-libraries.git

Put your GitHub username in the [RailsConf tutorial chat room](http://railsconftutorials.com/chat) and I'll add you as a collaborator on the repo so that you can push to it.

Next, pick a class or module to learn and make a spec for it (prefixing it with your username to avoid clashes):

    cp spec/sample_spec.rb spec/<username>_<class or module>_spec.rb

Follow the model from the sample spec to help you get started. 

When you have changes you want to share, add the file and push it up:

    git add spec/
    git commit -am "Add tests for <class or module>"
    git push

If you want, post a link to the commit in the [RailsConf tutorial chat room](http://railsconftutorials.com/chat) so that I can pull your changes and run the resulting tests.