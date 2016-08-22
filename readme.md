# OmniAuth standalone with Github

This repository shows how to use `OmniAuth` without devise. This requires us to create our own `User` model, define our own `SessionsController` as well our own `current_user` methods in the ApplicationController

See the individual commits for the step-by-step process.

## Step 1.

Add the following gems to your `Gemfile` and then `bundle install`

```
# Authentication via oauth
gem 'omniauth'
gem 'omniauth-github'

# Environment
gem 'dotenv-rails'
```

## Step 2

Create or update the user model

If the user doesn't already exist:
- `rails generate model User name provider uid nickname access_token`

If the user already exists:
- `rails generate migration AddOAuthToUser`
- Define the migration as:
```
  def up
    # Which provider are we using (e.g. github, facebook, twitter)
    add_column :users, :provider, :string

    # User id at the provider (e.g. github/facebook/twitter)
    add_column :users, :uid, :string

    # User's name at the provider
    add_column :users, :nickname, :string

    # Secret token identifying us to the provider
    # KEEP THIS SECRET
    add_column :users, :access_token, :string

    # If you already had password_digest and email, then add these:
    # remove_column :users, :email
    # remove_column :users, :password_digest
  end

  def down
    remove_column :users, :provider
    remove_column :users, :uid
    remove_column :users, :nickname
    remove_column :users, :access_token

    # If you already had password_digest and email, then add these:
    # add_column :users, :email, :string
    # add_column :users, :password_digest, :string
  end
```

## Step 3: Add a method for managing users

We will add a method to `User` to help with creating/updating users from the OAuth data

Add the following to `app/models/user.rb`
```
# Factory method to create a user from some omniauth data
# Omniauth will use this to build a *NEW* user for us
def self.from_omniauth(authentication_data)
  user = User.where(provider: authentication_data['provider'],
                    uid: authentication_data['uid']).first_or_create

  Rails.logger.debug "The user is #{user.inspect}"

  user.name         = authentication_data.info.name
  user.nickname     = authentication_data.info.nickname
  user.access_token = authentication_data.info.access_token

  user.save!

  Rails.logger.debug "After saving, the user is #{user.inspect}"

  return user
end
```

- If you already had email/password auth, then remove the following from `app/models/user.rb`

```
validates :name, presence: true
validates :email, presence: true

has_secure_password
```

## Step 4: Add a place to configure Omniauth

Now we setup Omniauth to understand the providers we want to use. For now we
will just use github authentication.

Create a file `config/initializers/omniauth.rb` -- the `config/initializers` directory will contain configuration/initialization files for various gems/utilities we add to our Rails environment. These files are loaded automatically by rails when our application starts up.

```
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :github, ENV['GITHUB_APP_ID'], ENV['GITHUB_APP_SECRET']
end
```

We are using the *ENV* (Environment) to store these secrets so they do not get into our github repository and accidentally get published online. This will also come in handy when we deploy to Heroku


## Step 5: Now we must sign up for an application id and Secret

Each provider has it's own way of signing up for an app ID and secret.

For github you will visit your profile and fill out a "Developer Application" OAuth.

- Visit `https://github.com/settings/profile`
- Select `OAuth Appliactions` on the left sidebar
- Select `Developer Applications`
- Register a new application
- Select "Local Development" as the name
- Select "http://localhost:3000" as the homepage URL
- Select "Local Development" as the description
- Select "http://localhost:3000" as the callback URL

- NOTE: Use a different number if the port your local application is different than 3000 (e.g. if it is 5000, or 4567)

## Step 6: Add these to our `.env` file

Create a file `.env` in the root of your project tree

```
GITHUB_APP_ID=*YOUR APP ID HERE*
GITHUB_APP_SECRET=*YOUR SECRET HERE*
```

## Step 7: Make *SURE* you put .env in .gitignore

Edit your `.gitignore` to have a line with `.env` listed.

## Step 8: Setup routes

Ensure your `config/routes.rb`

```
  get    '/auth/:provider'          => 'omniauth#auth',    as: :auth
  get    '/auth/:provider/callback' => 'session#create'
  get    '/auth/failure'            => 'session#failure'
```

If you do not have a login/logout path we can add that as well:

```
  get '/login' => 'session#new'
  post '/login' => 'session#create'
  get '/logout' => 'session#destroy'
```

## Step 9: Update our application_controller

To `app/controllers/application_controller.rb`

Add the following:

```
# Assign the current user
def current_user=(user)
  session[:user_id] = user.id
end
```

Ensure we have a method `current_user` with this definition:

```
def current_user
  @current_user ||= User.find_by(id: session[:user_id])
end
helper_method :current_user
```

Also make sure these methods exist:

```
  # Returns a boolean representing if the user is logged in
  def logged_in?
    !!current_user
  end
  helper_method :logged_in?

  # Method to use in filter to ensure the user is logged in
  def authenticate!
    unless logged_in?
      redirect_to login_path
    end
  end
```

## Step 10: Update our SessionController

Change `app/controller/session_controller.rb`

```
class SessionController < ApplicationController
  # logging in
  def new
  end

  # handle the post from the login page
  def create
    self.current_user = User.from_omniauth(request.env['omniauth.auth'])

    if current_user
      redirect_to root_path
    else
      redirect_to login_path
    end
  end

  # logout
  def destroy
    session[:user_id] = nil
    redirect_to root_path
  end

  # Show the failure page
  def failure
    # TODO, create failure.html.erb
  end
end
```

## Step 11: Change the login page

Change `app/views/session/new.html.erb` to only contain:

```
<%= link_to 'Login with Github', auth_path(provider: 'github') %>
```

*NOTE:* Style this nicely!

*NOTE:* We can also add other providers here if you have set them up!

## Step 12: Try logging in with Github!
