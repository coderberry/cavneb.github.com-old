---
layout: post
title: "Facebook from Scratch (Part 2)"
date: 2011-11-30 10:48
comments: true
categories: 
---

This tutorial assumes that you have finished [Part 1]("../facebook-from-scratch-with-rails-part-1/"). If not, please start there.

## Facebook Canvas Redirects

The facebook app resides in an iFrame. When you redirect internally within an application, you are redirecting within the iFrame. This means that if you ever refresh the browser, it will take you back to the page you initially started on. *This is bad.*

The way to fix this is to make sure that all redirects and links point to the *top* of the browser. Here's the goals:

1. Create a method of redirecting that will redirect to the parent browser and not just within the iFrame.
2. Create a method of linking that will link to the parent browser as well.

### redirect_to => redirect_to_facebook

In the SessionsController, we are performing a redirect upon successful authentication (see [Part 1]("../facebook-from-scratch-with-rails-part-1/")).
We want to modify this to redirect to the same URL, but also modify the browsers URL. Let's create a method in the ApplicationController so we can have global access to it:

```ruby application_controller.rb
##
# Helper controller method to redirect to facebook on the top (for canvas apps)
##
def redirect_to_facebook(target_path, *args)
  raise ActionControllerError.new("Cannot redirect to nil!") if target_path.nil?
  html_options = args.extract_options!.symbolize_keys
  base_url = html_options[:base_url] ||= FACEBOOK_CONFIG[:facebook_app_url]
  url = "#{base_url}#{target_path}"
  logger.debug "Redirecting to URL: #{url}"
  render :text => "<html><body><script type=\"text/javascript\">window.top.location='#{url}';</script></body></html>"
end
```

This code is very similar to the `link_to` method in Rails. All I do here is render the redirect as a javascript `window.top.location` call. Note that in this code, we refer to the *FACEBOOK_CONFIG[:facebook_app_url]*. Let's add that to our *facebook_config.yml* file:

```ruby config/facebook_config.yml
default: &defaults
  app_id: '120551398059624'
  app_secret: '81efe38a42b23090a3b5a1b385374a92'
  scope: 'email,user_location,user_birthday,user_likes'
  facebook_app_url: '//apps.facebook.com/rwf_rails'

...
```

Let's now modify the redirect within the SessionsController:

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
    redirect_to_facebook secret_path, :notice => "Signed in!"
  end
  
  protected

    # Helper method for easy access
    def auth_hash
      request.env['omniauth.auth']
    end
  
end
```

Let's restart the application so the new config is set. Then refresh the page again and it should show the proper URL in the browser.

### link_to => link_to_facebook

## signed_request

The typical flow for a user when using the application begins with the user landing at some Canvas URL, like [http://apps.facebook.com/rfw_rails/](http://apps.facebook.com/rfw_rails/). At this point, Facebook will load up it's chrome, and render a `<iframe>` tag to your application. You'll notice there isn't a `src` specified. Using some JavaScript and the `<form>` tag, Facebook triggers a `POST` request to your application. This is done for security reasons, as the sensitive user data won't be sent via the HTTP Referrer header as long it's sent as `POST` data. Additionally, in order to ensure the data Facebook sends over is not tampered with, it provides a signed JSON data structure, which is [signed using](http://developers.facebook.com/docs/authentication/canvas) the application secret.
  
To capture the signed_request on every request, we will need to convert the signed_request from `POST` to `GET`. This can be done using middleware. There are solutions out there to do so, but we will roll our own because it's simple.

Create a file in the lib folder called `facebook.rb` with the following:

```ruby lib/facebook.rb
module Rack
  class Facebook
    
    def initialize(app)
      @app = app
    end

    def call(env)
      request = Request.new(env)
      
      if request.POST['signed_request']
        env["REQUEST_METHOD"] = 'GET'
      end
            
      return @app.call(env)
    end
  
  end
end
```

Now that this is in place, you can modify your SessionsController to display the params, which will include the signed_request:

```ruby sessions_controller.rb
class HomeController < ApplicationController
  def index
    render :text => params[:signed_request]
  end

  def secret
    render :text => "home#secret"
  end
end
```

Visit the Facebook application page found on the application settings (see 2nd image above). In this example, the url is [http://apps.facebook.com/rwf_rails/](http://apps.facebook.com/rwf_rails/)

You should see something similar to this:

{% img /images/posts/fb_8.jpg %}