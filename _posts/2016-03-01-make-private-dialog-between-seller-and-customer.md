---
layout: post
title:  Denshobato - Private messaging between models
date:   2016-03-01 03:35:30 +0300
categories: denshobato ruby rails tutorial gem
---

![alt text](http://i.imgur.com/NuhMPrg.png "Denshobato")

## Create messaging system between seller and customer.

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

$ rails g devise Seller   first_name:string last_name:string

$ rails g devise Customer first_name:string last_name:string
{% endhighlight %}

And install Denshobato

{% highlight shell %}
$ rails g denshobato:install
{% endhighlight %}

Now run ```rake db:migrate```

Next, we need to set up a basic authentication for seller and customer model.

Add this helpful method to ```ApplicationController```

{% highlight ruby %}
class ApplicationController < ActionController::Base
# ....
  private

  def current_account
    current_seller || current_customer
  end
end
{% endhighlight %}

And to ```ApplicationHelper```

{% highlight ruby %}
module ApplicationHelper
  def current_account
    current_seller || current_customer
  end

  def current_account_signed_in?
    seller_signed_in? || customer_signed_in?
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
      = link_to 'Register as Seller', :new_seller_registration,
        class: 'btn btn-primary'
      hr
      = link_to 'Sign In as Seller', :new_seller_session,
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

{% highlight slim %}
// => _header.html.slim

nav.navbar.navbar-default
  .container-fluid
    .navbar-header
      button.navbar-toggle.collapsed aria-controls="navbar"
        aria-expanded="false" data-target="#navbar" data-toggle="collapse" type="button"
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
  li = link_to 'Sellers', :sellers
  li = link_to 'Customers', :customers
{% endhighlight %}

It should look like this
![alt text](http://i.imgur.com/EERnAZ2.png "Denshobato")

Next, we create the seller's controller and customer's controller.

Don`t forget about routes
{% highlight ruby %}
resources :sellers, :customers
{% endhighlight %}

{% highlight ruby %}
class SellersController < ApplicationController
  def index
    @sellers = Seller.all
  end
end
{% endhighlight %}

{% highlight slim %}
// => sellers/index.html.slim

- @sellers.each do |seller|
  = link_to seller.email, seller if current_account != seller
  hr
{% endhighlight %}

Make the same with Customer (controller and index view)

Let's create a few sellers and customers

go to ```rails c```

{% highlight ruby %}
Seller.create(email: 'seller_john@gmail.com', password: 'password123',
password_confirmation: 'password123', first_name: 'John', last_name: 'Doe')

Seller.create(email: 'seller_steve@gmail.com', password: 'password123',
password_confirmation: 'password123', first_name: 'Steve', last_name: 'Smith')

Customer.create(email: 'customer_mark@gmail.com', password: 'password123',
password_confirmation: 'password123', first_name: 'Mark', last_name: 'Paul')

Customer.create(email: 'customer_fred@gmail.com', password: 'password123',
password_confirmation: 'password123', first_name: 'Fred', last_name: 'Munch')
{% endhighlight %}

Okay, looks like we bootstraped everything we need at this point.

We ready  to set up messaging system for our seller and customer.
Go to Seller and Customer Models.

Add ```denshobato_for :your_class``` to these models.
This method does a lot of things - sets up associations and adds useful methods for you.

{% highlight ruby %}
denshobato_for :your_class
{% endhighlight %}

Add it to your models.

{% highlight ruby %}
class Seller < ActiveRecord::Base
  denshobato_for :seller
end

class Customer < ActiveRecord::Base
  denshobato_for :customer
end
{% endhighlight %}

Go to Sellers and Customers contoller.
Add to ```index``` action conversation builder for our form.

{% highlight ruby %}
@conversation = current_account.hato_conversations.build
{% endhighlight %}

Add this form to your index page, which shows all your sellers and customers.
{% highlight slim %}
// => sellers/index.html.slim

- @sellers.each do |seller|
  = link_to seller.email, seller if current_account != seller
  = form_for @conversation, url: :conversations do |form|
    = fill_conversation_form(form, seller)
    = form.submit 'Start Conversation', class: 'btn btn-primary'
  hr
{% endhighlight %}

```fill_conversation_form``` is a Denshobato view helper, it helps to create a conversation.

Don't forget to add route for conversations resource ```:conversations```

On a seller's page (if you're signed as a seller) you can see an extra button for creating conversation with yourself. There is also a useful helper which can hide this button.

{% highlight slim %}
- if can_create_conversation?(current_account, seller)
    = form_for @conversation, url: :conversations do |form|
      = fill_conversation_form(form, seller)
      = form.submit 'Start Conversation', class: 'btn btn-primary'
{% endhighlight %}

Add this to both forms (for seller and customer).

Okay, now if we click the button, we got an error ```uninitialized constant ConversationsController```

Let`s create this contoller.

{% highlight ruby %}
class ConversationsController < ApplicationController
  def create
    @conversation = current_account.hato_conversations.build(conversation_params)
    if @conversation.save
      redirect_to conversation_path(@conversation)
    else
      redirect_to :conversations, notice: @conversation.errors
    end
  end

  def destroy
    @conversation = Denshobato::Conversation.find(params[:id])
    redirect_to :conversations if @conversation.destroy
  end

  private

  def conversation_params
    params.require(:denshobato_conversation).permit(:sender_id, :sender_type,
       :recipient_id, :recipient_type)
  end
end
{% endhighlight %}

Done, we're ready to create a form for messages.

First, add ```resources :messages``` to ```routes.rb```

In ```ConversationController```, ```show``` action add form for message.

{% highlight ruby %}
def show
  @conversation = Denshobato::Conversation.find(params[:id])
  @message_form = current_account.hato_messages.build
  @messages     = @conversation.messages.includes(:author)
end
{% endhighlight %}

We set ```url: :messages``` for correct resource path (by default it searches for ```:denshobato_messages```)

{% highlight slim %}
// => conversations/show.html.slim

= form_for @message_form, url: :messages do |form|
  = form.text_field :body, class: 'form-control'
  = fill_message_form(form, current_account, @conversation)
  = form.submit 'Send message', class: 'btn btn-primary'
{% endhighlight %}

```fill_message_form``` is a Denshobato helper, it helps you to create a message form.

To show all messages for this conversation add this to the same view below. ```show.html.slim```

{% highlight slim %}
- @messages.each do |msg|
  = msg.body
  hr
{% endhighlight %}

Okay, when we try to create a message, we got an error, we don't have ```MessagesController```, so create it.

{% highlight ruby %}
class MessagesController < ApplicationController
  def create
    conversation_id = params[:denshobato_message][:conversation_id]
    @message = current_account.send_message_to(conversation_id, message_params)

    if @message.save
      @message.send_notification(conversation_id)
      redirect_to conversation_path(conversation_id), notice: 'Message Sent'
    else
      redirect_to conversation_path(conversation_id), notice: @message.errors
    end
  end

  private

  def message_params
    params.require(:denshobato_message).permit(:body, :author_id, :author_type)
  end
end
{% endhighlight %}

Notice: Don`t forget to send_notification to conversation after you save message, it sends notification both for you and recipient, so both of you get access to messages.

Now we'll make our conversation list.

Open ```/layouts/_links.html.slim```

{% highlight slim %}
li = link_to 'Conversations', :conversations
{% endhighlight %}

Add to ```index``` action in ```ConversationContoller```

{% highlight ruby %}
def index
  @conversations = current_account.my_conversations.includes(:sender)
end
{% endhighlight %}

And in ```index``` view

{% highlight slim %}
- @conversations.each do |room|
  = link_to "Conversation with: #{room.recipient.email}", conversation_path(room)
  = button_to 'Remove Conversation', conversation_path(room),
    method: :delete, class: 'btn btn-danger'
  hr
{% endhighlight %}

Now we can list all your conversations.

It is not safe now, because now everyone have an access to your conversations. So you need to go to conversation show action, and add this line

Notice: We use ```user_in_conversation?(current_account, @conversation)``` method in this example. This method checks if current_user presents in conversations.

{% highlight ruby %}
def show
  @conversation = Denshobato::Conversation.find(params[:id])
  redirect_to :conversations, notice: 'You can`t join this conversation'
    unless user_in_conversation?(current_account, @conversation)

  @message_form = current_account.hato_messages.build
  @messages     = @conversation.messages
end
{% endhighlight %}

If we go back to seller's or customer's index page, we can still see start conversation button, even if the conversation is already started. Let`s hide it.

In our ```index.html.slim``` templates for seller and customer, use ```conversation_exists?``` method.

{% highlight slim %}
- @sellers.each do |seller|
  = link_to seller.email, seller if current_account != seller
  - unless conversation_exists?(current_account, seller)
    - if can_create_conversation?(current_account, seller)
      = form_for @conversation, url: :conversations do |form|
        = fill_conversation_form(form, seller)
        = form.submit 'Start Conversation', class: 'btn btn-primary'

{% endhighlight %}

Good, now go to conversation index page, we'll use some view helpers to show our recipient's name and avatar. You will also see the last message of a shown conversation.

{% highlight slim %}
- @conversations.each do |room|
  = interlocutor_image(room.recipient, :user_avatar, 'img-circle')
  = link_to "Conversation with: #{interlocutor_info(room.recipient,
    :first_name, :last_name)}", conversation_path(room)
  p = room.messages.last.try(:body)

  = interlocutor_image(room.messages.last.try(:author),
    :user_avatar, 'img-circle')
  p = "Last message from: #{message_from(room.messages.last, :first_name, :last_name)}"
  = button_to 'Remove Conversation', conversation_path(room),
    method: :delete, class: 'btn btn-danger'
  hr
{% endhighlight %}

It should look like this.

![alt text](http://i.imgur.com/Jr8iMzr.png "Denshobato")

Next, go into a conversation and use this helpers to make it look better.

Open ```conversation/show.html.slim``` and replace messages with formatted outp

{% highlight slim %}
- @messages.each do |msg|
  p = interlocutor_info(msg.author, :first_name, :last_name)
  = interlocutor_image(msg.author, :user_avatar, 'img-circle')
  p = msg.body
  hr
{% endhighlight %}

Great! We are almost in the end, but we have to do two more features - send conversation to trash and ignore users.

We start with trash feature.

### Trash
First, create buttons for these actions.

Again, open your conversation index view and add the button

{% highlight slim %}
- @conversations.each do |room|
	// ....
  = button_to 'Move to Trash', to_trash_path(id: room),
    class: 'btn btn-warning', method: :patch
{% endhighlight %}

Add route
{% highlight ruby %}
patch :to_trash, to: 'conversations#to_trash', as: :to_trash
{% endhighlight %}

And action
{% highlight ruby %}
def to_trash
  room = Denshobato::Conversation.find(params[:id])
  room.to_trash
  redirect_to :conversations
end
{% endhighlight %}

Add to index action our trashed conversations
{% highlight ruby %}
def index
  @conversations = current_account.my_conversations
  @trash         = current_account.trashed_conversations
end
{% endhighlight %}

And back to our index view, add this under conversations.
{% highlight slim %}
- if @trash.any?
  h1 Trash
  - @trash.each do |room|
    = link_to "Conversation with #{room.recipient.full_name}",
      conversation_path(room)
    = button_to 'Remove Conversation', conversation_path(room),
      method: :delete, class: 'btn btn-danger'
    = button_to 'Move from Trash', from_trash_path(id: room),
      class: 'btn btn-warning', method: :patch
    hr
{% endhighlight %}

As you can see, we add 'Move from Trash button'
{% highlight ruby %}
= button_to 'Move from Trash', from_trash_path(id: room),
  class: 'btn btn-warning', method: :patch
{% endhighlight %}

Define route for this action
{% highlight ruby %}
patch :from_trash, to: 'conversations#from_trash', as: :from_trash
{% endhighlight %}

Add an action for it. As longs as we have two similar actions we can DRYing it by ruby ```define_method```, like this.
{% highlight ruby %}
%w(to_trash from_trash).each do |name|
  define_method name do
    room = Denshobato::Conversation.find(params[:id])
    room.send(name)
    redirect_to :conversations
  end
end
{% endhighlight %}

Great, it works!

### BlackList
We have one last thing to do, it`s a blacklist.
You can add model to your blacklist, blocked model can't start conversation with you or send messages, and vice versa, if you want to send a message to this model, remove it from blacklist.

{% highlight slim %}
- @customers.each do |customer|
  = link_to customer.email, customer if current_account != customer
  - unless conversation_exists?(current_account, customer)
    - if can_create_conversation?(current_account, customer)
      - if user_in_black_list?(current_account, customer)
        p 'This user in your blacklist'
        = button_to 'Remove from black list',
          remove_from_blacklist_path(user: customer, klass: customer.class.name), class: 'btn btn-info'
      - else
        = button_to 'Add to black list',
          black_list_path(user: customer, klass: customer.class.name), class: 'btn btn-danger'
        = form_for @conversation, url: :conversations do |form|
          = fill_conversation_form(form, customer)
          = form.submit 'Start Conversation', class: 'btn btn-primary'
    hr

{% endhighlight %}

Looks terrible. Here is an advice for you: move some logic to helpers or decorators.

Okay, we added these two buttons - add to black list and remove, - and helpers to show or hide it.
{% highlight slim %}
- if user_in_black_list?(current_account, u)
  p 'This user in your blacklist'
  = button_to 'Remove from black list',
    remove_from_blacklist_path(user: u, klass: u.class.name), class: 'btn btn-info'
 - else
   = button_to 'Add to black list',
    black_list_path(user: u, klass: u.class.name), class: 'btn btn-danger'
{% endhighlight %}

Go to ```routes.rb``` and define routes for these actions.

{% highlight ruby %}
post :black_list, to: 'blacklists#add_to_blacklist',
  as: :black_list
post :remove_from_blacklist, to: 'blacklists#remove_from_blacklist',
  as: :remove_from_blacklist
{% endhighlight %}

Create ```BlacklistsController```

{% highlight ruby %}
class BlacklistsController < ApplicationController
  [%w(add_to_blacklist save), %w(remove_from_blacklist destroy)].each do |name, action|
    define_method name do
      user   = params[:klass].constantize.find(params[:user])
      record = current_account.send(name, user)
      record.send(action) ? (redirect_to user.class.name.downcase.pluralize.to_sym) : (redirect_to :root)
    end
  end
end
{% endhighlight %}

Now you can create page with your blacklist.
For example, page in your ```blacklists_controller```
{% highlight ruby %}
def blacklist
  @blacklist = current_account.my_blacklist
end
{% endhighlight %}

We ```define_method``` again for very similar methods.
Now, it looks as it should look. Feel free to customize it, e.g, add Mailer, when you send message.
When seller sends message to customer, we can also send a notification to customer's email. Like a "You have a new message from John Doe..." etc. If something is unclear, read documentation on the [Denshobato  repo](https://github.com/ID25/rails_emoji_picker)

Thanks for reading.
