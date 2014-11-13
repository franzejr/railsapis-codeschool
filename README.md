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
- 




Listening

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


Our controller rendering json
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


Testing
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
    assert_includes names, 'Jonh'
    refute_includes names, 'Allan'
  end
end
```


 






