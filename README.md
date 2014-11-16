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

##### Testing with language set to english

Use the Accept-Language request header for language negotiation.


##### Setting the language for the response

Use the request.headers to access request headers


app/controllers/applicaition_controller.rb
```ruby
class ApplicationController < ActionController::Base
protect_from_forgery with: :exception
before_action :set_locale

protected
	def set_locale
		I18n.locale = request.headers['Accept-Language']
	end
end
```
Use the I18n.locale method to set application wide locale

##### Using the http_accept_language gem

Use the http_accept_language gem for a more robust support for locales.


- Sanitizes the list of preferred languages
- Sorts list of preferred languages
- Finds best fit if multiple languages supported

aap/controllers/application_controller.rb
```ruby
class ApplicationController < ActionController::Base
	protect_from_forgery with: :exception
	before_action :set_locale
	
	protected
	def set_locale
		locales = I18n.available_locales
		I18n.locale = http_accept_language.compatible_language_from(locales)
	end
end
```
The method compatible_language_from checks header and returns the first language compatible with the available locales.



##### Using jbuilder to return localized json

Jbuilder provides a DSL for generating JSON

```ruby
class HumansController < ApplicationController
  def index
    @humans = Human.all
    respond_to do |format|
      format.json
    end
  end
end
```



```ruby
json.array(@humans) do |human|
  json.extract! human, :id, :name, :brain_type
  json.message I18n.t('human_message',name: human.name)
end
```

## Level 4: POST, PUT, PATCH and DELETE

##### The POST Method

The POST method is used to create new resources

POST is neither safe or idempotent.

##### Responding successfully to post methods

A couple of things are expected from a successful POST request:

- The status code for the response should be 201 - Created.
- The response body should contain a representation of the new resource.
- The location header should be set with the location of the new resource

The 201 code means the request has been fulfilled and resulted in a new resource being created.

##### Integration testing the post method

test/integration/creating_episodes_test.rb
```ruby
class CreatingEpisodesTest < ActionDispatch::IntegrationTest
	test 'creates episodes' do
		post '/episodes/',
		{episode:
		   {title:'Bananas', description:'Learn about bananass.'}
		}.to_json,
		{'Accept' => Mime::JSON, 'Content-Type' => Mime::JSON.to_s }
		
		assert_equal 200, response.status
		assert_equal Mime::JSON, response.content_type
		
		episode = json(response.body)
		assert_equal episode_url(episode[:id]), response.location
	end
end
```


##### Posting data with curl

curl can help detect errors not caught by tests.

the -X option specifies the method
```shell
$ curl -i -X POST -d 'episode[title]=ZombieApocalypseNow' \
   http://localhost:3000/episodes
```
Use -d to send data on the request.

```shell
HTTP/1.1 422 Unprocessable Entity Content-Type: text/html; charset=utf-8
```
422 - Unprocessable Entity means the client submitted request was well-formed but semantically invalid.

##### Forgery protection is disabeld on test

Rails checks for an authenticity token on POST, PUT/PATCH and DELETE.

app/controllers/aplication_controller.rb
```ruby
class ApplicationController < ActionController::Base
	#Prevent CSRF attacks by raising an exception
	#For APIs, you may want to use :null_session instead
	protect_from_forgery with: :excpetion
end
```

```ruby
class ApplicationController < ActionController::Base
  protect_from_forgery with: :null_session
end
```

Defaults to disable on test environment.
config/environments/test.rb
```ruby
# Disable request forgery protection in test environment.
config.action_controller.allow_forgery_protection = false
```
the reason why CSRF error isn't raised during tests

##### Using empty sessions on API requests

API calls should be stateless.

app/controllers/application_controller.rb
```ruby
class ApplicationController < ActionController::Base
   # Prevent CSRF attacks by raising an exception.
   # For APIs, you may want to use :null_session instead.
   protect_from_forgery with: :null_session
end
```

##### Successful responses with no content

Some successful responses might not need to include a resonse body.
Ajax responses can be made a lot of faster with no response body.

```ruby
class EpisodesController < ApplicationController !
   def create
     episode = Episode.new(episode_params)
     if episode.save
       render nothing: true, status: 204, location: episode
     end
￼￼￼￼￼￼￼￼￼￼￼￼￼end

...
end
```

The 204 code - No Content means the server has fulfilled the request but does not need to return an entity-body.

```ruby
class HumansController < ApplicationController
  protect_from_forgery with: :null_session

  def create
    human = Human.new(human_params)

    if human.save
      render nothing: true,status: 204, location: human
    end
  end

  private

  def human_params
    params.require(:human).permit(:name, :brain_type)
  end
end
```

