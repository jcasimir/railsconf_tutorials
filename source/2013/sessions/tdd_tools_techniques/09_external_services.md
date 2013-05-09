---
layout: page
title: Working with 3rd party external services
section: TDD Tools, Techniques, and Discipline
sidebar: false
---

Tests should run in isolation.

When integrating with external services we want to make sure our test suite
isn't actually hitting any 3rd party service.

Requests to external services can cause issues like:

* Dramatically slower tests
* Hitting API rate limits on 3rd party sites (e.g. Twitter)
* 3rd party doesn't have a testing or sandbox server
* Service may not exist
* Can't run tests without a network connection
* Tests can fail if service responses change

## Webmock

We will use [webmock][1], a gem which helps in mocking out external web
requests. In this example we'll search the [Twitter API for RailsConf][4].

First, let's make sure out test suite doesn't make external requests by setting
it in the `spec_helper.rb`:

```ruby
# spec/spec_helper.rb
    ...
    WebMock.disable_net_connect!(allow_localhost: true)
    ...
```

Now if your test suite tries to make any external requests it will raise an
exception and break the build:

```ruby
# spec/features/external_request_spec.rb
feature 'External request' do
  it 'queries Twitter search endpoint' do
    uri = URI('http://search.twitter.com/search.json?q=railsconf&rpp=1')

    response = Net::HTTP.get(uri)

    expect(response).to be_an_instance_of(String)
  end
end
```

{% terminal %}
$ rspec spec/features/external_request_spec.rb
F

Failures:

  1) External request queries Twitter search endpoint
     Failure/Error: response = Net::HTTP.get(uri)
     WebMock::NetConnectNotAllowedError:
       Real HTTP connections are disabled. Unregistered request: GET http://search.twitter.com/search.json?q=railsconf&rpp=1 with headers {'Accept'=>'*/*', 'User-Agent'=>'Ruby'}

       You can stub this request with the following snippet:

       stub_request(:get, "http://search.twitter.com/search.json?q=railsconf&rpp=1").
         with(headers: {'Accept'=>'*/*', 'User-Agent'=>'Ruby'}).
         to_return(status: 200, body: "", headers: {})

       ============================================================
     # ./spec/features/external_request_spec.rb:7:in `block (2 levels) in <top (required)>'
{% endterminal %}

We can fix this by stubbing any external requests with webmock.

```ruby
# spec/spec_helper.rb
RSpec.configure do |config|
  config.before(:each) do
    stub_request(:get, /search.twitter.com/).
       with(headers: {'Accept'=>'*/*', 'User-Agent'=>'Ruby'}).
       to_return(status: 200, body: "fake response", headers: {})
  end
end
```

Run the test again and now it will pass.

{% terminal %}
$ rspec spec/features/external_request_spec.rb
.

Finished in 0.01116 seconds
1 example, 0 failures
{% endterminal %}

## VCR

Another approach for preventing external requests is to record a live
interaction and 'replay' it back during tests.  The [VCR][2] gem has a concept
of `cassettes` which will record your test suites outgoing HTTP requests and
then replay them for future test runs.

Considerations when using VCR:

* Needs the external service to be available for first test run.
* Must clarify how cassettes are shared with other developers (check in to git
  repo?)

## Create a Fake (Hello Sinatra!)

When your application depends heavily on a third party service, consider
building a fake service inside your application with [Sinatra][3]. This will let
us test in total isolation and control the responses to our test suite.

First we use Webmock to route all requests to our Sinatra application,
`FakeTwitter`.
```ruby
RSpec.configure do |config|
  config.before(:each) do
    stub_request(:any, /search.twitter.com/).to_rack(FakeTwitter)
  end
end
```

Now let's create the `FakeTwitter` application using Sinatra.

```ruby
# spec/support/fake_twitter.rb
require 'sinatra/base'

class FakeTwitter < Sinatra::Base
  get '/search.json' do
    json_response 200, 'twitter_search.json'
  end

  private

  def json_response(response_code, file_name)
    content_type :json
    status response_code
    File.open(File.dirname(__FILE__) + '/fixtures/' + file_name, 'rb').read
  end
end
```

Download the [Twitter search results][4] and store in a local file.

```ruby
# spec/support/fixtures/twitter_search.json
{
  completed_in: 0.011,
  max_id: 327949397356847100,
  max_id_str: "327949397356847106",
  next_page: "?page=2&max_id=327949397356847106&q=railsconf&rpp=1",
  page: 1,
  query: "railsconf",
  refresh_url: "?since_id=327949397356847106&q=railsconf",
  results: [
    {
    created_at: "Sat, 27 Apr 2013 00:56:43 +0000",
    from_user: "amateurhuman",
    from_user_id: 41456770,
    from_user_id_str: "41456770",
    from_user_name: "Chris Kelly",
    geo: null,
    id: 327949397356847100,
    id_str: "327949397356847106",
    iso_language_code: "en",
    metadata: {
    result_type: "recent"
    },
    profile_image_url: "http://a0.twimg.com/profile_images/3367316797/d007420a547564a586957a252c9a1fe6_normal.jpeg",
    profile_image_url_https: "https://si0.twimg.com/profile_images/3367316797/d007420a547564a586957a252c9a1fe6_normal.jpeg",
    source: "&lt;a href=&quot;http://tapbots.com/tweetbot&quot;&gt;Tweetbot for iOS&lt;/a&gt;",
    text: "RT @newrelic: Wish you could make it to #RailsConf? We're streaming the keynotes to our #SanFrancisco office. Register here: http://t.co/4jYIojxJiN"
    }
  ],
  results_per_page: 1,
  since_id: 0,
  since_id_str: "0"
}
```

Update the test to verify the fake response is being returned.

```ruby
feature 'External request' do
  it 'queries Twitter search endpoint' do
    uri = URI('http://search.twitter.com/search.json?q=railsconf&rpp=1')

    response = JSON.load(Net::HTTP.get(uri))

    expect(response['query']).to eq 'railsconf'
  end
end
```

Run the specs.

{% terminal %}
$ rpec spec/features/external_request_spec.rb
.

Finished in 0.04713 seconds
1 example, 0 failures
{% endterminal %}

There are some downsides to using a fake:

* Maintaining a fake version of the service takes time
* Your fake can get out of sync with the service

[1]: https://github.com/bblimke/webmock
[2]: https://github.com/vcr/vcr
[3]: http://www.sinatrarb.com/
[4]: http://search.twitter.com/search.json?q=railsconf&rpp=1
