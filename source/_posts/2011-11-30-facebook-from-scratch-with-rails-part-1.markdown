---
layout: post
title: "Facebook from Scratch (Part 1)"
date: 2011-11-30 09:23
comments: true
categories: 
---

In this tutorial, I will cover how to create a basic **Facebook application** that uses Facebook to perform the **authentication**, as well as running in the **canvas**.

## Getting Started

1. [Create a Facebook Application](http://www.facebook.com/developers/createapp.php)

1. In the **Web Site** section: 
      * Configure the **Site URL**, and point it to your Web Server. If you're developing locally, you can use `http://localhost:3000/`
      
{% img /images/posts/fb_1.jpg %}
  
1. In the **Facebook Integration** section:
      * Choose a **Canvas Page** name, which is your application's URL on facebook.com.
      * Configure the **Canvas URL**, and point it to your webserver. If you're developing locally, you can use 
        `http://localhost:3000/`. This will get used as the iframe src.
      * Switch the *Sandbox Mode* to **Sandbox**. This will allow testing with the SSL requirement.
      * Also switch **Canvas Height** to **Fluid** in the **Advanced** tab, since we'll be using the JavaScript SDK to resize our iframe.
      
{% img /images/posts/fb_3.jpg %}
{% img /images/posts/fb_3-5.jpg %}
{% img /images/posts/fb_4.jpg %}
      
1. Create a simple **Rails application**:

        $ rails new rwf_rails
      
1. Once the **Rails application** has been created, **create a controller** named 'home' with the actions 'index' and 'secret':

        $ cd rwf_rails
        $ rails g controller home index secret

    What we are going to do is have the index page *accessible to anyone*, but require the secret page to be *authenticated*.
    
1. Start up the Rails application:

        $ rails s
        
1. Go to <http://localhost:3000>

## The User Model

One of the main benefits of integrating with Facebook is the ease with which you can use the user's real identity provided by Facebook, and the wealth of existing user data that is [made available](http://developers.facebook.com/docs/api). For the sample application, we used a few different pieces of data to represent the *User*:

* Facebook User ID - the core stable identifier for a user
* Name
* Email
* Profile Picture
* <strike>Friends (set of Facebook User IDs)</strike>

The first step to getting this information is having the user [authorize your application](http://developers.facebook.com/docs/authentication/) and grant it the necessary [permissions](http://developers.facebook.com/docs/authentication/permissions). The result of granting these permissions gives you the `access_token` which enables access to the [APIs](http://developers.facebook.com/docs/api) that will give you the above data.

The first step would be to create the user model. We will do so using the `rails g model` command:

    $ rails g model user uid:string access_token:string name:string email:string picture:string

## Authentication with OmniAuth
  
[OmniAuth](https://github.com/intridea/omniauth) will perform the authentication needed to capture the initial signed_request data.

This portion of the tutorial is taken from the [Simple OmniAuth RailsCast by Ryan Bates](http://railscasts.com/episodes/241-simple-omniauth). If you have not watched that screencast, stop and go there first.

Add the **OmniAuth Facebook gem** to your `Gemfile`. At the time of writing this tutorial, the version number is 1.0.0.
        
```ruby Gemfile
gem 'omniauth-facebook', '1.0.0'
```

Run `bundle install` to update the Rails application with the gems.

        $ bundle install
        
Add the OmniAuth configuration to the initializer. I prefer using a configuration YAML file and loading it in the initializer:

```ruby config/facebook_config.yml
default: &defaults
  app_id: '120551398059624'
  app_secret: '81efe38a42b23090a3b5a1b385374a92'
  scope: 'email,user_location,user_birthday,user_likes'

development:
  <<: *defaults

test:
  <<: *defaults

production:
  <<: *defaults
  app_id: __PRODUCTION_APP_ID__
  app_secret: __PRODUCTION_APP_SECRET__
```

The data for the YAML file is found on the Facebook Basic Settings area:

{% img /images/posts/fb_5.jpg %}

Now let's load the configuration on initialization:

```ruby config/initializers/load_facebook_config.rb
# Parse the configuration file
raw_config = File.read("#{Rails.root}/config/facebook_config.yml")
FACEBOOK_CONFIG = YAML.load(raw_config)[Rails.env].symbolize_keys

# Initialize the OmniAuth middleware onto the stack
Rails.application.config.middleware.use OmniAuth::Builder do
  provider( :facebook, 
            FACEBOOK_CONFIG[:app_id], 
            FACEBOOK_CONFIG[:app_secret],
            { :scope => FACEBOOK_CONFIG[:scope] } )
end
```

Create the **Sessions Controller** with the following:

    $ rails g controller sessions

```ruby SessionsController.rb
class SessionsController < ApplicationController

  def create
    # Shows the OmniAuth signed_request hash in the error
    raise request.env['omniauth.auth'].to_yaml
  end
  
end
```

Delete the `public/index.html` file and add the callback route to `routes.rb`:

```ruby routes.rb
RwfRails::Application.routes.draw do

  # semi-static pages
  namespace :home do
    match 'index'
    match 'secret'
  end
  
  # capture Facebook's authentication response
  match "/auth/facebook/callback" => "sessions#create"
  
  root :to => "home#index"
end
```

You can now restart your server and open <http://localhost:3000/auth/facebook>.

{% img /images/posts/fb_6.jpg %}
{% img /images/posts/fb_7.jpg %}

With this data, we can now populate our User model. I will go over these steps very quickly b/c they are already covered in the [RailsCast](http://railscasts.com/episodes/241-simple-omniauth).

First, let's modify the SessionsController to the following:

```ruby sessions_controller.rb
class SessionsController < ApplicationController
  
  def create
    # Find an existing user if one exists
    user = User.where(:uid => auth_hash['uid']).first
    
    # Create a new user from the auth if the user doesn't already exist
    user ||= User.create_with_omniauth(auth_hash)

    begin
      # Update token on each create request
      if auth_hash['credentials']['token']
        user.update_attribute(:access_token, auth_hash['credentials']['token'])
      end
    rescue
    end

    session[:user_id] = user.id
    redirect_to home_secret_path, :notice => "Signed in!"
  end
  
  protected

    # Helper method for easy access
    def auth_hash
      request.env['omniauth.auth']
    end
  
end
```

You can see that the method `create_with_omniauth` is being called above. Let's add that to our **User** model:

```ruby user.rb
class User < ActiveRecord::Base
  
  def self.create_with_omniauth(auth) 
    user = User.new
    user.uid          = auth['uid']
    user.access_token = auth['credentials']['token']
    user.name         = auth['info']['name']
    user.email        = auth['info']['email']
    user.picture      = auth['info']['image']
    if user.save
      return user
    else
      puts "Unable to save user:\n\n#{user.errors.to_yaml}"
      return nil
    end
  end
  
end
```

Let's add the **current_user** to our `application_controller.rb` so we can access it from any erb file:

```ruby application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery
  helper_method :current_user
  
  private

    # Helper method to load the current user, or nil if not exist
    def current_user
      begin
        @current_user ||= User.find(session[:user_id]) if session[:user_id]
      rescue ActiveRecord::RecordNotFound
        nil # user_id is still in session, but is not valid.
      end
    end
    
end
```

What should happen now is once the user is authorized, Facebook redirects the user to the callback URL, which is `/auth/facebook/callback`, which points to `SessionsController#create`.

Let's update `secret.html.erb` to reflect the changes:

```erb secret.html.erb
<h1>Secret Page</h1>

Your email is <%= current_user.email %>
```

Run `rake db:migrate` then visit the url <http://apps.facebook.com/rwf_rails/auth/facebook>.

You should now see something similar to this:

{% img /images/posts/fb_9.jpg %}

We have a mild problem, however. If you look at the url in the browser, it shows that our current url is still **/auth/facebook**. If we click '*refresh*', it  will re-authenticate. So how do we fix this? Go to Part 2 of this tutorial.