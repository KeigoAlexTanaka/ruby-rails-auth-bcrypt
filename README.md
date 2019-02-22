# Rails Authentication

### Objectives
*After this lesson, students will be able to:*

- Understand how to set and retrieve data from `session`
- Know how to set and check passwords using `BCrypt`
- Understand how to keep track of a `current_user`
  - `sign_in`/`sign_out`, `ensure_signed_in`/`ensure_signed_out`
- Be able to create a Rails app with Auth without using [`Devise`](https://github.com/plataformatec/devise)
  - Not from memory, but from following along with an example

  
### Preparation
*Before this lesson, students should:*

- Be able to make a Rails app without auth

---

This tutorial is for adding authentication to a vanilla Ruby on Rails application using [`bcrypt`](https://github.com/codahale/bcrypt-ruby) and `has_secure_password`.

The point of authentication is to ensure that the user is who they say they are. The standard way of managing this is through registering your user using a sign-up form. Once the registered user is logged in, you keep track of them using a session until they log out.

User Authentication allows us to restrict access to a portion of the application so that only users who log in with a valid username and password can have access.

### Create a new Rails application

To create a new Rails app, we use the `rails new` command. This sets us up with our skeleton Rails app. In the Terminal, in the directory where you want to create a project, generate a new Rails project and cd into it:

```
$ rails new my-rails-auth-project -T --database=postgresql
$ cd my-rails-auth-project
    
# -T means without Test::Unit (maybe you prefer to use rspec instead)
```

### Setup the database
Set up an empty database:

```
$ rails db:create
```
it will return the following:

```
Created database 'my-rails-auth-project_development'
Created database 'my-rails-auth-project_test'
```

Open the project in the current directory using your text editor (in this case it's Sublime Text):

```
$ subl .
```
Start the server and open up the project in the browser, `localhost:3000`:

```
$ rails server
```

## USERS

### Generate User model 
Create a User model with `name` and `email` attributes as well as the optional string parameters. **Remember:** in Rails, models are singlular and capitalized. Controllers and routes are plural and lowercase.

```
$ rails g model User name:string email:string

# run the migration to update the database with this change
$ rails db:migrate
```

Before running the migration, check the `db/migrate/[timestamp]_create_users.rb` folder to make sure the migration is accurate:

```
class CreateUsers < ActiveRecord::Migration[5.2]
  def change
    create_table :users do |t|
      t.string :name
      t.string :email

      t.timestamps
    end
  end
end
```

Your Terminal window will return the following information:

```
== [timestamp] CreateUsers: migrating ======================================
-- create_table(:users)
   -> 1.8143s
== [timestamp] CreateUsers: migrated (1.8169s) =============================
```

while your schema should look like the following:

```
  create_table "users", force: :cascade do |t|
    t.string "name"
    t.string "email"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
  end
```

Navigate to the `app/models/User.rb` file:

```
class User < ApplicationRecord
end
```
Then, step into the Rails console to create a new User:

```
$ rails c --sandbox

# create a new user object
>> User.new
=> #<User id: nil, name: nil, email: nil, created_at: nil, updated_at: nil>

>> user = User.new(name: 'Celeste Layne', email: 'celeste.layne@hotmail.com')
=> #<User id: nil, name: "Celeste Layne", email: "celeste.layne@hotmail.com", created_at: nil, updated_at: nil>

>> user.valid?
=> true

>> user.save
   (14.2ms)  SAVEPOINT active_record_1
  User Create (49.3ms)  INSERT INTO "users" ("name", "email", "created_at", "updated_at") VALUES ($1, $2, $3, $4) RETURNING "id"  [["name", "Celeste Layne"], ["email", "celeste.layne@hotmail.com"], ["created_at", "2019-02-21 22:03:32.018011"], ["updated_at", "2019-02-21 22:03:32.018011"]]
   (9.8ms)  RELEASE SAVEPOINT active_record_1
=> true

>> user
=> #<User id: 1, name: "Celeste Layne", email: "celeste.layne@hotmail.com", created_at: "2019-02-21 22:03:32", updated_at: "2019-02-21 22:03:32">

>> user.name
=> "Celeste Layne"

>> user.email
=> "celeste.layne@hotmail.com"

>> user.created_at
=> Thu, 21 Feb 2019 22:03:32 UTC +00:00

>> User.find(1)
=> #<User id: nil, name: "Celeste Layne", email: "celeste.layne@hotmail.com", created_at: nil, updated_at: nil>

>> User.find(3)
=> ActiveRecord::RecordNotFound (Couldn't find User with 'id'=3)

>> user.email = "celeste.layne@gmail.com"
=> "celeste.layne@gmail.com"

>> user
=> #<User id: 1, name: "Celeste Layne", email: "celeste.layne@gmail.com", created_at: "2019-02-21 22:03:32", updated_at: "2019-02-21 22:03:32">

>> user.save
   (26.3ms)  SAVEPOINT active_record_1
  User Update (45.6ms)  UPDATE "users" SET "email" = $1, "updated_at" = $2 WHERE "users"."id" = $3  [["email", "celeste.layne@gmail.com"], ["updated_at", "2019-02-21 22:27:35.687162"], ["id", 1]]
   (12.0ms)  RELEASE SAVEPOINT active_record_1
=> true
```
_Note:_ The `id` has been assigned a value of 1; while `updated_at` / `created_at` have been assigned the current time and date.

Another way to update records is using `update_attributes`:

```
>> user.update_attributes(name: "Dennis Layne", email: "dennis@trinimail.com")
   (1.8ms)  SAVEPOINT active_record_1
  User Update (8.3ms)  UPDATE "users" SET "name" = $1, "email" = $2, "updated_at" = $3 WHERE "users"."id" = $4  [["name", "Dennis Layne"], ["email", "dennis@trinimail.com"], ["updated_at", "2019-02-21 22:29:58.364706"], ["id", 1]]
   (0.4ms)  RELEASE SAVEPOINT active_record_1
=> true

>> user.name
=> "Dennis Layne"
```

### Validations

At this point, any string we decide to type into the name and email fields is valid. Active Record allows us to impose constraints using validations. Some of the most common validations include: _presence, length, and uniqueness_.

###### Validating the presence of a name and email attribute.
Presence verifies that a given attribute (in this case, name and email) is present using the validates method with argument presence: true.

```
class User < ApplicationRecord
    validates :name, presence: true
    validates :email, presence: true
end
```
###### Validating length
Length verifies that a name field with a maximum of 50 characters will not validate names that are 51 characters in length.

```
class User < ApplicationRecord
    validates :name, presence: true, length: { maximum: 50 }
    validates :email, presence: true, length: { maximum: 255 }
end
```
###### Validating uniqueness
Uniqueness of email addresses uses the :unique option for the validates method.

```
class User < ApplicationRecord
    validates :name, presence: true, length: { maximum: 50 }
    validates :email, presence: true, length: { maximum: 255 }, uniqueness: true
end
```
_Note:_ Active Record uniqueness validation does not guarantee uniqueness at the database level

The solution is to enforce uniqueness at the database level as well as at the model level. Our method is to create a database index on the email column, and then require that the index be unique.

###### Generate index on name [or email]

```
$ rails generate migration add_index_to_users_email
```
The migration will look like the following:

```
# db/migrate/[timestamp]_add_index_to_users_email.rb
class AddIndexToUsersEmail < ActiveRecord::Migration
  def change
  	# use add_index method to add an index on the email column of the users table
  	add_index :users, :email, unique: true
  end
end
```
_Note:_  the option `unique: true` enforces uniqueness

Don't forget to migrate:

```
$ rails db:migrate
```
The users table in the `db/schema.rb` file now includes the following line:

```
  create_table "users", force: :cascade do |t|
    ...
    t.index ["email"], name: "index_users_on_email", unique: true
  end
```

### Add Dependencies

In the Gemfile, uncomment bcrypt:

```
gem 'bcrypt', '~> 3.1.7'
```
We need bcrypt to securely store passwords in our database. Then, from the root of the application, run the following command:

```
$ bundle install
```

Then, restart your server.

### Passwords

Rails has a method called `has_secure_password` which you include in your User model and it  adds a lot of the functionality. The only requirement for `has_secure_password` is for the corresponding model to have an attribute called `password_digest`.

Let's generate a migration to add the `password_digest` column to the users table:

```
$ rails generate migration add_password_digest_to_users password_digest:string
```
The migration will look like the following:

```
# db/migrate/[timestamp]_add_password_digest_to_users.rb
class AddPasswordDigestToUsers < ActiveRecord::Migration[5.2]
  def change
    add_column :users, :password_digest, :string
  end
end
```
To apply it, we just migrate the database:

```
$ rails db:migrate
```
It's at this point we add `has_secure_password` to the User model. `has_secure_password` uses bcrypt to hash (encrypt?) the password via authentication methods.

```
class User < ApplicationRecord
    validates :name, presence: true, length: { maximum: 50 }
    validates :email, presence: true, length: { maximum: 255 }, uniqueness: true
    
    has_secure_password
end
```
Now, include a presence validation and minimum length to ensure nonblank passwords, to the password attribute:

```
class User < ApplicationRecord
    validates :name, presence: true, length: { maximum: 50 }
    validates :email, presence: true, length: { maximum: 255 }, uniqueness: true
    
    has_secure_password
    validates :password, presence: true, length: { minimum: 6 }
end
```

Now in our rails console, we can set a password and check if it matches:

```
>> user.password = 'epiphany'
=> "epiphany"

>> user.is_password?('epiphany')
=> true

>> user.save
   (32.3ms)  SAVEPOINT active_record_1
  User Exists (36.0ms)  SELECT  1 AS one FROM "users" WHERE "users"."email" = $1 LIMIT $2  [["email", "celeste.layne@hotmail.com"], ["LIMIT", 1]]
  User Create (56.7ms)  INSERT INTO "users" ("name", "email", "created_at", "updated_at", "password_digest") VALUES ($1, $2, $3, $4, $5) RETURNING "id"  [["name", "Celeste Layne"], ["email", "celeste.layne@hotmail.com"], ["created_at", "2019-02-21 23:04:10.621861"], ["updated_at", "2019-02-21 23:04:10.621861"], ["password_digest", "$2a$10$UZoyMjCRLI84VJe2XsAX3.wA7xa2xQMFyTfgcqzYO0a1OaNHso8XW"]]
   (0.6ms)  RELEASE SAVEPOINT active_record_1
```
_Note:_ Saving the record returns the encrypted password digest and NOT the password ('epiphany')

Authenticate a user given the name and password:

```
class User < ApplicationRecord
    validates :name, presence: true, length: { maximum: 50 }
    validates :email, presence: true, length: { maximum: 255 }, uniqueness: true
    
    has_secure_password
    validates :password, presence: true, length: { minimum: 6 }
    
	def self.find_from_credentials(email, password)
		user = find_by(email: email)
		return nil unless user
		user if BCrypt::Password.new(user.password_digest).is_password?(password)
	end
end
```
Now, find the user record by the email address:

```
>> User.find_by(email: "celeste.layne@hotmail.com")
=> #<User id: 3, name: "Celeste Layne", email: "celeste.layne@hotmail.com", created_at: "2019-02-22 00:09:44", updated_at: "2019-02-22 00:09:44", password_digest: "$2a$10$oulAW4mTmy4sOmLaKRdsz.rglfMyQBrUeSq4C1ZBwU5...">
```

Next, test an email address and password that works and one that doesn't:

```
# this pair works
>> User.find_from_credentials('celeste.layne@hotmail.com', 'epiphany')
=> #<User id: 4, name: "Celeste Layne", email: "celeste.layne@hotmail.com", created_at: "2019-02-22 00:52:05", updated_at: "2019-02-22 00:52:05", password_digest: "$2a$10$ivsPxtrCMdvUxxFRik9YbexHHcHdNNPxsYB2gohzduH...">

# this email is incorrect
>> User.find_from_credentials('celeste.layne@gmail.com', 'epiphany')
=> nil

# this password is incorrect
>> User.find_from_credentials('celeste.layne@hotmail.com', 'bananas')
=> nil
```

---

### Create Users Resource
In the `config/routes.rb` file, add the following resources:

```
$ resources :users, only: [:new, :create, :index, :show]
```
then, run `rails routes` in Terminal:

| Prefix    | Verb   | URI Pattern | Controller#Action | Used for |
|-----------|--------|-------------|-------------------|------------------------------------|
| users     | GET    | /users      | users#index       | display a list of all users
|           | POST   | /users      | users#create      | create a new user, '/users'
| new_user  | GET    | /users/new  | users#new         | return an HTML form for creating a new user, '/signup'
| user      | GET    | /users/:id  | users#show        | display a specific user

---

### Create users controller
Create a users controller:

```
$ rails g controller users new create
```
and add the new (to render the signup form) and create (for receiving the form and creating a user with the forms' parameters) actions:

```
# app/controllers/users_controller.rb

class UsersController < ApplicationController

    def new
    end

    def create
    end  

end
```

Now create the view where we place the signup form. _Remember:_ view files are placed inside a folder with the same name of the controller and named for the actions they render.

```ruby
# app/views/users/new.html.erb

<h1>Sign Up / Register</h1>

<%= form_for :user do |f| %>

  Name: <%= f.text_field :name %>
  Email: <%= f.text_field :email %>
  Password: <%= f.password_field :password %>
  Password Confirmation: <%= f.password_field :password_confirmation %>
  <%= f.submit "Submit" %>

<% end %>
```

Add logic to the create action and add the private user_params method to sanitize the input from the form.

```ruby
# app/controllers/users_controller.rb

class UsersController < ApplicationController

    def new
       @user = User.new
    	render :new
    end

    def create
		@user = User.new(user_params)
		if user.save
			sign_in(@user)
	      flash[:notice] = 'You are signed in!'
	      redirect_to users_path
		else
			redirect_to '/users/new'
		end
    end  
    
    private
    
    def user_params
  		params.require(:user).permit(:name, :email, :password, :password_confirmation)
	 end

end
```
---
## SESSIONS

With Rails the user can easily get access to their session with the [session object](). We can reference session inside of any controller. It behaves like a hash which means we can easily set and retrieve serialized data on it.

### Sign-in

Let's define ApplicationController#sign_in so that we can sign in a user from any where in the app. Assuming we have a session_token column on the users table, let's set a token on the session to be a random string and set that same string as the session_token on the user.

```ruby
class ApplicationController < ActionController::Base
  # ...
  def sign_in(user)
    token = SecureRandom.urlsafe_base64 # random 22-char string
    session[:session_token] = token
    user.update_attribute(:session_token, token)
  end
```

### Create Sessions Resource
In the `config/routes.rb` file, lets add actions for signing in and logging out:

```
$ resource :session, only: [:new, :create, :destroy]
```
then, run `rails routes` in Terminal:

| Prefix    | Verb   | URI Pattern | Controller#Action | Alias
|-----------|--------|-------------|-------------------|---------------|
|           | POST   | /session      | sessions#create      | '/login'
| new_session  | GET    | /session/new  | sessions#new         | '/login'
| session      | DELETE    | /session  | sessions#destroy     | '/logout'

---

### Create sessions controller

Create a sessions controller. This is where we create (login) and destroy (logout) sessions.

```
$ rails g controller sessions name email
```

Now, inside the SessionsController we will define the `new` and `create` methods:

```
# app/controllers/sessions_controller.rb

class SessionsController < ApplicationController

	def new
		@user = User.new
		render :new
   end

   def create
		user = params[:user][:email]
		user = params[:user][:password]
		
		# using the find_from_credentials method from the User model
		user = User.find_from_credentials(email, password)
		
		if user.save
			session[:user_id] = user.id
			flash[:notice] = "Hello, #{email}! You are now signed in."
			redirect_to '/'
		else
			flash[:error] = "email or password incorrect"
			@user = User.new(email: email)
			render :new
		end
   end  
    
   private
    
   def user_params
  		params.require(:user).permit(:name, :email, :password)
	end

end
```

### Finding the current user

We define the `current user` method in the ApplicationController. We should probably **cache** current_user so that we don't have to do a query every time we call current_user.

```
class ApplicationController < ActionController::Base
  ...
  def current_user
  	 # if @current_user is nil then call find_current_user method
  	 # and assign it to the @current_user
  	 # else don't have to call the find_current_user method 
    @current_user ||= find_current_user
  end
  
  def find_current_user
    token = session[:session_token]
    token && User.find_by(session_token: token)
  end
```

### Ensuring user is signed in
If we want to ensure a user is signed in before visiting a page we can define a method in our `ApplicationController`.

```
def ensure_signed_in
    return if current_user
    flash[:error] = 'you must be signed in to see this'
    redirect_to :root
end
```
Then, in the `UsersController ` we can call our method with [before_action](https://apidock.com/rails/v4.0.2/AbstractController/Callbacks/ClassMethods/before_action):

```
class UsersController < ApplicationController

  before_action :ensure_signed_in, only: [:show, :index]
  
  ...
end
```
Now the client will be redirected if they are not signed in when hitting any route inside the `UsersController`.

### Signing out

When we are ready to sign out a user, we can simply delete the token from the session and remove the token from the database.

```
class ApplicationController < ActionController::Base
  ...
  def sign_out
    session.delete(:session_token)
    current_user.update_attribute(:session_token, nil)
  end
```

To sign out we can add a log-out button to the app:

```
<%= button_to 'Log out', session_path, method: :delete %>
```

In the SessionsController, define the destroy action:

```
  def destroy
  	 # call the sign_out method defined in the ApplicationController
    sign_out
    flash[:notice] = 'You signed out!'
    # redirects to the login form
    redirect_to new_session_path
  end
```

### Ensuring user is logged out

If there are routes you need to be signed out in order to visit:

```
  def ensure_signed_out
    return unless current_user
    flash[:error] = 'you are already signed in'
    redirect_to users_path
  end
```

For example, let's assume you need to be signed out to create a new user. We can hit our ensure_signed_out method:

```
class UsersController < ApplicationController

  before_action :ensure_signed_out, only: [:new, :create]

  ...

```

### Using current_user

Always assume you will have malicious users. Just because you don't have a link to something doesn't mean a user can't hit that action. Keep this in mind when writing your controllers.

For example, assume a User has_many :posts and a Post belongs_to :user. If we only want to show a Post to the user it belongs to, our controller should not look like this:

```
class PostsController < ApplicationController
  ...
  def show
    @post = Post.find(params[:id])
  end
```

This would allow any user to see any Post. Instead, do something like this:

```
class PostsController < ApplicationController
  ...
  def show
    @post = current_user.posts.find(params[:id])
  end
```

This will ensure you are only retrieving a Post that belongs to the current_user

### Using current_user in views

If you want to reference `current_user` in the views, use the [`helper_method`](https://apidock.com/rails/ActionController/Helpers/ClassMethods/helper_method) method:

```erb
class ApplicationController < ActionController::Base
  ...
  helper_method :current_user
  ...
```

Then in a view you can do something like:

```erb
<% if current_user %>
  Hi, <%= current_user.name %>
<% end %>
```

## Important routes

These are important auth-related routes

| Route | Method | Controller | Use |
|---|---|---|---|
| `/session/new` | `GET` | SessionsController | Renders log in form |
| `/session` | `POST` | SessionsController | Logs in user ("creates" a session)|
| `/session` | `DELETE` | SessionsController | Logs out user ("deletes" a session) |
| `/users/new` | `GET` | UsersController | Renders new user form |
| `/users` | `POST` | UsersController | Creates a user |

## Important methods

All of these methods are defined in `ApplicationController` (thus accessible by any controller)

| Method | Use |
| --- | --- |
| `current_user` | Current signed in user. `nil` if not signed in |
| `sign_in(user)` | Signs in a given `user` (adds session token) |
| `sign_out` | Signs out `current_user` (removes session token) |
| `ensure_signed_in` | Redirect unless user is signed in |
| `ensure_signed_out` | Redirect unless user is signed out |

## LAB

Let's continue building this thing:

> Note: Following these steps on your own may require brain-power. You cannot just copy and paste.

* `rails new MyFirstRailsAuthApp --database=postgresql`
* Create a `User` model: `rails g model User`
  - `username` (with index), `session_token` (with index), `password_digest`
* `rails db:create`, `rails db:migrate`
* Add `User` `username` and `password` validations
* Define `sign_in`/`sign_out`, `current_user` methods
* Define `ensure_signed_in`/`ensure_signed_out` methods
* Create a controller: `rails g controller users`
  - `index` (new), `new` (view), `create`, `show` (view)
  - `resources :users, only: [:new, :create, :index, :show]`
* Render `flash` messages in `application.html.erb`
* Create a controller: `rails g controller sessions`
  - `new` (view), `create`, `destroy`
  - `resource :session, only: [:new, :create, :destroy]`
* Create a `Post` model: `rails g model Post`
  - `name`, `description`, `user_id` (with index)
* Add `Post`-`User` associations
* `rails g controller posts`
  - `new`, `create`, `index`, `show`, `destroy`, `edit`, `update`
  - `resources :posts`