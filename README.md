# Rails Authentication

This tutorial is for adding authentication to a vanilla Ruby on Rails application using `bcrypt` and `has_secure_password`.

The point of authentication is to ensure that the user is who they say they are. The standard way of managing this is through logging in your user using a sign-in form. Once the user is logged in, you keep track of them using the session until they log out.

User Authentication allows us to restrict access to a portion of the application so that only users who log in with a valid username and password can have access.

### [Step 1] :: Create a new Rails application

To create a new Rails app, we use the `rails new` command. This sets us up with our skeleton Rails app. In the Terminal, in the directory where you want to create a project, generate a new Rails project and cd into it:

```
$ rails new my-rails-auth-project -T --database=postgresql
$ cd my-rails-auth-project
    
# -T means without Test::Unit (maybe you prefer to use rspec instead)
```

### [Step 2] :: Setup the database
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

### [Step 3] :: Generate User model 
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

### [Step 4] :: Validations

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

### [Step 5] :: Add Dependencies

In the Gemfile, uncomment bcrypt:

```
gem 'bcrypt', '~> 3.1.7'
```
We need bcrypt to securely store passwords in our database. Then, from the root of the application, run the following command:

```
$ bundle install
```

Then, restart your server.

### [Step 6] :: Passwords

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

Validate a user given the name and password:

```
class User < ApplicationRecord
    validates :name, presence: true, length: { maximum: 50 }
    validates :email, presence: true, length: { maximum: 255 }, uniqueness: true
    
    has_secure_password
    
    def 
end
```
Now, find the user record by the email address:

```
>> User.find_by(email: "celeste.layne@hotmail.com")
=> #<User id: 3, name: "Celeste Layne", email: "celeste.layne@hotmail.com", created_at: "2019-02-22 00:09:44", updated_at: "2019-02-22 00:09:44", password_digest: "$2a$10$oulAW4mTmy4sOmLaKRdsz.rglfMyQBrUeSq4C1ZBwU5...">
```

---

### [Step ] :: Create users controllers
Create a users controller with index, new, create and show:

```
$ rails g controller users name email
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

```
# app/views/users/new.html.erb

<h1>Sign Up</h1>

<%= form_for :user, url: '/users' do |f| %>

  Name: <%= f.text_field :name %>
  Email: <%= f.text_field :email %>
  Password: <%= f.password_field :password %>
  Password Confirmation: <%= f.password_field :password_confirmation %>
  <%= f.submit "Submit" %>

<% end %>
```

Add logic to the create action and add the private user_params method to sanitize the input from the form.

```
# app/controllers/users_controller.rb

class UsersController < ApplicationController

    def new
    end

    def create
		user = User.new(user_params)
		if user.save
			session[:user_id] = user.id
			redirect_to '/'
		else
			redirect_to '/signup'
		end
    end  
    
    private
    
    def user_params
  		params.require(:user).permit(:name, :email, :password, :password_confirmation)
	 end

end
```

### [Step 3] :: Create Users Resource
In the `config/routes.rb` file, add the following resources:

```
$ resources :users, only: [:new, :create, :index, :show]
```
then, run `rails routes` in Terminal:

| Prefix    | Verb   | URI Pattern | Controller#Action | Used for |
|-----------|--------|-------------|-------------------|------------------------------------|
| users     | GET    | /users      | users#index       | display a list of all users
|           | POST   | /users      | users#create      | create a new user
| new_user  | GET    | /users/new  | users#new         | return an HTML form for creating a new user
| edit_user | GET    | /users/:id  | users#edit        | return an HTML form for editing a user
| user      | GET    | /users/:id  | users#show        | display a specific user
|           | PATCH  | /users/:id  | users#update      | update a specific user
|           | PUT    | /users/:id  | users#update      | update a specific user
|           | DELETE | /users/:id  | users#destroy     | delete a specific user


### [Step 8] :: Sessions!!!

Create a sessions controller. This is where we create (aka login) and destroy (aka logout) sessions.

```
$ rails g sessions users name email
```