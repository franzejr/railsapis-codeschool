railsapis-codeschool
====================


#Level 1: REST, Routes,  Constraints and namespaces

## Using Constraints to enforce subdomain

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



### Using namespaces to keep controllers organized 

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

### We can use path on routes to remove the duplicate name api from the route. For example:

config/routes/rb
```ruby
  constraint :subdomain 'api' do
    namespace :api, path: '/' do 
      resources :zombies
    end
  end
```

and now we can only use the subdomain, like http://api.mysite.com/zombies


### Using a shorter syntax for constraints and namespaces

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



