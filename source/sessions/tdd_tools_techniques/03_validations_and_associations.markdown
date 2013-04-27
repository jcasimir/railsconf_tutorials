---
layout: page
title: Testing Associations and Validations
section: TDD Tools, Techniques, and Discipline
sidebar: false
---

Before adding any code to our ActiveRecord models we'll want to write failing tests.
This includes code for both Associations and Validations.

## Testing validations

The basic premise behind testing validations is to initialize an object with an
invalid value for the attributes we'd like to test.

ActiveRecord includes a method `valid?` which runs the validations on an object
and returns `true` or `false` based on whether the object current state is valid.

In this case we'll pass in a blank `email`, and verify that the current state of
the `user` is not valid.

```ruby
# spec/models/user_spec.rb
require 'spec_helper'

describe User do
  it 'validates presence of an email address' do
    user = User.create(email: '')
    expect(user.errors[:email]).to match /can't be blank/
  end
end
```

## Shoulda-Matchers

We'll use [shoulda-matchers][1] to add a new DSL around testing validations and
associations in our specs.

This will allow one-line assertions for validations:

```ruby
# spec/models/user_spec.rb
require 'spec_helper'

describe User, 'validations' do
  it { should validate_presence_of :email }
end
```

We can do the same for associations:

```ruby
# spec/models/user_spec.rb
require 'spec_helper'

describe User, 'associations' do
  it { should have_many :posts }
end
```

## Failing Tests

{% terminal %}
$ rspec spec/models/user_spec.rb
Failures:

  1) User validations
     Failure/Error: it { should validate_presence_of(:email) }
       Expected errors to include "can't be blank" when email is set to nil, got no errors
     # ./spec/models/user_spec.rb:9:in `block (2 levels) in <top (required)>'
{% endterminal %}

Add the code to our model to make the test pass.

```ruby
# app/models/user.rb
class User < ActiveRecord::Base
  has_many :posts

  validates :email, presence: true, uniqueness: true
end
```

Run the tests again and make sure that the validations are passing.

{% terminal %}
$ rspec spec/models/user_spec.rb -fd

User validations
  should require email to be set

User associations
  should have many posts

Finished in 0.67573 seconds
2 examples, 0 failures

Randomized with seed 43099
{% endterminal %}

[1]: https://github.com/thoughtbot/shoulda-matchers
