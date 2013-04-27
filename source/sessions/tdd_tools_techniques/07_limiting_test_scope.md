---
layout: page
title: Limiting test scope
sidebar: false
---

Using Stubs and Spies allows us to test the interactions between collaborating
objects.

## Listen to your tests

If mocking and stubbing external dependencies gets complex there is a chance
that the tests are telling you the class is doing too much.

## Stubbing and Spying

A method stub is an implementation that returns a pre-determined value. Method
stubs can be declared on test doubles or real objects using the same syntax.
rspec-mocks supports 3 forms for declaring method stubs.

We'll use the [rspec-mocks][1] to create test doubles. The test double is a
collaborator. We won't test the methods on the doubles, we'll just make sure
they are called with the correct data.

This way we can ensure that a class methods get called with the correct arguments.

```ruby
# spec/models/importer.rb
require 'spec_helper'

describe Importer, '#import' do
  it 'enumerates emails and creates users' do
    user = double(create: true)
    emails = 'harlow@thoughtbot.com, adarsh@thoughtbot.com'

    Importer.new(contact, emails).import

    expect(contact).to have_received(:create).with('harlow@thoughtbot.com')
    expect(contact).to have_received(:create).with('adarsh@thoughbot.com')
  end
end
```

