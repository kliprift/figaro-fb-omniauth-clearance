reference_links:
```
1) https://github.com/thoughtbot/clearance
2) https://github.com/NextAcademy/how_to_integrate_clearance_wtih_facebook
3) https://gist.github.com/stevebourne/2394427
4) https://github.com/bestofthesoul/June16_tutorial/blob/master/figaro_fbomniauth_clearance.md
```

1) gemfile
```
#user authentication
gem 'omniauth'
gem 'omniauth-facebook'
gem 'clearance'
```

2)
```$bundle install```

3)
```
$bundle exec rake db:create
```
```
$rails generate clearance:install
```

4) app/views/layout/application.html.erb

```
<% if signed_in? %>
  Signed in as: <%= current_user.email %>
  <%= button_to 'Sign out', sign_out_path, method: :delete %>
<% else %>
  <%= link_to 'Sign in', sign_in_path %>
<% end %>

<div id="flash">
  <% flash.each do |key, value| %>
  <div class="flash <%= key %>"><%= value %></div>
  <% end %>
</div>
```



5) config/environments/{development,test}.rb

```
config.action_mailer.default_url_options = { host: 'localhost:3000' }
```

6) config/initializers/clearance.rb

 ```
Clearance.configure do |config|
  config.routes = false
  config.allow_sign_up = true
  config.cookie_domain = '.example.com'
  config.cookie_expiration = lambda { |cookies| 1.year.from_now.utc }
  config.cookie_name = 'remember_token'
  config.cookie_path = '/'
  config.routes = false
  config.httponly = false
  config.mailer_sender = 'reply@example.com'
  config.password_strategy = Clearance::PasswordStrategies::BCrypt
  config.redirect_url = '/'
  config.secure_cookie = false
  config.sign_in_guards = []
  config.user_model = User
end

 ```

 7) facebook omniauth application
 ```
 $touch config/initializers/omniauth.rb
 ```
config/initializers/omniauth.rb
 ```
 Rails.application.config.middleware.use OmniAuth::Builder do
  provider :facebook, ENV['facebook_key'], ENV['facebook_key_secret']
end
```

app/views/layout/application.erb
```<%= link_to "Connect With Facebook", "/auth/facebook" %>```

8) Use gem `figaro` to store confidential stuff

``` gem 'figaro'```

```bundle install```

```bundle exec figaro install```

9) config/application.yml
```
facebook_key: 'XXXX'
facebook_key_secret: 'XXXX'

```

10) create model `authentication`
```
$rails g model authentication uid:string provider:string token:string user_id:integer
```
app/models/authentication.rb
```
class Authentication < ActiveRecord::Base

  belongs_to :user

  def self.create_with_omniauth(auth_hash)
    create! do |auth|
      auth.provider = auth_hash["provider"]
      auth.uid = auth_hash["uid"]
      auth.token = auth_hash["credentials"]["token"]
    end
  end

  def update_token(auth_hash)
    self.token = auth_hash["credentials"]["token"]
    self.save
  end

end
```

```
$bundle exec rake db:migrate
```

11) create `users controller` and `welcome controller`
```
$rails g controller users show
```
```
$rails g controller welcome index
```



12)
```
$rails g clearance:routes
```
config/routes.rb
```

root 'welcome#index'

#*** rails g clearance:routes show all these default Clearance routes
resources :passwords, controller: "clearance/passwords", only: [:create, :new]
resource :session, controller: "clearance/sessions", only: [:create]
resources :users, controller: "clearance/users", only: [:create] do
  resource :password,
    controller: "clearance/passwords",
    only: [:create, :edit, :update]
end

get "/sign_in" => "clearance/sessions#new", as: "sign_in"
delete "/sign_out" => "clearance/sessions#destroy", as: "sign_out"
get "/sign_up" => "clearance/users#new", as: "sign_up"
#*** end of default routes of Clearance

resources :users, controller: "users", only: :show
get "/auth/:provider/callback" => "sessions#create_from_omniauth"
```

13) create `sessions controller`

```
$touch app/controllers/sessions_controller.rb
```

```
class SessionsController < Clearance::SessionsController

  def create_from_omniauth
    auth_hash = request.env["omniauth.auth"]

    authentication = Authentication.find_by_provider_and_uid(auth_hash["provider"], auth_hash["uid"]) || Authentication.create_with_omniauth(auth_hash)
    if authentication.user
      user = authentication.user
      authentication.update_token(auth_hash)
      @next = user_path(user)
      @notice = "You're signed in again with fb"
    else
      user = User.create_with_auth_and_hash(authentication,auth_hash)
      @next = user_path(user)
      @notice = "Signed in first time with facebook"
    end
    sign_in(user)
    redirect_to @next, :notice => @notice
  end

end
```

14) app/models/user.rb
```
class User < ActiveRecord::Base
  include Clearance::User

  validates :email, uniqueness: true
  has_many :authentications, :dependent => :destroy

  def self.create_with_auth_and_hash(authentication,auth_hash)
    create! do |u|
      #u.first_name = auth_hash["info"]["first_name"]
      # u.last_name = auth_hash["info"]["last_name"]
      # u.friendly_name = auth_hash["info"]["name"]
      u.email = auth_hash["extra"]["raw_info"]["email"]
      u.password = SecureRandom.hex(6)
      u.authentications<<(authentication)
    end
  end

  def fb_token
    x = self.authentications.where(:provider => :facebook).first
    return x.token unless x.nil?
  end


  # def password_optional?
  #   true
  # end

end
```


15) to change clearance default view files
```
$rails generate clearance:views
```


