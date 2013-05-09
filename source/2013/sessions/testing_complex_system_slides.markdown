---
layout: page
title: "Testing Complex Systems"
date: 2013-05-09 09:35
comments: true
sharing: true
footer: true
---

### Noel Rappin
### Table XI

## We are going to talk about two hard problems

## Creating Data for tests

## Limiting tests to a particular part of the code

## Creating data is hard when objects are entangled

## Limiting tests is hard when objects are entangled

## Idea: Let's not have objects be entangled

## We're going to talk about ONE hard problem

## Driving modular design with tests

## Things to note:

## It always takes longer to talk about TDD than to do it

## This example is, you know, an example.

## Testing is part of the complete professional breakfast

## So... We have a website

## There isn't much there

## It's got some basic data
* Trips
* A trip has multiple Hotels
* A trip has multiple Activities
* Users

## I want users to be able to purchase a trip

## A purchase unifies
* A User
* A Trip
* A Hotel
* Days being stayed
* Zero or more activities

## We're not going to worry about the view logic

## Where to start?

## Three options:
* Dive Right In
* Top-Down (Outside-in)
* Bottom-Up

## What does it need to do?
* Calculate price
* Calculate tax
* Validate availability
* Process payment

## What do I test first?
* Initialized state
* Basic happy path
* What makes this break?
* Special cases

## First test

spec/models/purchase_spec.rb

```ruby
require 'spec_helper'

describe Purchase do
  describe "pricing" do
    it "can calculate price given just a trip" do
      trip = Trip.create!(price: 200)
      purchase = Purchase.new(trip)
      expect(purchase.pre_tax_total).to eq(200)
    end
  end
end
```

## Make it pass

app/models/purchase.rb

```ruby
class Purchase

  attr_accessor :trip

  def initialize(trip)
    @trip = trip
  end

  def pre_tax_total
    trip.price
  end

end
```

## That's great, but...

## Later...

app/models/trip.rb

```ruby
class Trip < ActiveRecord::Base

  attr_accessible :activity, :description,
      :end_date, :image_name, :location,
      :name, :price, :start_date, :tag_line

  validates_presence_of :description, :start_date,
      :end_date, :price, :tag_line

end
```

## Meaning...

## Lots of data

spec/models/purchase_spec.rb

```ruby
require 'spec_helper'
describe Purchase do
  describe "pricing" do
    it "can calculate price given a trip and a hotel" do
      trip = Trip.create!(price: 200,
          description: "See Noel's Birthday",
          start_date: Date.parse("Jan 22, 1971"),
          end_date: Date.parse("Jan 24, 1971"),
          tag_line: "Why would you want to do this?")
      purchase = Purchase.new(trip)
      expect(purchase.pre_tax_total).to eq(200)
    end
  end
end
```

## Why is this bad?
* Test is hard to read
* Test is fragile
* Too much setup means too little testing

## Remedies
* Don't save to database
* Fixtures
* Factories
* Stubs

## Don't save to database
* Not a bad thought
* Working with invalid objects can be a problem
* Rails associations don't always work

## Fixtures
```yaml
my_birthday:
  activity: Watching Things
  description: The day of Noel's Birth
  end_date: "Jan 24, 1971"
  image_name: baby.png
  location: "Illinois"
  name: "Watch Noel's Birthday"
  price: 200
  start_date: "Jan 22, 1971"
  tag_line: "Why"
```

## Then

spec/models/purchase_spec.rb

```ruby
require 'spec_helper'

describe Purchase do
  describe "pricing" do
    it "can calculate price given a trip and a hotel" do
      trip = trips(:my_birthday)
      purchase = Purchase.new(trip)
      expect(purchase.pre_tax_total).to eq(200)
    end
  end
end
```

## Yay Fixtures
* Fast
* Relatively easy to set up with associations
* Concise in code

## Boo Fixtures
* Opaque
* Fragile
* Global

## Factories
```ruby
FactoryGirl.define do
  factory :trip do
    name "The day of Noel's Birth"
    description "Watch Noel's Birthday"
    start_date "1971-01-22"
    end_date "1971-01-23"
    image_name "baby.png"
    tag_line "Why?"
    price 100
    location "Illinois"
    activity "Watching Things"
  end
end
```

## And then
```ruby
require 'spec_helper'

describe Purchase do
  describe "pricing" do
    it "can calculate price given a trip and a hotel" do
      trip = FactoryGirl.create(:trip, price: 200)
      purchase = Purchase.new(trip)
      expect(purchase.pre_tax_total).to eq(200)
    end
  end
end
```

