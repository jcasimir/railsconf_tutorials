---
layout: page
title: Generating test data
section: TDD Tools, Techniques, and Discipline
sidebar: false
---

## Where we started (fixtures)

Fixtures is a fancy word for sample data. Fixtures allow you to populate your
testing database with predefined data before your tests run.

```yml
# spec/fixtures/users.yml
john:
  email: john@example.com
  password_hash: [\xAAa\xE4\xC9\xB9??\x06\x82%\vl\xF83\e~\xE6\x8F\xD8
```

The fixtures are loaded during test runs, and we can then access the fixtures in
our specs:

```ruby
# spec/models/user_spec.rb
require 'spec_helper'

describe User, '.authenticate' do
  fixtures :all

  it 'authenticates with email and password' do
    user = User.authenticate('john@example.com', 'example')
    expect(user).to eq users(:john)
  end
end
```

Fixtures are often hard to follow, and its not obvious from the fixture files
what the raw data means. I.e. in the above example the password 'secret' is
stored as a SHA1 Hash in the database and its not obvious from the fixture file
what the original value was.

## Using Objects

Another option is to use the objects in your codebase directly.

```ruby
require 'spec_helper'

describe User, '.authenticate' do
  it 'authenticates with email and password' do
    user = User.create!(email: 'john@example.com', password: 'secret')

    result = User.authenticate('john@example.com', 'secret')

    expect(result).to eq user
  end
end
```

This works well until we start adding other required fields to the object. Lets
say we add required fields `first_name`, `last_name`.

```ruby
require 'spec_helper'

describe User, '.authenticate' do
  it 'authenticates with email and password' do
    user = User.create!(
      email: 'john@example.com',
      first_name: 'John',
      last_name: 'Doe',
      password: 'secret'
    )

    result = User.authenticate('john@example.com', 'secret')

    expect(result).to eq user
  end
end
```

As you can imagine this becomes a maintenance nightmare as new fields are added to
models in your application. All tests will need to be updated.

## Factories (hello FactoryGirl)

We can solve the issue of valid data by using Factories. Factories can be used
to generate valid default values for the attributes of our models.

We'll use [factory girl][1] to generate our test data.

```ruby
# spec/factories.rb
FactoryGirl.define do
  factory :user do
    email 'john@example.com'
    password: 'secret'
    first_name 'John'
    last_name 'Doe'
  end
end
```

Now that we've defined our factory for `user` we can use it in our tests.

```ruby
require 'spec_helper'

describe User, '.authenticate' do
  it 'authenticates with email and password' do
    user = FactoryGirl.create(:user, email: 'john@example.com', password: 'secret')

    result = User.authenticate('john@example.com', 'secret')

    expect(result).to eq user
  end
end
```

If there is a unique constraint on the `email` in the `User` model we can use
FactoryGirl sequences to generate unique data.

```ruby
# spec/factories.rb
FactoryGirl.define do
  sequence :email do |n|
    "person#{n}@example.com"
  end

  factory :user do
    email { generate(:email) }
    password: 'secret'
    first_name 'John'
    last_name 'Doe'
  end
end
```

This will generate unique email handles `person1`, `person2`, etc.

[1]: https://github.com/thoughtbot/factory_girl
