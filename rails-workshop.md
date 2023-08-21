# Rails Workshop

## Setting up the project

### Install the appropriate version of Ruby

There's a multitude of ways to install and manage Ruby versions. Usually, projects will require you to be able to spin up many versions of Ruby. It's simply too annoying to uninstall and install a different version each time. That's why there exist different version managers that support Ruby. Some of them are `rbenv`, `rvm` and `asdf`.

At Xmartlabs we've mostly used [rbenv](https://github.com/rbenv/rbenv), so these next instructions take for granted that you have it installed on your machine.

```sh
rbenv install 3.1.1
rbenv rehash
```

### Install bundler

```sh
gem install bundler
```

### Install Rails

```sh
gem install rails -v 7.0.3.1
```

### Generate a rails project

```sh
rails new --database postgresql --api backend --no-git
```

### Start the rails server

```sh
cd backend
bundle exec rails s
```

You should see an output similar to this:

```
=> Booting Puma
=> Rails 7.0.3.1 application starting in development
=> Run `bin/rails server --help` for more startup options
Puma starting in single mode...
* Puma version: 5.6.4 (ruby 3.1.1-p18) ("Birdie's Version")
*  Min threads: 5
*  Max threads: 5
*  Environment: development
*          PID: 13939
* Listening on http://127.0.0.1:3000
* Listening on http://[::1]:3000
Use Ctrl-C to stop
```

### The database

Make sure you have Postgres installed and running on your machine. Then, you'll need to provide Rails with credentials to connect to it. If you already know the credentials then just add them to the environment file as detailed below.

#### Booting up the database with Docker Compose

This is an optional step. First, create a file named `dev.docker-compose.yml` file with the following content:

```yml
# dev.docker-compose.yml
version: "3"
services:
  database:
    image: postgres:14-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - ./db-data:/var/lib/postgresql/data
    ports:
      - 5432:5432
volumes:
  db:
    driver: local
```

Create the folder where our database will be stored. It's recommended you do this since Docker will create it with the root user if not.

```sh
mkdir db-data
```

Then, start the database in background with this command:

```sh
docker-compose -f dev.docker-compose.yml up -d
```

#### Environment variables

In order to be able to read environment variables from a file we need to install a new library and add it to our Gemfile. You can do this by adding a line on the Gemfile:

```ruby
group :development, :test do
  # See https://guides.rubyonrails.org/debugging_rails_applications.html#debugging-with-the-debug-gem
  gem "debug", platforms: %i[ mri mingw x64_mingw ]
  gem 'dotenv-rails', '~> 2.7.6' # <-- This is the new line!
end
```

Then, simply install the new library:

```sh
bundle install
```

Lastly, create a `.env.development.local` file with the following variables:

```sh
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=postgres
```

**Note**: these will only work if you used the docker-compose DB detailed above. If you are using your own installation it's likely your credentials will be different.

#### Configuring the database

First, we need to tell Rails to load our environment variables somewhere for ease of access. Usually this is done on the configuration files. In this case we're going to use `application.rb` which stores general configuration for all environments.

```ruby
config.database_host = ENV.fetch('DB_HOST')
config.database_port = ENV.fetch('DB_PORT', 5432)
config.database_username = ENV.fetch('DB_USERNAME')
config.database_password = ENV.fetch('DB_PASSWORD')
```

Finally you'll need to point to these values on the `database.yml` file. On the `default` block, add these lines:

```yml
default: &default
  adapter: postgresql
  encoding: unicode
  # For details on connection pooling, see Rails configuration guide
  # https://guides.rubyonrails.org/configuring.html#database-pooling
  # NOTE: NEW LINES START HERE:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  host: <%= Rails.application.config.database_host %>
  port: <%= Rails.application.config.database_port %>
  username: <%= Rails.application.config.database_username %>
  password: <%= Rails.application.config.database_password %>
```

#### Creating the database

Run this command to create our databases:

```sh
bundle exec rails db:create
```

## Creating our first model

Let's start by creating the model we'll be using: Message. Rails provides us with a quick way to create a model by running a simple generator.

```sh
bundle exec rails generate model Message content:string due_date:date is_complete:boolean
```

This should output something like this:

```
invoke  active_record
create    db/migrate/20220823192835_create_messages.rb
create    app/models/message.rb
invoke    test_unit
create      test/models/message_test.rb
create      test/fixtures/messages.yml
```

## Running migrations

To run the migrations, this command should work:

```sh
bundle exec rails db:migrate
```

*Et Voilà*, our database now has its first table.

### Checking your model works

You can quickly check your model by running an interactive Rails console:

```sh
bundle exec rails c
```

Then you can start running Ruby code:

```ruby
Message.count # should output 0
Message.create!(content: 'This is the content of the message', due_date: Time.now, is_complete: false)
Message.count # Should output 1
Message.first # Should output the message you just created.
```

## Serving data

Let's create our first controller and route. First, let's add a line to the routes file:

```ruby
Rails.application.routes.draw do
  # Define your application routes per the DSL in https://guides.rubyonrails.org/routing.html

  # Defines the root path route ("/")
  # root "articles#index"

  resources :messages, only: %i[index] # <-- NEW LINE
end
```

Then, to check our route was created correctly, run:

```sh
bundle exec rails routes
```

You should see the message route we just created.

### Creating the controller

Create a controller file on `app/controllers/messages_controller.rb`. Then, add code for it:

```ruby
# frozen_string_literal: true

class MessagesController < ApplicationController
  def index
    render json: Message.all
  end
end
```

You can very easily check that your controller works by accessing the route on a browser: `http://localhost:3000/messages`. You should see this response:

```json
[
  {
    "id": 1,
    "content": "This is the content of the message",
    "due_date": "2022-08-23",
    "is_complete": false,
    "created_at": "2022-08-23T19:35:03.268Z",
    "updated_at": "2022-08-23T19:35:03.268Z"
  }
]
```

Or something similar (dates will differ).

## Creating data

Now let's add an endpoint to create a message. The endpoint should receive the three fields a message has and return the newly-created field. First, as always, routes! Change the line you added on the `routes` file and add the `create` action to the symbol array:

```ruby
resources :messages, only: %i[index create]
```

Then let's create an action on the `MessagesController`:

```ruby
# app/controllers/messages_controller.rb

def create
  create_params = params.require(:message).permit(:content, :due_date, :is_complete)

  message = Message.create!(create_params)

  render json: message
end
```

## Service Objects

Service objects are a way to centralize application logic outside of models and controllers. A service object is not always needed, but it's a good way to encapsulate business logic without other dependencies. Service objets should be testeable, clear and concise. They should always do only one thing.

For example, let's create a service object that handles the creation of our `Message` instances.

```ruby
# app/services/messages/create_message_service.rb
class Messages::CreateMessageService
  class InvalidNameError < StandardError; end

  def initialize(message_data:)
    @message_data = message_data
  end

  def run
    validate_data!

    create!
  end

  private

  def validate_data!
    raise InvalidNameError, 'name must not be blank' if message_data[:name].blank?
  end

  def create!
    Message.create!(**message_data)
  end
end
```

Here's how the controller looks now:

```ruby
# frozen_string_literal: true

class MessagesController < ApplicationController
  def index
    render json: Message.all
  end

  def create
    create_params = params.require(:message).permit(:content, :due_date, :is_complete)

    render json: Services::CreateMessageService.new(message_data: create_params).run
  end
end

```
