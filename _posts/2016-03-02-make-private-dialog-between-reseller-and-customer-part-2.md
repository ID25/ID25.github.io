---
layout: post
title:  Denshobato - Private messaging between models (PART 2)
date:   2016-03-02 03:00:30 +0300
---

![alt text](http://i.imgur.com/NuhMPrg.png "Denshobato")

### Create messaging system between Reseller and Customer.

[Denshobato Github Repository](https://github.com/ID25/denshobato){:target="_blank"}

### Part 2

In [PART 1]({% post_url 2016-03-01-make-private-dialog-between-reseller-and-customer-part-1 %}) we built a basic app with reseller and customer models, devise authentication and index templates to list our models. In part 2 we’ll make the rest of our app.

### In Part 2:

* We'll create conversations
* Send messages to conversations
* Send conversations to Trash
* Add users to BlackList

Go to Reseller and Customer Models.

Add ```denshobato_for :your_class``` to these models.
This method does a lot of things - sets up associations and adds useful methods for you.

{% highlight ruby %}
denshobato_for :your_class
{% endhighlight %}

Add it to your models.

{% highlight ruby %}
class Reseller < ActiveRecord::Base
  denshobato_for :reseller
end

class Customer < ActiveRecord::Base
  denshobato_for :customer
end
{% endhighlight %}

Go to Resellers and Customers contoller.
Add to ```index``` action conversation builder for our form.

{% highlight ruby %}
@conversation = current_account.hato_conversations.build
{% endhighlight %}

Add this form to your index page, which shows all your resellers and customers.
{% highlight slim %}
// => resellers/index.html.slim

- @resellers.each do |reseller|
  = link_to reseller.email, reseller if current_account != reseller
  = form_for @conversation, url: :conversations do |form|
    = fill_conversation_form(form, reseller)
    = form.submit 'Start Conversation', class: 'btn btn-primary'
  hr
{% endhighlight %}

```fill_conversation_form``` is a Denshobato view helper, it helps to create a conversation.

Don't forget to add route for conversations resource ```:conversations```

On a reseller's page (if you're signed as a reseller) you can see an extra button for creating conversation with yourself. There is also a useful helper which can hide this button.

{% highlight slim %}
- if can_create_conversation?(current_account, reseller)
    = form_for @conversation, url: :conversations do |form|
      = fill_conversation_form(form, reseller)
      = form.submit 'Start Conversation', class: 'btn btn-primary'
{% endhighlight %}

Add this to both forms (for reseller and customer).

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
  redirect_to :conversations, notice: 'You can`t join this conversation' unless user_in_conversation?(current_account, @conversation)

  @message_form = current_account.hato_messages.build
  @messages     = @conversation.messages
end
{% endhighlight %}

If we go back to reseller's or customer's index page, we can still see start conversation button, even if the conversation is already started. Let`s hide it.

In our ```index.html.slim``` templates for reseller and customer, use ```conversation_exists?``` method.

{% highlight slim %}
- @resellers.each do |reseller|
  = link_to reseller.email, reseller if current_account != reseller
  - unless conversation_exists?(current_account, reseller)
    - if can_create_conversation?(current_account, reseller)
      = form_for @conversation, url: :conversations do |form|
        = fill_conversation_form(form, reseller)
        = form.submit 'Start Conversation', class: 'btn btn-primary'
  hr
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
  - if room.messages.last.present?
    p = "Last message from: #{message_from(room.messages.last, :first_name, :last_name)}"
  = button_to 'Remove Conversation', conversation_path(room),
    method: :delete, class: 'btn btn-danger'
  hr
{% endhighlight %}

It should look like this.

Of course, your models should have an image url, and at least one message in conversation

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
    = link_to "Conversation with #{interlocutor_info(room.recipient,
    :first_name, :last_name)}",
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
post :black_list, to: 'blacklists#add_to_blacklist', as: :black_list
post :remove_from_blacklist, to: 'blacklists#remove_from_blacklist', as: :remove_from_blacklist
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

We use ```define_method``` again for very similar methods.
Now, it looks as it should look. Feel free to customize it, e.g, add Mailer, when you send message.
When reseller sends message to customer, we can also send a notification to customer's email. Like a "You have a new message from John Doe..." etc. If something is unclear, read documentation on the [Denshobato  repo](https://github.com/ID25/denshobato)

Thanks for reading.
