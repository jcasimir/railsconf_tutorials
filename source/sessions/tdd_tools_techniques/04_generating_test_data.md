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
  admin: false

jane:
  email: jane@example.com
  password_hash: [\xAAa\xE4\xC9\xB9??\x06\x82%\vl\xF83\e~\xE6\x8F\xD8
  admin: true
```

The fixtures are loaded during test runs, and we can then access the fixtures in
our specs. The call to `users(:john)` loads `john` from the fixtures file:

```ruby
# spec/models/user_spec.rb
require 'spec_helper'

describe User, '.authenticate' do
  fixtures :all

  it 'authenticates with email and password' do
    user = User.authenticate('john@example.com', 'secret')
    expect(user).to eq users(:john)
  end
end
```

Fixtures are often hard to follow, and its not always obvious from the fixture files
what the raw data means. In the above example the password 'secret' is stored as
a SHA1 Hash in the database -- the fixture file doesn't show what the  plain
text value is.

## Using Objects

Another option is to use the objects from your codebase directly.

```ruby
# spec/models/user_spec.rb
require 'spec_helper'

describe User, '.authenticate' do
  it 'authenticates with email and password' do
    user = User.create!(email: 'john@example.com', password: 'secret')

    result = User.authenticate('john@example.com', 'secret')

    expect(result).to eq user
  end
end
```

This works well until we start adding additional required fields to the object.
Let's say we add required fields `first_name`, `last_name`.

```ruby
# spec/models/user_spec.rb
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

As you can imagine this becomes a maintenance nightmare as new attributes are added,
and leads to lots of duplication in the test suite.

## Factories (hello FactoryGirl)

We can solve the issues of creating test data by using Factories. Factories can be
used to generate values for the attributes in our models.

We'll use the [factory girl][1] gem to generate our test data.

Tip: To avoid repeating the text `FactoryGirl` throughout our tests, we can include the
syntax methods and omit the verbose syntax.

```ruby
# spec/support/factory_girl.rb
RSpec.configure do |config|
  config.include FactoryGirl::Syntax::Methods
end
```

Example Before:

```ruby
FactoryGirl.create(:user, email: 'john@example.com', password: 'secret')
```

Example After:

```ruby
create(:user, email: 'john@example.com', password: 'secret')
```

Create a factory for our `user` object.

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
# spec/models/user_spec.rb
require 'spec_helper'

describe User, '.authenticate' do
  it 'authenticates with email and password' do
    user = create(:user, email: 'john@example.com', password: 'secret')

    result = User.authenticate('john@example.com', 'secret')

    expect(result).to eq user
  end
end
```

If there is a unique constraint on the `email` in the `User` model we can use
FactoryGirl sequences to generate unique data. This enables the generation of a
sequence of unique email handles (`person1`, `person2`, etc).

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

To help reduce duplication in factories we can use Traits. These will also us
to put a name around an attribute. Builing on the example abmove we low example
the `User` has a boolean flag `admin`.

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

    trait :admin do
      admin true
    end
  end
end
```

In our specs we can use the trait to create an `admin` user:

```ruby
# spec/models/user_spec.rb
require 'spec_helper'

feature 'Admin creates a post' do
  scenario 'with valid attributes' do
    user = create(:user, :admin)

    # ...
  end
end
```

In a scenario where a `User` could have many `Posts` we can use factories in our
test to create associated data.

```ruby
# spec/factories.rb
FactoryGirl.define do
  factory :post do
    title 'A great post'
    association :user
  end
end
```

In our tests we could create several posts for a user:

```ruby
# spec/models/user_spec.rb
require 'spec_helper'

feature 'User updates a post' do
  scenario 'with valid attributes' do
    user = create(:user, :admin)
    post = create(:post, user: user)

    visit post_path post
    click_link 'Edit post'
    # ...
  end
end
```

Take a look at the [GETTING STARTED][2] page for more features and documentation
on using FactoryGirl.

[1]: https://github.com/thoughtbot/factory_girl
[2]: https://github.com/thoughtbot/factory_girl/blob/master/GETTING_STARTED.md
