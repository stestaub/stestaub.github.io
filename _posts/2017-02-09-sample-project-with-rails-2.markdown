---
layout: post
title:  "Create a simple Photoblog with Ruby on Rails - Part 2"
date:   2017-02-09 20:57:37 +0100
categories: rails coding devise
---
In the last section, we created a simple webapp, with just a welcome page and deployed it to heroku. In this part we're going
to cover authentication, which is something you'll need in most webapplications.

# Devise is your friend
As often, when it comes to developing something that is commonly used, there is a gem for that. In the case of authentication,
its [Devise](https://github.com/plataformatec/devise). If you don't want to build authentication from scratch, you definitely are going to
use Devise. It brings all the functionality you need, from sign up to over recovery and still allows total customisation.

There are a lot of good video tutorials about devise on [Railscast](http://railscasts.com/episodes?utf8=%E2%9C%93&search=devise). Even if they are outdated,
it still gives you a good introduction to devise.

# Integrate Devise

Integrating devise is very easy. Lets start with adding the gem to our Gemfile:

{% highlight ruby %}
# ...
gem 'devise', '~> 4.2.0'
# ...
{% endhighlight %}

and run `bundle install` or just `bundle`, which is short for `bundle install`

A lot of rails libraries bring their own generators to help you set them up. So does devise. Run

```
rails generate devise:install
```

to create the initializer for devise thats used to configure its settings. You find that file in `config/initializers/devise.rb`

Next we going to create our user model, which holds the username (by default its an email address) and the password. For this,
there is also a generator, which creates our model named User and also adds a lot of other helpful stuff we will cover later.

```
rails generate devise User
```

Now its time that we need the database. As we didn't create one so far, we need to do that first. This command will create
two sqlite database files in the `db/` folder, one for development, and one for test.

```
rails db:create
```

The command before created a migration, which you can find in `db/migrations`. It creates the database table with all required columns for our
user model. To migrate the database, we can simply rub `rails db:migrate` and will will run all pending migrations.

When we restart our application and navigate to `localhost:3000` there will not be much difference yet.

Lets have a look what the generators actually created for us.

## The User model
First we will have a look at the model class.

app/models/user.rb
{% highlight ruby %}
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable
end
{% endhighlight %}

The generated class inherits from ApplicationRecord, which inherits from ActiveRecord::Base. All models that map to a database table, will inherit
from ActiveRecord::Base, which brings all the cool stuff of rails with it. ActiveRecord is the OR Mapper which follows the equally named (pattern)[https://en.wikipedia.org/wiki/Active_record_pattern].

It allows you to create save load update and delete records directly through the models class and instance methods, without having to write a single line of code.

So with this User model we can already do stuff like

{% highlight ruby %}
# Create a new user record and store it to the database. The attributes are the ones provided by rails
User.create(email: 'test@sample.com', password: 'password', password_confirmation: 'password')

# Load the first user from the database
user = User.first

# Find the first user that matches the given email
user = User.find_by(email: 'test@sample.com')

# delete the user from the database
user.destroy
#...

# call exit to leave the console
exit
{% endhighlight %}

You can try this by starting the rails console by typing `rails c` which starts the whole application in the commandline.

# Allow users to sign up
The next thing we want to have is some sort of a sign up page. Actually, devise already provides us with that. If we go to
the routes file in `config/routes.rb` we can see that the generator added the following line of code:

{% highlight ruby %}
devise_for :users
{% endhighlight %}

To see which routes that are available to our application, call `rails routes` on the command line (not the rails console). You should see the following output:

```
                  Prefix Verb   URI Pattern                    Controller#Action
        new_user_session GET    /users/sign_in(.:format)       devise/sessions#new
            user_session POST   /users/sign_in(.:format)       devise/sessions#create
    destroy_user_session DELETE /users/sign_out(.:format)      devise/sessions#destroy
       new_user_password GET    /users/password/new(.:format)  devise/passwords#new
      edit_user_password GET    /users/password/edit(.:format) devise/passwords#edit
           user_password PATCH  /users/password(.:format)      devise/passwords#update
                         PUT    /users/password(.:format)      devise/passwords#update
                         POST   /users/password(.:format)      devise/passwords#create
cancel_user_registration GET    /users/cancel(.:format)        devise/registrations#cancel
   new_user_registration GET    /users/sign_up(.:format)       devise/registrations#new
  edit_user_registration GET    /users/edit(.:format)          devise/registrations#edit
       user_registration PATCH  /users(.:format)               devise/registrations#update
                         PUT    /users(.:format)               devise/registrations#update
                         DELETE /users(.:format)               devise/registrations#destroy
                         POST   /users(.:format)               devise/registrations#create
              home_index GET    /home/index(.:format)          home#index
                    root GET    /                              home#index
```
So we see that rails actually created a lot of routes for us.

So lets add a button on our home page to navigate to the registration route named `new_user_registration`. Our home view should now look like this:

app/views/home/index.html.erb
{% highlight erb %}
<div class="jumbotron">
  <h1>Awesome Photoblog</h1>
  <p>Welcome to our Awesome Photoblog. Sign up now to explore and share your best snapshots!</p>
  <%= link_to "Sign Up", new_user_registration_path, class: 'btn btn-primary btn-lg' %>
</div>
{% endhighlight %}

As we can see, we add dynamic content to our views with the tags `<%= %>` which is [erb](http://apidock.com/ruby/ERB) syntax.
The `link_to` method is a helper provided by rails which obviously generates link tags. If we now open our application again,
we see a nice button that says "Sign Up" and clicking on that will...

...oh cool, we already have a sign up page as well... and a login page. That's awesome. And if I sign up? Well it just takes me back to
the Homepage and now I can't open the signup page again.

That is because you've actually signed up and are logged in now. If you go to the logs you can see that it created a user record in the database.

So lets do something about it that we can see we're signed in. First of all, we only want to have a sign up link when we are not logged in. So
we wrap our button inside a condition where we check if a user is signed in. The method we use was generated by devise, when we generated the user.
That's also why `signed_in?` is prefixed with `user`. If we would have called our model **Admin**, this method would be called `admin_signed_in?`

As you may noticed, this also applies to the routes.

app/views/home/index.html.erb
{% highlight erb %}
<% unless user_signed_in? %>
    <%= link_to "Sign Up", new_user_registration_path, class: 'btn btn-primary btn-lg' %>
<% end %>
{% endhighlight %}

To be able to log out, we are going to add a logout link to the header.

app/views/layouts/application.html.erb
{% highlight erb %}
<div class="container-fluid">
...
    <% if user_signed_in? %>
        <ul class="nav navbar-nav navbar-right">
          <li><%= link_to 'Sign out', destroy_user_session_path, method: :delete %></li>
        </ul>
        <p class="navbar-text navbar-right">Signed in as <%= current_user.email %></p>
    <% else %>
        <ul class="nav navbar-nav navbar-right">
          <li><%= link_to 'Sign in', new_user_session_path %></li>
        </ul>
    <% end %>
</div>
{% endhighlight %}

We now have a fully working sign up, sign in and sign out on our page.

Let's commit this to git.
```
git add .
git commit -am "Added signup and login"
```

This is the result of Chapter 2:

![start page when signed in]({{ site.baseurl }}/assets/img/signed_in_page.png "Page when signed in")

[Part 3]({% post_url 2017-02-09-sample-project-with-rails-3 %})