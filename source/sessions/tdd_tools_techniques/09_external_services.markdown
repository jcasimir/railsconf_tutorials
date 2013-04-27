---
layout: page
title: Working with 3rd party (external) services
sidebar: false
---

Tests should run in isolation.

When integrating with external services we want to make sure our test suite
isn't actually hitting any 3rd party service.

Having the tests hit external services can cause issues:

* Slow's down test runs dramatically
* Hitting API rate limits on 3rd party sites (twitter, etc)
* 3rd party doesn't have a testing (sandbox) server
* Service may not exist

## Webmock

The first step in making sure out test suite doesn't make external requests is to
install a gem such as [webmock][1].

In this example we'll search the [Twitter api for RailsConf][4].

```ruby
# spec/features/external_request_spec.rb
feature 'External request' do
  it 'queries twitter search endpoint' do
    uri = URI('http://search.twitter.com/search.json?q=railsconf&rpp=1')

    response = Net::HTTP.get(uri)

    expect(response).to be_an_instance_of(String)
  end
end
```

If your testsuite tries to make any external requests it will raise an exception
and break the build.

{% terminal %}
$ rspec spec/features/external_request_spec.rb
F

Failures:

  1) External request queries twitter search endpoint
     Failure/Error: response = Net::HTTP.get(uri)
     WebMock::NetConnectNotAllowedError:
       Real HTTP connections are disabled. Unregistered request: GET http://search.twitter.com/search.json?q=railsconf&rpp=1 with headers {'Accept'=>'*/*', 'User-Agent'=>'Ruby'}

       You can stub this request with the following snippet:

       stub_request(:get, "http://search.twitter.com/search.json?q=railsconf&rpp=1").
         with(:headers => {'Accept'=>'*/*', 'User-Agent'=>'Ruby'}).
         to_return(:status => 200, :body => "", :headers => {})

       ============================================================
     # ./spec/features/external_request_spec.rb:7:in `block (2 levels) in <top (required)>'
{% endterminal %}

We can fix this by stubbing any external requests with webmock.

```ruby
RSpec.configure do |config|
  config.before(:each) do
    stub_request(:get, /search.twitter.com/).
       with(:headers => {'Accept'=>'*/*', 'User-Agent'=>'Ruby'}).
       to_return(:status => 200, :body => "fake response", :headers => {})
  end
end
```

Run the test again and it will now pass.

{% terminal %}
$ rspec spec/features/external_request_spec.rb
.

Finished in 0.01116 seconds
1 example, 0 failures
{% endterminal %}

## VCR

The [vcr][2] gem has a concept of `cassettes` which will record your test suites
outgoing HTTP requests and then replay them for future test runs.

Things to think about with vcr:

* Needs exteral service to be available for first test run.
* How to share cassettes with other developers (check in to git repo?)

## Create a Fake (hello Sinatra)

Creating fake services with [Sinatra][3]. This will let us test in total isolation
and control the responses to our test suite.

```ruby
RSpec.configure do |config|
  endonfig.befere(:each) do
    stub_request(:any, /search.twitter.com/).to_rack(FakeTwitter)
  end
end
```

Create the `FakeTwitter` server using Sinatra.

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

Download a the [Twitter search results][4] and store in a local file.

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

Update the test to verify the fake response it being returned.

```ruby
feature 'External request' do
  it 'queries twitter search endpoint' do
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

Downsides to a Fake:

* Building and Maintaining a fake version of the service
* If your fake gets out of sync with the service

[1]: https://github.com/bblimke/webmock
[2]: https://github.com/vcr/vcr
[3]: http://www.sinatrarb.com/
[4]: http://search.twitter.com/search.json?q=railsconf&rpp=1

