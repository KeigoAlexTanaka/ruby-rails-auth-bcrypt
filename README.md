# Rails Authentication

This tutorial is for adding authentication to a vanilla Ruby on Rails application using `bcrypt` and `has_secure_password`.

### Create a new Rails application

To create a new Rails app, we use the `rails new` command. This sets us up with our skeleton Rails app. In the Terminal, in the directory where you want to create a project, generate a new Rails project and cd into it:

```
$ rails new my-rails-auth-project -T --database=postgresql
$ cd my-rails-auth-project
    
# -T means without Test::Unit (maybe you prefer to use rspec instead)
```

### Add Dependencies

In the Gemfile, uncomment bcrypt:
```
gem 'bcrypt', '~> 3.1.7'
```
and from the root of the application, run the following command:
```
$ bundle install
```

### Create Users and Posts Resources
In the `config/routes.rb` file, add the following resources:
```
$ resources :users :posts
```
then, run `rails routes` in Terminal:
| Prefix    | Verb   | URI Pattern | Controller#Action |
|-----------|--------|-------------|-------------------|
| users     | GET    | /users      | users#index       |
|           | POST   | /users      | users#create      |
| new_user  | GET    | /users/new  | users#new         |
| edit_user | GET    | /users/:id  | users#edit        |
| user      | GET    | /users/:id  | users#show        |
|           | PATCH  | /users/:id  | users#update      |
|           | PUT    | /users/:id  | users#update      |
|           | DELETE | /users/:id  | users#destroy     |
### Setup the database
Set up an empty database:
```
$ rails db:create
```
Open the project in the current directory using your text editor:
```
$ subl .
```
Start the server and open up the project in the browser, `localhost:3000`:
```
$ rails server
```

### Create User and Post models 
Create a User model with a name and email attributes as well as the optional string parameters.
```
$ rails g model User name:string email:string

# run the migration to update the database with this change
$ rails db:migrate
```
**Remember:** in Rails, models are singlular and capitalized. Controllers and routes are plural and lowercase.

Let's look at the code inside the `app/models/User.rb` file:
```
class User < ApplicationRecord
end
```
Then, step into the Rails console to create a new User:
```
$ rails c

# create a new user object
>> User.new
```

### User Validations
Validating the presence of a name attribute.
```
class User < ApplicationRecord
    validates :name, presence: true
end
```
Validating the presence of a email attribute.
```
class User < ApplicationRecord
    validates :name, presence: true
    validates :email, presence: true
end
```
Putting a limit on the length of the characters in the fields:
```
class User < ApplicationRecord
    validates :name, presence: true, length: { maximum: 50 }
    validates :email, presence: true
end
```
Validating the uniqueness of email addresses:
```
class User < ApplicationRecord
    validates :name, presence: true, length: { maximum: 50 }, uniqueness: true
    validates :email, presence: true
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
    add_index :users, :email, unique: true
  end
end
```
_Note:_  the option `unique: true` enforces uniqueness

Don't forget to migrate:
```
$ rails db:migrate
```
### Passwords
Including `has_secure_password` adds the following functionality to your application:
  -  to save a securely hashed password_digest attribute to the database
  -  presence validation upon object creation and a validation that the passwords match 
  -  authenticate method that returns the user when the password is correct (otherwise, it returns false)

The only requirement for `has_secure_password` to work its magic is for the corresponding model to have an attribute called password_digest.
Let's generate a migration to add the password_digest column to the users table:
```
$ rails generate migration add_password_digest_to_users password_digest:string
```
The migration will look like the following:
```
# db/migrate/[timestamp]_add_password_digest_to_users.rb
class AddPasswordDigestToUsers < ActiveRecord::Migration
  def change
    add_index :users, :email, unique: true
  end
end
```
To apply it, we just migrate the database:
```
$ rails db:migrate
```
It's at this point, `has_secure_password` uses bcrypt to hash (encrypt?) the password.

```
class User < ApplicationRecord
    validates :name, presence: true, length: { maximum: 50 }, uniqueness: true
    validates :email, presence: true
    
    has_secure_password
end
```
Add a presence validation and minimum length to ensure nonblank passwords, to the password:
```
class User < ApplicationRecord
    validates :name, presence: true, length: { maximum: 50 }, uniqueness: true
    validates :email, presence: true
    
    has_secure_password
    validates :password, presence: true, length: { minimum: 6 }
end
```
### Create users and posts controllers
Create a users controller
```
$ rails g controller users name email
```