## Yay Factories
* Easy to build
* Allow for flexibility in test
* focus on important attributes

## Boo Factories
* Too easy to build
* Can be slow
* Easy to build long slow attribute trees

## Stubs
```ruby
require 'spec_helper'

describe Purchase do
  describe "pricing" do
    it "can calculate price given a trip and a hotel" do
      trip = stub(price: 200)
      purchase = Purchase.new(trip)
      expect(purchase.pre_tax_total).to eq(200)
    end
  end
end
```

## Stubs?
* A stub replaces all or part of an object
* Intercepts messages and replaces them with canned values

## Mocks vs. Stubs vs. Spies
* Stub: replaces method definition with canned value
* Mock: like a stub, but asserts that the method is called
* Spy: like a stub, but allows for later assertions of behavior

## Yay stubs
* Very fast
* Robust against some implementation changes
* Allows you to test behavior without testing state

## Boo Stubs
* Not at all robust against other implementation changes
* Implementation can be opaque
* Very sensitive to design

## Remedies

#### Don't save to database
Fast, incomplete, bad with association

#### Fixtures
Fast, global, brittle

#### Factories
Slower, focused, readable

#### Stubs
Fast, behavior, brittle

## Moving On

spec/models/purchase_spec.rb

```ruby
it "can calculate price given a trip and a hotel stay" do
  trip = FactoryGirl.create(:trip, price: 200)
  hotel = FactoryGirl.create(:hotel, price: 50)
  purchase = Purchase.new(trip, hotel, 3)
  expect(purchase.pre_tax_total).to eq(350)
end
```

## Actually...

spec/models/hotel_stay_spec.rb

```ruby
describe HotelStay do
  describe "price" do
    it "calculates its own price" do
      hotel = FactoryGirl.create(:hotel, price: 50)
      stay = HotelStay.new(hotel, 3)
      expect(stay.pre_tax_total).to eq(150)
    end
  end
end
```

## Which passes with
```ruby
class HotelStay

  attr_accessor :hotel, :length_of_stay

  def initialize(hotel, length_of_stay = 0)
    @hotel = hotel
    @length_of_stay = length_of_stay
  end

  def pre_tax_total
    hotel.price * length_of_stay
  end

end
```

## Now back to the purchase

spec/models/purchase_spec.rb

```ruby
it "can calculate price given a trip and a hotel stay" do
  trip = FactoryGirl.create(:trip, price: 200)
  hotel_stay = stub(pre_tax_total: 150)
  purchase = Purchase.new(trip, hotel_stay)
  expect(purchase.pre_tax_total).to eq(350)
end
```

## Purchase
```ruby
class Purchase

  attr_accessor :trip, :hotel_stay

  def initialize(trip, hotel_stay = nil)
    @trip = trip
    @hotel_stay = hotel_stay
  end

  def pre_tax_total
    result = trip.price
    result += hotel_stay.pre_tax_total if hotel_stay
    result
  end

end
```

## Now with activities
```ruby
it "can calculate a price given some activities " do
  cheap_activity = Activity.new(price: 50)
  expensive_activity = Activity.new(price: 1000)
  purchase = Purchase.new(
      trip, nil, [cheap_activity, expensive_activity])
  expect(purchase.pre_tax_total).to eq(200 + 50 + 1000)
end
```

## Alternative API Options
```ruby
purchase = Purchase.new(trip, nil,
    cheap_activity, expensive_activity)
```
```ruby
schedule = new Schedule(cheap_activity, expensive_activity)
purchase = Purchase.new(trip, nil, schedule)
```

## Passing test
```ruby
def pre_tax_total
  result = trip.price
  result += hotel_stay.pre_tax_total if hotel_stay
  result += activities.map(&:price).sum
  result
end
```

## Your turn

## Taxes and Fees

## Requirements:
* Trip tax: 5% on any trip more than 100 years from user's start point
* Hotel tax: 7% on any hotel night later than 1900
* Activity tax: 10%

## Considerations
* new data needed -- user's start year, hotel location
* assume this data is volatile -- how can you make tests robust?

## First Pass test
```ruby
require 'spec_helper'
describe Trip do
  describe "taxes" do
    let(:trip) { FactoryGirl.build(:trip,
        :start_date => "Jan 22, 1971", :price => 100) }
    let(:user) { FactoryGirl.build(:user) }
    it "knows that there is no tax on nearby trips" do
      user.home_year = 1971
      expect(trip).not_to be_taxable_for_user(user)
    end
    it "knows there is a tax on far away trips" do
      user.home_year = 2100
      expect(trip).to be_taxable_for_user(user)
    end
  end
end
```

