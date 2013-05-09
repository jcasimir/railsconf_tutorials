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

A method stub is an implementation that returns a pre-determined value. For example,
you could create a stub like `user_stub = stub(name: 'Bob')`, which would just respond
to `user_stub.name` without and return the string 'Bob'.

We'll use the [rspec-mocks][1] gem to create test doubles. The test double is a
collaborator. We won't test the methods on the doubles, we'll just make sure
they are called with the correct data.

This way we can ensure that a class methods get called with the correct arguments.

Spies watch stubs to see if they get called. For example, if we set up
`user_stub = stub(:name)`, then do some stuff, we can see if name got called on
`user_stub` by saying `expect(user_stub).to have_received(:name)`.

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
