---
layout: page
title: Crafting Gems
sidebar: true
---

# Tutorial: Building a Gem

Now that you've heard about building a gem, let's actually build one. These tutorial instructions will step through building a gem that provides the common elements of tags for a Rails app - a pretty common feature.

What we want is to add this line to any model in a Rails app, and then be able to add and remove tags:

```ruby
class Article < ActiveRecord::Base
  has_many_tags
end
```

```ruby
article = Article.create :title => 'My Ruby Adventure'
article.tag_names << 'portland'
article.tag_names << 'railsconf'
article.tag_names << '2013'
article.tag_names #=> ['portland', 'railsconf', '2013']
```

## Step One: give your gem a name

Naming things is one of the hardest parts of writing software. This is just a tutorial though, so let's keep it clean and simple. Pick an animal name for some inspiration, add on "tag_" as a prefix, then check on rubygems.org to see if it's taken. Personally, I'm going with `tag_echidna`, so you'll need to pick something else (but I'm going to use it throughout the example code).

## Step Two: First files and folders

1. Create a directory for your gem - call it whatever your gem's called (tag_echidna).
2. Create a directory within your new project directory with the name 'lib'.
3. Create a ruby file in your lib directory using the name of your gem. Leave the contents blank for the moment.
4. Create a gemspec file for your gem - again using your gem name, with gemspec as the file extension. Don't worry about the contents for the moment.

You should have a directory/file structure like this:

* `tag_echidna/lib/tag_echidna.rb`
* `tag_echidna/tag_echidna.gemspec`

We'll be working within our project directory, so make sure you jump into that, and then set the project up as a git repository - we don't want to lose all of our hard work!

{% terminal %}
$ cd tag_echidna
$ git init
{% endterminal %}

Next, let's fill out our gemspec file. Here's a template, but put your name, email address and gem name in.

```ruby
# -*- encoding: utf-8 -*-
Gem::Specification.new do |s|
  s.name        = 'tag_echidna'
  s.version     = '0.0.1'
  s.authors     = ['Pat Allan']
  s.email       = ['pat@freelancing-gods.com']
  s.homepage    = 'https://github.com/pat/tag_echidna'
  s.summary     = 'The Tag Echidna'
  s.description = 'This friendly echidna gives you tags in your Rails app.'

  s.files         = `git ls-files`.split("\n")
  s.test_files    = `git ls-files -- {spec}/*`.split("\n")
  s.require_paths = ['lib']
end
```

Now's a perfect time to add all of our work thus far into a git commit:

{% terminal %}
$ git add tag_echidna.gemspec
$ git add lib/tag_echidna.rb
$ git commit -m "The start of a gem"
{% endterminal %}

## Step Three: Ship it

We already have a valid gem though - and to prove it, let's get it published. First, let's build the gem file:

{% terminal %}
$ gem build tag_echidna.gemspec
{% endterminal %}

Then, go and register an account on rubygems.org if you haven't done that already. Once you have an account, you can upload your gem:

{% terminal %}
$ gem push tag_echidna-0.0.1.gem
{% endterminal %}

Congratulations, you just published a gem! You can go ahead and install it (it may take rubygems.org a minute to catch up, though):

{% terminal %}
$ gem install tag_echidna
{% endterminal %}

## Step Four: Getting Tests In Place

We have a gem, but it doesn't do anything! Let's remedy that quickly.

Every good gem has a test suite, and ours will be no exception - though I know we're a little pressed for time, so I've written one for you.

### `spec/spec_helper.rb`

```ruby
require 'bundler'

Bundler.require :default, :development

Combustion.initialize! :active_record

require 'rspec/rails'

RSpec.configure do |config|
  config.use_transactional_fixtures = true
end
```

### `spec/acceptance/tags_spec.rb`

Change TagEchidna in this test to match your gem name (with the corresponding upper/lower case).

```ruby
require 'spec_helper'

describe 'Adding and removing tags' do
  let(:article)  { Article.create }
  let(:pancakes) { TagEchidna::Tag.create :name => 'pancakes' }

  it "stores new tags" do
    article.tags << pancakes

    article.tags.reload.should == [pancakes]
  end

  it "removes existing tags" do
    article.tags << pancakes

    article.tags.delete pancakes

    article.tags.reload.should == []
  end
end
```

### `spec/internal/app/models/article.rb`

```ruby
class Article < ActiveRecord::Base
  has_many_tags
end
```

### `spec/internal/config/database.yml`

```yaml
test:
  adapter:  sqlite3
  database: db/combustion_test.sqlite
```

### `spec/internal/db/schema.rb`

```ruby
ActiveRecord::Schema.define do
  create_table :articles, :force => true do |t|
    t.string  :title
    t.timestamps
  end
end
```

### `spec/internal/log/.gitignore`

```
*.log
```

We also want to add a Gemfile to our project, so Bundler can help use manage our project context:

### `Gemfile`

```
source 'https://rubygems.org'

gemspec
```

We do not want to commit our Gemfile.lock though, nor our test database:

### `.gitignore`

```
Gemfile.lock
spec/internal/db/*.sqlite
```

And then, we need to add a few dependencies to our gemspec. First up, ActiveRecord, as we're extending Rails app models. Our gem won't work without ActiveRecord, so it's a runtime dependency.

We also have three development dependencies - gems we need just so we can build our gem, but aren't required for it to work in Rails apps:

* Combustion, a library for testing Rails engines (which is what our gem will be),
* RSpec-Rails, the testing framework with some tweaks for Rails, and
* SQlite3, so our tests can persist data while they're running.

So, in our gemspec, add these lines at the bottom (but within the specification block):

```ruby
s.add_runtime_dependency     'activerecord', '>= 3.2.0'
s.add_development_dependency 'combustion',   '~> 0.4.0'
s.add_development_dependency 'rspec-rails',  '~> 2.13'
s.add_development_dependency 'sqlite3',      '~> 1.3.7'
```

With all of that set up, we can bundle:

{% terminal %}
$ bundle install
{% endterminal %}

Commit our changes:

{% terminal %}
$ git add .gitignore Gemfile spec tag_echidna.gemspec
$ git commit -m "Setting up a test suite"
{% endterminal %}

And run our tests:

{% terminal %}
$ rspec spec/acceptance
{% endterminal %}

And it will explode dramatically. But that's fine - if they passed, that'd be a bit worrying, because our gem still doesn't do anything.

## Step Five: The first implementation

Our tests are currently failing, because there's no `has_many_tags` method available to our models. If we're going to insert methods into all models, they need to be a part of ActiveRecord::Base - so let's write a module that can do this.

### `lib/tag_echidna/active_record.rb`

Don't forget: change TagEchidna references to match your gem name.

```ruby
module TagEchidna::ActiveRecord
  def self.included(base)
    base.extend TagEchidna::ActiveRecord::ClassMethods
  end

  module ClassMethods
    def has_many_tags
      has_many :taggings, :class_name => 'TagEchidna::Tagging',
        :as => :taggable, :dependent => :destroy
      has_many :tags,     :class_name => 'TagEchidna::Tag',
        :through => :taggings
    end
  end
end
```

We also need to make sure our gem includes this module into ActiveRecord::Base as Rails apps using our gem load. This is done in an Engine subclass:

### `lib/tag_echidna/engine.rb`

Again: change TagEchidna references to match your gem name.

```ruby
require 'rails/engine'

class TagEchidna::Engine < Rails::Engine
  engine_name :tag_echidna

  ActiveSupport.on_load :active_record do
    include TagEchidna::ActiveRecord
  end
end
```

And now we need something to load these two files - we already have the file, it just needs some code:

### `lib/tag_echidna.rb`

Tweak this to match your gem name and path:

```ruby
module TagEchidna
  #
end

require 'tag_echidna/active_record'
require 'tag_echidna/engine'
```

If you run our tests again, you'll see that the situation has improved - the tests actually run now, but they're red. Not quite there, but before we continue, let's commit our changes:

{% terminal %}
$ git add lib
$ git commit -m "Engine and ActiveRecord extensions"
{% endterminal %}

Our tests are complaining because we don't have tag models - so let's add those models in, plus a migration so they have tables as well.

### `app/models/tag_echidna/tag.rb`

We need a model for our tag objects:

```ruby
class TagEchidna::Tag < ActiveRecord::Base
  has_many :taggings, :class_name => 'TagEchidna::Tagging',
    :dependent => :destroy

  validates :name, :presence => true, :uniqueness => true
end
```

### `app/models/tag_echidna/tagging.rb`

And we need a model that links our tags to the model instances within Rails apps using our gem:

```ruby
class TagEchidna::Tagging < ActiveRecord::Base
  belongs_to :taggable, :polymorphic => true
  belongs_to :tag, :class_name => 'TagEchidna::Tag'

  validates :taggable, :presence => true
  validates :tag,      :presence => true
end
```

### `db/migrate/1_tag_tables.rb`

All migration files need to start with a unique number - they only need to be unique within our engine though, hence we can keep it nice and simple and use the number 1.

```ruby
class TagTables < ActiveRecord::Migration
  def up
    create_table :taggings do |t|
      t.integer :tag_id,        :null => false
      t.integer :taggable_id,   :null => false
      t.string  :taggable_type, :null => false
      t.timestamps
    end

    add_index :taggings, :tag_id
    add_index :taggings, [:taggable_type, :taggable_id]
    add_index :taggings, [:taggable_type, :taggable_id, :tag_id],
      :unique => true, :name => 'unique_taggings'

    create_table :tags do |t|
      t.string :name, :null => false
      t.timestamps
    end

    add_index :tags, :name, :unique => true
  end

  def down
    drop_table :tags
    drop_table :taggings
  end
end
```

And that's it - run the tests, and they'll be green. We now have a very simple tagging gem ready to go! Update our gem version in our gemspec (because we can't overwrite our existing 0.0.1 release):

```ruby
s.version     = '0.0.2'
```

Commit our additions:

{% terminal %}
$ git add app db tag_echidna.gemspec
$ git commit -m "Green tests"
{% endterminal %}

And then ship this release!

{% terminal %}
$ gem build tag_echidna.gemspec
$ gem push tag_echidna-0.0.2.gem
{% endterminal %}

## Step Six: Tags by name

You're making great progress - but it'd be really nice if developers using our gem didn't need to worry about our TagEchidna::Tag objects, and could just add and remove tags by their names instead.

So first, here's the failing tests:

### `spec/acceptance/tag_names_spec.rb`

```ruby
require 'spec_helper'

describe "Managing tags via names" do
  let(:article) { Article.create }

  it "returns tag names" do
    article.tags << TagEchidna::Tag.create(:name => 'melbourne')

    article.tag_names.to_a.should == ['melbourne']
  end

  it "adds tags via their names" do
    article.tag_names << 'melbourne'

    article.tags.collect(&:name).should == ['melbourne']
  end

  it "accepts a completely new set of tags" do
    article.tag_names = ['portland', 'oregon']

    article.tags.collect(&:name).should == ['portland', 'oregon']
  end

  it "enumerates through tag names" do
    article.tag_names = ['melbourne', 'victoria']
    names = []

    article.tag_names.each do |name|
      names << name
    end

    names.should == ['melbourne', 'victoria']
  end

  it "does not allow duplication of tags" do
    existing = Article.create
    existing.tags << TagEchidna::Tag.create(:name => 'portland')

    article.tag_names = ['portland']

    existing.tag_ids.should == article.tag_ids
  end

  it "appends tag names" do
    article.tag_names  = ['portland']
    article.tag_names += ['oregon', 'ruby']

    article.tags.collect(&:name).should == ['portland', 'oregon', 'ruby']
  end

  it "removes a single tag name" do
    article.tag_names = ['portland', 'oregon']
    article.tag_names.delete 'oregon'

    article.tags.collect(&:name).should == ['portland']
  end

  it "removes tag names" do
    article.tag_names  = ['portland', 'oregon', 'ruby']
    article.tag_names -= ['oregon', 'ruby']

    article.tags.collect(&:name).should == ['portland']
  end
end
```

You can run the tests and confirm that yes, it's not all green anymore. Let's step through getting the tag names implemented, bit by bit.

First, we need to ensure that each model has a method called `tag_names`. This is easy enough, but we also know whatever gets returned from that method has further methods called on it. We really don't want to add any more to the model than necessary, and this is quite a focused piece of the gem, so it can be an object instead, called TagNames.

### `lib/tag_echidna/active_record.rb`

Add this method to our TagEchidna::ActiveRecord module (but not inside the ClassMethods module):

```ruby
def tag_names
  @tag_names ||= TagEchidna::TagNames.new self
end
```

### `lib/tag_echidna/tag_names.rb`

```ruby
class TagEchidna::TagNames
  def initialize(taggable)
    @taggable = taggable
  end

  def to_a
    taggable.tags.collect &:name
  end

  private

  attr_reader :taggable
end
```

We also need to make sure that this new class is loaded when the rest of the gem is - so add the following line to the end of `lib/tag_echidna.rb`:

```ruby
require 'tag_echidna/tag_names'
```

That fixes one test... there's still seven that are unhappy though. If we want to append a single tag name, we need the `<<` method. So put this in the TagNames class, below the to_a definition.

```ruby
def <<(name)
  tag = TagEchidna::Tag.where(:name => name).first ||
        TagEchidna::Tag.create(:name => name)

  taggable.tags << tag
end
```

We also want to be able to provide a completely new set of tags - so here's that writer method for our `TagEchidna::ActiveRecord` module:

```ruby
def tag_names=(names)
  if names.is_a?(TagEchidna::TagNames)
    @tag_names = names
  else
    @tag_names = TagEchidna::TagNames.new_with_names self, names
  end
end
```

That then relies on a new class method and a new instance method in our `TagNames` class:

```ruby
def self.new_with_names(taggable, names)
  tag_names = new(taggable)
  tag_names.clear
  names.each { |name| tag_names << name }
  tag_names
end

def clear
  taggable.tags.clear
end
```

Now we only have four failing tests - that's some good progress indeed! You're not going to stop here though, are you?

Let's add the code that lets us enumerate through each tag name using `each` (this goes in `TagNames`):

```ruby
def each(&block)
  to_a.each &block
end
```

It'd be nice if we added the other common enumeration methods too - which just takes a single line:

```ruby
include Enumerable
```

Next, the `delete` method (a pair with our existing `<<` method):

```ruby
def delete(name)
  taggable.tags.delete TagEchidna::Tag.where(:name => name).first
end
```

And finally our bulk actions - `+` and `-`:

```ruby
def +(array)
  array.each { |name| self.<< name }
  self
end

def -(array)
  array.each { |name| self.delete name }
  self
end
```

Give those tests a spin - they should be green again!

{% terminal %}
$ rspec spec/acceptance
{% endterminal %}

Time to prepare another release. This one's got some clear functionality improvements, but doesn't break anything, so it can be a minor release:

```ruby
s.version     = '0.1.0'
```

Commit our additions:

{% terminal %}
$ git add lib spec tag_echidna.gemspec
$ git commit -m "Tag names"
{% endterminal %}

And then ship this release!

{% terminal %}
$ gem build tag_echidna.gemspec
$ gem push tag_echidna-0.1.0.gem
{% endterminal %}

Congratulations - you have a working tag gem with a green test suite. Perhaps you should create a new Rails app and put this in to give it a spin? Once the gem's in your app's Gemfile, you need to run a rake task to copy migrations over:

{% terminal %}
$ rake tag_echidna:install:migrations
{% endterminal %}

## Extra Credit

Perhaps it's time to write up a README so other people can find out how to use it? Adding a LICENSE file is also wise - for examples on both, have a browse of some other gems on GitHub. Here's a few to get you started:

* https://github.com/pat/combustion
* https://github.com/mperham/sidekiq
* https://github.com/roidrage/lograge