## Issues
* The years are magic
* The test is volatile against changes in data

## Another possibility
```ruby
describe TripTaxCalculator do
  it "can determine if a trip is eligible for taxes" do
    tc = TripTaxCalculator.new(100, 0.05)
    trip = stub(:start_date => Date.parse("Jan 22, 1971"))
    user = stub(:home_year => 1971)
    expect(tc.tax_percentage_for(trip, user)).to eq(0)
  end
end
```
```ruby
it "is using the right tax calculator" do
  trip = Trip.new
  expect(trip.tax_calculator.year_distance).to eq(100)
end
```

## Moving on

## Working with external services

## The General plan
* Wrap the 3rd party server
* Test against your wrapper
* Test the wrapper against the API

## Testing for availability

## Requirements
* Assume a third-party API with a Ruby wrapper
* It checks if a given hotel is available on a given night

## Here's the "wrapper"
```ruby
class HotelApi

  def self.available?(hotel_api_id, date_string)
    sleep(rand * 5)
    result = rand
    if result < 0.95
      true
    elsif result < 0.99
      false
    else
      sleep(rand * 10)
      raise "Ooops"
    end
  end

end
```

## We want to use this to check availability of a hotel stay

## First pass at trying a test
```ruby
describe "availability" do
  it "should description" do
    hotel = FactoryGirl.build(:hotel)
    stay = HotelStay.new(
        hotel, length_of_stay: 3, start_of_stay: "Jan 21, 1971")
    stay.availablity_checker.should_receive(:available_on?)
        .with(Date.parse("Jan 21, 1971")).and_return(true)
    stay.availablity_checker.should_receive(:available_on?)
        .with(Date.parse("Jan 22, 1971")).and_return(true)
    stay.availablity_checker.should_receive(:available_on?)
        .with(Date.parse("Jan 23, 1971")).and_return(true)
    expect(stay.available?).to be_true
  end
end
```

## Make it pass
```ruby
def available?
  (start_of_stay ... start_of_stay + length_of_stay).to_a.all? do |date|
    availablity_checker.available_on?(date)
  end
end
```

## we can do better
```ruby
describe "availability" do
  it "should description" do
    hotel = FactoryGirl.build(:hotel)
    stay = HotelStay.new(
        hotel, length_of_stay: 3, start_of_stay: "Jan 21, 1971")
    stay.availablity_checker.should_receive(:available_for_range)
        .with(hotel.api_id, Date.parse("Jan 21, 1971"), 3)
        .and_return(true)
    expect(stay.available?).to be_true
  end
end
```

## which gets us to
```ruby
def initialize(hotel, length_of_stay: 0, start_of_stay: nil)
  @hotel = hotel
  @length_of_stay = length_of_stay
  @start_of_stay = Date.parse(start_of_stay) if start_of_stay
  @availablity_checker = HotelAvailabilityChecker.new(hotel)
end

def pre_tax_total
  hotel.price * length_of_stay
end

def available?
  availablity_checker.available_for_range(
      start_of_stay, length_of_stay)
end
```

## Let's test the wrapper
```ruby
describe HotelAvailabilityChecker do
  describe "availability" do
    it "translates a range" do
      hotel = stub(api_id: "abc123")
      checker = HotelAvailabilityChecker.new(hotel)
      HotelApi.should_receive(:available?)
          .with("abc123", "1971-01-21").and_return(true)
      HotelApi.should_receive(:available?)
          .with("abc123", "1971-01-22").and_return(true)
      HotelApi.should_receive(:available?)
          .with("abc123", "1971-01-23").and_return(true)
      checker.available_for_range(Date.parse("Jan 21, 1971"), 3)
    end
  end
end
```

## And write the wrapper
```ruby
class HotelAvailabilityChecker
  attr_accessor :hotel

  def initialize(hotel)
    @hotel = hotel
  end

  def available_for_range(date, length_of_stay)
    (date ... date + length_of_stay).to_a.all? do |date|
      HotelApi.available?(hotel.api_id, date.to_s)
    end
  end
end
```

## Your turn

## Payment

## Requirements
* Communicate with PaymentApi.payment
* Which takes a credit card number, the user's name, a zip code, and an amount

## Considerations
* The API seems clunky, the wrapper can have better input and output
* Error handling?

## Conclusions

## Encapsulation reduces cognitive load

## Tests can drive design

## If the test seems hard to set up, the code might be at fault

## Refactor






