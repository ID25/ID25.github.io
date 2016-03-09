---
layout: post
title:  Denshobato - Private messaging between models (PART 1)
date:   2016-03-01 03:35:30 +0300
---

![alt text](http://i.imgur.com/NuhMPrg.png "Denshobato")

### Create messaging system between Reseller and Customer.

[Denshobato Github Repository](https://github.com/ID25/denshobato){:target="_blank"}

### Part 1

Deshobato is a Rails gem that helps models communicate with each other. It gives simple api for creating a complete conversation system. You can create conversation with any model. Denshobato provides api methods for making conversations, messages and BlackList and Trash. It also provides Helper methods for controller and view. It has a built-in chat panel and a particular documentation and you can use it any time something is unclear.

In this tutorial, we'll create messaging system from scratch. A most part of work Denshobato gem will take over.

{% highlight bash %}
$ rails new shop
{% endhighlight %}

Add bootstrap-sass gem for styles, Devise for quick users authentication, slim-rails for templates, and Denshobato for messaging.

{% highlight ruby %}
gem 'bootstrap-sass'
gem 'devise'
gem 'slim-rails'
gem 'denshobato'
{% endhighlight %}
run ```bundle install```

Install Devise

{% highlight shell %}
$ rails g devise:install

$ rails g devise Reseller first_name:string last_name:string

$ rails g devise Customer first_name:string last_name:string
{% endhighlight %}

And install Denshobato

{% highlight shell %}
$ rails g denshobato:install
{% endhighlight %}

Now run ```rake db:migrate```

Next, we need to set up a basic authentication for reseller and customer model.

Add this helpful method to ```ApplicationController```

{% highlight ruby %}
class ApplicationController < ActionController::Base
# ....
  private

  def current_account
    current_reseller || current_customer
  end
end
{% endhighlight %}

And to ```ApplicationHelper```

{% highlight ruby %}
module ApplicationHelper
  def current_account
    current_reseller || current_customer
  end

  def current_account_signed_in?
    reseller_signed_in? || customer_signed_in?
  end
end
{% endhighlight %}

Generate ```WelcomeController```, with ```home``` action for root path.

{% highlight shell %}
rails g controller Welcome home
{% endhighlight %}

Add route to root path.
```routes.rb```
{% highlight ruby %}
root 'welcome#home'
{% endhighlight %}

Add this to ```welcome/home.html.slim```

{% highlight slim %}
h1.text-danger.text-center Welcome to the Shop!
- if current_account_signed_in?
  h2.text-success.text-center = "Hello, #{current_account.try(:email)}"
- else
  hr
  .row.text-center
    .col-md-4.col-md-offset-2
      = link_to 'Register as reseller', :new_reseller_registration,
        class: 'btn btn-primary'
      hr
      = link_to 'Sign In as reseller', :new_reseller_session,
        class: 'btn btn-primary'
    .col-md-4
      = link_to 'Register as Customer', :new_customer_registration,
        class: 'btn btn-success'
      hr
      = link_to 'Sign In as Customer', :new_customer_session,
        class: 'btn btn-success'

{% endhighlight %}

Next, add header to our application layout and import bootstrap styles, if you haven`t done it yet.

{% highlight erb %}
<div class="container">
  <%= render 'layouts/header' %>
  <%= yield %>
</div>
{% endhighlight %}

```app/assets/stylesheets/welcome.scss```

{% highlight scss  %}
  @import 'bootstrap';
{% endhighlight %}

{% highlight slim %}
// => _header.html.slim

nav.navbar.navbar-default
  .container-fluid
    .navbar-header
      button.navbar-toggle.collapsed aria-controls="navbar" aria-expanded="false" data-target="#navbar" data-toggle="collapse" type="button"
        span.sr-only Toggle navigation
        span.icon-bar
        span.icon-bar
        span.icon-bar
      = link_to 'Shop', :root, class: 'navbar-brand'
    #navbar.navbar-collapse.collapse
      - if current_account_signed_in?
        = render 'layouts/account_header'
{% endhighlight %}

As long as we have two different models, for sake of brevity, we'll use ```devise_url_helper```, which generates correct route, obviously based on your model. name.

{% highlight slim %}
// => 'layouts/_account_header.html.slim'

= render 'layouts/links'
ul.nav.navbar-nav.navbar-right
  li = link_to 'Edit Profile',
    devise_url_helper(:edit, current_account, :registration)
  li = link_to 'Log out',
    devise_url_helper(:destroy, current_account, :session), method: :delete
{% endhighlight %}

In Links partial
{% highlight slim %}
// => layouts/_links.html.slim

ul.nav.navbar-nav
  li = link_to 'Home',  :root
  li = link_to 'Resellers', :resellers
  li = link_to 'Customers', :customers
{% endhighlight %}

It should look like this
![alt text](http://i.imgur.com/EERnAZ2.png "Denshobato")

Next, we create the reseller's controller and customer's controller.

Don`t forget about routes
{% highlight ruby %}
resources :resellers, :customers
{% endhighlight %}

{% highlight ruby %}
class ResellersController < ApplicationController
  def index
    @resellers = Reseller.all
  end
end
{% endhighlight %}

{% highlight slim %}
// => resellers/index.html.slim

- @resellers.each do |reseller|
  = link_to reseller.email, reseller if current_account != reseller
  hr
{% endhighlight %}

Make the same with Customer (controller and index view)

Let's create a few resellers and customers

go to ```rails c```

{% highlight ruby %}
Reseller.create(email: 'reseller_john@gmail.com', password: 'password123',
password_confirmation: 'password123', first_name: 'John', last_name: 'Doe')

Reseller.create(email: 'reseller_steve@gmail.com', password: 'password123',
password_confirmation: 'password123', first_name: 'Steve', last_name: 'Smith')

Customer.create(email: 'customer_mark@gmail.com', password: 'password123',
password_confirmation: 'password123', first_name: 'Mark', last_name: 'Paul')

Customer.create(email: 'customer_fred@gmail.com', password: 'password123',
password_confirmation: 'password123', first_name: 'Fred', last_name: 'Munch')
{% endhighlight %}

Okay, looks like we bootstraped everything we need at this point.

We ready to set up messaging system for our Reseller and Customer.

### [PART 2]({% post_url 2016-03-02-make-private-dialog-between-reseller-and-customer-part-2 %})
