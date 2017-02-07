---
layout: post
title:  "Create a simple Photoblog with Ruby on Rails"
date:   2017-02-07 21:57:37 +0100
categories: rails coding
---
# What it's about
In the following sections we will create a Sample Application with Ruby on Rails. The goal is to get an overview over the Ruby Language and the Concepts of Rails and some of the common Gems. It tries to cover a broad field in a short time.

# What it’s NOT about
This tutorials aim is NOT to give a deep dive and full understanding of the Rails Framework and the Ruby language. You should not expect to understand all the details. Each part references to other resources to get a better deepdive.


# Other good resources to learn RoR
The following is probably one of the best Resources for getting started with Ruby on Rails. If you really want to learn rails, take a few days, and go through the tutorial, It’s free!

*RUBY ON RAILS TUTORIAL* by Micheal Hartl

[https://www.railstutorial.org/book](https://www.railstutorial.org/book)

There are also very good video tutorials by Ryan Bates. Although they are out of date, most of them are still worth watching:

[http://railscasts.com/](http://railscasts.com/)

Always a good source for information are the official guides of ruby on rails:

[http://guides.rubyonrails.org/](http://guides.rubyonrails.org/)

Last but not least, a very good practice to learn the concepts of rails is to read other people’s code. There is an incredible amount of gems (ruby libraries), which are freely available on github.


# Our Application
Our demo application will be a simple photoblog where users can do the following:

* Sign up with an email address and display name
* Upload multiple photos and give it a name
* Browse all available photos
* Comment on a photo

# C’mon, let’s code!
To start off, we have some prerequisites, first of all we should have ruby installed on our system (which is in this case a unix based system, believe me, developing Ruby on a Windows machine is no fun).
Setup the development environment

There are three options to have ruby installed:

1. Use the systems default ruby (this is actually not an option as you will need different version of ruby for different projects)
2. Use [RVM](https://rvm.io/rvm/install)
3. Use [rbenv](https://github.com/rbenv/rbenv), together with [ruby-build](https://github.com/rbenv/ruby-build#readme)  -> *Recommended!*

Follow the instructions to install rbenv and ruby, and set the global version for ruby.

Once we have setup your ruby environment, we need two more things to get started with our rails project, that’s bundler (which is the dependency manager for ruby) and rails.

To do so, just go to the console and type:

```
gem install bundler
gem install rails
```

`gem` is the package manager for ruby. These commands will install the libraries bundler and rails for the current ruby version we’ve selected
Creating the project
Now we are ready to get started with Rails! In the console, navigate to the directory you want to have your new project and typ:

rails new awesome-photoblog --skip-test-unit

This will create a new folder with a complete setup for your new application. With --skip-test-unit we skipped the default setup for tests, because we will use rspec later on as our testing framework. You will get a lot of output on the console, informing you what the script is doing.

When it has finished we can cd into our newly created project:

```
cd awesome-photoblog/
```

And start up the rails server (which is, since rails 5, the puma webserver)

```
rails server
```

When we open the browser and type localhost:3000 we see a welcome page when everything is working:

![rails start page]({{ site.url }}/assets/img/rails_start.png "Rails start page")


Awesome right? We already have a working rails application. But now let’s add some basic stuff that will help us a lot while developing.

### Which editor should I use?
Well, as always, you need some kind of editor. Basically a simple texteditor is enough, you could use sublime-text for example. If you prevere a fully featured IDE, Ruby Mine (IntelliJ for ruby) is probably your best choice.

## Before we start coding

I highly recommend using a version control for every project you create. It allows you to always go back if you messed up. Additionally we need it as soon as we gonna
deploy the application.

So lets initiate a new [git](https://git-scm.com/) repository and commit our current version:

```
git init
git add .
git commit -am "initiated new rails application"
```

## Creating a welcome page
Let’s start with creating a welcome page. We can do this by creating a new Controller (which are responsible to handle the request and initiate view rendering). We call this one home_controller and it should have a index action that renders an index view.
We can use the rails built in generators to automatically bootstrap the code for us:

```
rails g controller Home index
```

The command **g** is the short version for generate. **controller** is the generator we want to use, **Home** the name of our controller, followed by the actions it should have.
This command creates a new file `app/controllers/home_controller.rb` and automatically adds a route.

We can access this page via [http://localhost:3000/home/index](http://localhost:3000/home/index), but the root page still shows
the default rails page. We can change that in the routes file:

config/routes.rb
{% highlight ruby %}
Rails.application.routes.draw do
  # The route added by the generator
  get 'home/index'

  # For details on the DSL available within this file, see http://guides.rubyonrails.org/routing.html

  # set our root path to the index action on the home_controller
  root to: 'home#index'
end
{% endhighlight %}

### Add some styling and content to our page

For this purpose we gonna add bootstrap.