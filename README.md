railsapis-codeschool
====================


##Level 1: REST, Routes,  Constraints and namespaces

##### Using Constraints to enforce subdomain

Keeping our API under its own subdomain allows load balancing traffic at the DNS Level.

```ruby
resources :episodes
resources :zombies, constraints: {subdomain: 'api'}
resources :humans, constraints: {subdomain: 'api'}
```

Or

```ruby
resources :episodes

constraints :subdomain 'api' do
  resources :zombies
  resources :humans
end
```



##### Using namespaces to keep controllers organized 

config/routes.rb
```ruby
constraints subdomain: 'api' do
  namespace :api do
    resources :zombies
  end
end
```

app/controllers/api/zombies_controller.rb

- web API controllers are part of the API module
```ruby
  module Api do
    class ZombiesController < ApplicationController
      
    end
  end
```

and web site controllers remain on top-level namespace, for example:
app/controllers/pages_controller.rb
```ruby
  class PagesController < ApplicationController
  end
```

##### We can use path on routes to remove the duplicate name api from the route. For example:

config/routes/rb
```ruby
  constraint :subdomain 'api' do
    namespace :api, path: '/' do 
      resources :zombies
    end
  end
```

```ruby
SurvivingRails::Application.routes.draw do
  namespace :api,path: '/', constraints: { subdomain: 'api'} do
		resources :zombies
		resources :humans    
  end
  resources :announcements
end
```


and now we can only use the subdomain, like http://api.mysite.com/zombies


##### Using a shorter syntax for constraints and namespaces

config/routes.rb
```ruby
  constraints subdomain: 'api'do
    namespace :api, path: '/' do
      resources :zombies
      resources :humans
    end
  end
```


```ruby
namespaces :api, path: '/', constraints: { subdomain: 'api'} do
  resources :zombies
  resources :humans
end
```

##### Using with_options

The with_options method is an elegant way to factor duplication out of options passed to a series of method calls.

```ruby
SurvivingRails::Application.routes.draw do
  resources :zombies, only: :index
  resources :humans, only: :index
  resources :medical_kits, only: :index
  resources :broadcasts, only: :index
end
```

```ruby
SurvivingRails::Application.routes.draw do
  with_options only: :index do |list_only|
    list_only.resources :zombies
    list_only.resources :humans
    list_only.resources :medical_kits
    list_only.resources :broadcasts
  end
end
````


##Level 2: Resources and GET

##### It's all about the resources

Any information that can be named can be a resource
Some examples of a resource:
 - A music playlist
 - A song
 - The leader of the Zombie horde
 - Survivors
 - Remaining medical kits

"A resource is a conceptual mapping to a set of entities, not the entity that corrresponds to the mapping at any  particular point of time" - Steve Klabnik, Designing Hypermeda APIs


##### Understanding the get method

The GET method is used to read information identified by a giben URI

Important characteristics:
- Safe: it should not take any action other than retrieval
- Idempotent: sequential GET requests to the same URI should not generate side-effects




A simple listening test to verify if the route is ok

```ruby
class ListingHumansTest < ActionDispatch::IntegrationTest
	setup { host! 'api.example.com' }

  test 'returns a list of humans' do
    get '/humans'
    assert_equal 200, response.status
    refute_empty response.body
  end
end
```


Our simple controller rendering json and listing all objects
```ruby
module API
  class HumansController < ApplicationController
    def index
      humans = Human.all
      render json: humans, status: :ok
    end
  end
end
```


Testing a simple query and verify if the response is right
```ruby
class ListingHumansTest < ActionDispatch::IntegrationTest
  setup { host! 'api.example.com'}

  test 'returns a list of humans by brain type' do
    allan = Human.create(name: 'Allan', brain_type: 'large')
    jonh = Human.create(name: 'John', brain_type: 'small')
    
    get '/humans?brain_type=small'
    assert_equal 200, response.status
    
    zombies = JSON.parse(response.body, symbolize_names:true)
    names = zombies.collect {|z| z[:name] }
    assert_includes names, 'John'
    refute_includes names, 'Allan'
  end
end
```

Our controller will looks like this:
```ruby
module API
  class HumansController < ApplicationController
    def index
      humans = Human.all
      if params[:brain_type]
        humans = Human.where(brain_type: params[:brain_type])
      end
      render json: humans, status: :ok
    end
  end
end
```

Let's gonna use a method helper to parse JSON:
```ruby
ENV['RAILS_ENV'] = 'test'
require File.expand_path('../../config/environment', __FILE__)
require 'rails/test_help'

class ActiveSupport::TestCase
  ActiveRecord::Migration.check_pending!
  fixtures :all

  def json(body)
    JSON.parse(body, symbolize_names: true)
  end
end
```

And we can use now on our follow test:
id
```ruby
class ListingHumansTest < ActionDispatch::IntegrationTest
  setup { host! 'api.example.com' }

  test 'returns human by id' do
    human = Human.create!(name: 'Ash')
    
    get "/humans/#{human.id}"
    assert_equal 200, response.status
    
    zombie_response = json(response.body)
    assert_equal human.name, zombie_response[:name]
    
  end
end
```

We can also use curl to check our response.


##Level 3: Content Negotiation

##### Different clients need different formats

Web APIs need to cater to differnet types of clients.

##### Setting the response format from the URI

Rails allows switching formats by adding an extension to the URI

For example:

config/routes.rb
```ruby
	resources :zombies
``` 

It means that we can use: http://mywebapplication.com/zombies.JSON or zombies.XML

This is a nicety from Rails and it is NOT a standard.


##### Using the accept header to request a media type

Media types(used to be called Mime Types) specify the scheme for resource representations


Testing our content type
```ruby
class ListingZombiesTest  < ActionDispatch::IntegrationTest
	test 'returns zombies in JSON' do
		get '/zombies', {}, {'Accept' => Mime::JSON}
		assert_equal 200, response.status
		assert_equal Mime::JSON, response.content_type
	end
end
```

##### Using respond_to to serve JSON

```ruby
class ZombiesController < ApplicationController
	def index
		zombies = Zombie.all
		respond_to do |format|
			format.json {render json: zombies, status: :ok}
		end
	end
end
```

##### Listing all media types from Rails

The following command will list all supported media types
```ruby
Mime::SET
```

##### Using curl to get the response and test
```shell
$ curl -IH "Accept: application/json" localhost:3000/zombies
```

```shell
$ curl -IH "Accept: application/json" localhost:3000/zombies
```