Responding with an empty body solved our performance issue. Now let’s go back and refactor our response to be a bit more expressive.


```ruby
  def create
    human = Human.new(human_params)

    if human.save
      head 204, location: human
    end
  end
 ```
 
 Responding with 422 and rendering json error
 
 ```ruby
 def create
    human = Human.new(human_params)

    if human.save
      head 204, location: human
    else
      render json: human.errors, status: 422
    end
  end
  ```

##### PUT is for replacing resources

PUT and PATCH are used for updating existing resources.

##### PATCH is for partial updates to existing resources

Always use PATCH for partial updates

##### DELETE - Discarding resources

The DELETE method indicates client is not interested in the given resource


##### There is a couple ways the server can implement delete method

1) Server deletes the record from the database

```ruby
class EpisodeController < ApplicationController
	def destroy
		episode = Episode.find params[:id]
		episode.destroy
		head 204
	end

end
```

2)Responding to delete by archiving records

Flag records as archived and new finder for unarchived records.

app/models/episode.rb
```ruby
class Episode < ActiveRecord::Base
	def self.find_unarchived(id)
		find_by!(id:id,archived:false)
	end
	
	def archive
		self.arhived = true
		self.save
	end
end
```
￼

##### Unsuccessful update with patch 
Now we need to write tests which ensure that clients cannot update existing humans with invalid data. We’ll intentionally issue a PATCH request with bad data to make sure our server reponds with the proper error.


```ruby
class UpdatingHumansTest < ActionDispatch::IntegrationTest
  setup { @human = Human.create!(name: 'Robert', brain_type: 'small') }

  test 'unsuccessful update on bad name' do
    patch "/humans/#{@human.id}",
      { human: { name: nil } }.to_json,
      { 'Accept' => Mime::JSON, 'Content-Type' => Mime::JSON.to_s }
    assert_equal 422, response.status
  end
end
```


##### Responding to unsuccessful updates

```ruby
 def update
    human = Human.find(params[:id])

    if human.update(human_params)
      render json: human, status: 200
    else
      render json: human.errors, status: 422
    end
  end
```

￼


## Level 5: API versioning

##### Introducing changes to a live API

Changes to the API cannot disrupt existing clients.

##### API versioning

Versioning helps prevent major changes from breaking existing clients

##### Versioning using the URI

Namespaces helps isolate controllers from different versions.


##### Testing routes for URI versioning

```ruby
class RoutesTest < ActionDispatch::IntegrationTest
  test 'routes to proper versions' do
    assert_generates "/v1/zombies", {controller: "v1/zombies", action: 'index'}
    assert_generates "/v2/zombies", {controller: "v2/zombies", action: 'index'}
  end
end
```


##### Versioning using custom media type and the accept header

application/xml

application/json

Custom Media Type

Example:
application/vnd.apocalypse[.version]+json

- application: payload is application-specific
- vnd.apocalypse: media type is vendor-specific
- [.version]: API version
- +json: response formats should be JSON
 

Integration testing API versions using the accept-header

```ruby
class ListingZombiesTest < ActionDispatch::IntegrationTest
  test 'show zombie from API version 1' do
    get '/zombies/1', {}, { 'Accept' => 'application/vnd.zombies.v1+json' }
    assert_equal 200, response.status
    assert_equal Mime::JSON, response.content_type
    zombie = json(response.body)
    assert_equal "This is version one", zombie[:message]
  end
end
```


##### Writing a route constraint class to check version


lib/api_version.rb
```ruby
class ApiVersion

  def initialize(version, default_version = false)
    @version, @default_version = version, default_version
  end

  def matches?(request)
    @default_version || check_headers(request.headers)
  end

  private
    def check_headers(headers)
      accept = headers['Accept']
      accept && accept.include?("application/vnd.zombies.#{@version}+json")
    end
end
```

##### Applying route constraint

config/routes.rb
```ruby
SurvivingRails::Application.routes.draw do
  require 'api_version'
  
  scope defaults: { format: 'json' } do
    scope module: :v1, constraints: ApiVersion.new('v1') do # Task 2
      resources :zombies
    end
    scope module: :v2, constraints: ApiVersion.new('v2',true) do # Task 3
      resources :zombies
    end
  end
end
```

##### Testing routes for the default API version

```ruby
class RoutesTest < ActionDispatch::IntegrationTest
  test 'defaults to v2' do
    assert_generates '/zombies', 
    { controller: 'v2/zombies/', action: 'index' }
  end
end
```

For a more robust solution for API versioning see:
https://github.com/bploetz/versionist/


## Level 6: Authentication












