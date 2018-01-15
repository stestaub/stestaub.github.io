---
layout: post
title:  "Rails Interactors and Transacions"
date:   2017-09-21 12:00:00 +0100
categories: rails interactor transactions
---

In this post we're going to explore a little bit the rails interactor pattern. We
will have a look at what interactors are, how they can be implemented with
interactor-rails gem and how transactions can be handled

# The interactor pattern
Of course, there is a lot more to interactors than what I describe here, but
essentially, you can think of an Interactor as a class that handles one UseCase
of your application.

Imagine you have a UserController and a user record should be updated. As you know
in rails this is quite easy:

```ruby
def update
    @user = User.find params[:id]
    if @user.update_attributes(user_params)
        #...
    else
        #...
    end
end
```

Now let's assume, we want to notify the users friends by email after an update.
We have multiple options to do so:

Simply put the code in the controller:
```ruby
def update
    @user = User.find params[:id]
    if @user.update_attributes(user_params)
        @user.friends.each do |friend|
           NotificationMailer.friend_updated.deliver(@user, friend)
        end
    else
        #...
    end
end
```

This would bloat up our code pretty fast and we all know controller actions
should be kept simple. So put it into the model:

```ruby
def update
    @user = User.find params[:id]
    if @user.update_attributes(user_params)
        @user.notify_friends
    else
        #...
    end
end
```

But at some point, the model will become way to large as well. That is where
the interactor comes into play. That could look something like this:

```ruby
def update
    result = UpdateUser.call id: params[:id], params: user_params
    if result.success?
        #...
    else
        #...
    end
end
```

So you basically write an Interactor for each of your usecases. This
has the big advantage of being well testable and it will keep the controllers
AND the models clean of business logic.

A handy way to implement interactors is the [interactor-rails](https://github.com/collectiveidea/interactor-rails) gem, which we
will use in the upcoming examples. Thanks to [collective idea](https://collectiveidea.com/) for the cool gem.

## Side note on naming of Interactors
For some reasons, interactors are named after what they do as a Verb. This
is basically not consistent with other principles of object oriented programming,
where we say, nouns fo classes, verbs for methods.

For the sake of consistency with other examples of interactors, we stick with
the verbs for interactor names. But you could of course also give them
other names such as: `UserUpdateInteractor` instead of `UpdateUser`.

In my opinion, I would prefere nouns in a new project.

# Interactor Transactions
This section is based on the usage of the interactor gem. 
Interactors can be chained together by using `Interactor::Organizer`. Thats pretty cool, but what happens
if one of the Interactors fail?

Lets have a look at the following example: We have `UpdateUserInteractor` and a `NotifyFriendsInteractor`.
Now lets assume, we do not want to update the user if notifing fails.

You could do that by adding 