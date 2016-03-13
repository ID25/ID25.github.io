---
layout: post
title:  Denshobato - Private messaging between models (PART 3)
date:   2016-03-02 03:00:30 +0300
---

![Chat Panel Denshobato](http://i.imgur.com/NuhMPrg.png "Denshobato")

### Create messaging system between Reseller and Customer.

[Denshobato Chat Panel Github Repository](https://github.com/ID25/denshobato_chat_panel){:target="_blank"}

### Part 3

In [PART 1]({% post_url 2016-03-01-make-private-dialog-between-reseller-and-customer-part-1 %}) we built a basic app with reseller and customer models, devise authentication and index templates to list our models.

In [PART 2]({% post_url 2016-03-02-make-private-dialog-between-reseller-and-customer-part-2 %}) we did the most of our app, made users able to send messages to each other, create conversations, add conversations to trash and add other users to blacklist.

In Part 3 we'll install an additional plugin to our conversation, which adds a chat panel.

![Chat Panel Denshobato](http://i.imgur.com/0sUUfDl.jpg "Chat Panel Denshobato")


We start with adding this gem to our Gemfile

{% highlight ruby %}
gem 'denshobato'
gem 'denshobato_chat_panel'
{% endhighlight %}

run ```bundle install```


Next, run generator

{% highlight bash %}
rails g denshobato_chat_panel:install
{% endhighlight %}

* Add ```denshobato_chat_panel``` method to your messagable models

{% highlight ruby %}
class Customer < ActiveRecord::Base
  denshobato_for :customer
  denshobato_chat_panel
end
{% endhighlight %}

* Copy this line to your ```config/initializers/assets.rb```

{% highlight ruby %}
Rails.application.config.assets.precompile += %w( denshobato.js )
{% endhighlight %}

* In your ```application.scss``` import css

{% highlight scss %}
@import 'denshobato';
{% endhighlight %}

* In ```layouts/application.erb``` include javascript file in the bottom

{% highlight erb %}
<body>
  <div class='container'>
    <%= render 'layouts/header' %>
    <%= yield %>
  </div>

<%= javascript_include_tag 'denshobato' %>
</body>
{% endhighlight %}

* Mount API route in your ```routes.rb```

{% highlight ruby %}
mount Denshobato::DenshobatoApi => '/'
{% endhighlight %}


* On the page with your conversation (```show``` action), e.g ```localhost:3000/conversation/32```, add this helper with arguments

{% highlight slim %}
= render_denshobato_messages(@conversation, current_user)
// =>  When @conversation = Denshobato::Conversation.find(params[:id])
// => and current_user is your signed in user, e.g Devise current_user etc.
{% endhighlight %}

Now, your conversation page should look like this:
![Chat Panel Denshobato](http://i.imgur.com/PWbQTcx.png "Denshobato Chat Panel")

By default, your display name is the name of your model, your avatar is default gravatar image.

Let’s rewrite these methods.

We need to show full name of our Seller and Customer. We expect that your models have a first_name and last_name fields, same with image.

{% highlight ruby %}
class Customer < ActiveRecord::Base
  denshobato_for :customer
  denshobato_chat_panel

  def full_name
    "#{first_name} #{last_name}"
  end

  def image
    # avatar.url is a CarrierWave method to show url for uploaded image

    avatar.url
  end
end
{% endhighlight %}

Now it looks better.
![Chat Panel Denshobato](http://i.imgur.com/NgNKZwI.png "Denshobato Chat Panel")

That's all, thanks for reading.

[Denshobato Chat Panel Github Repository](https://github.com/ID25/denshobato_chat_panel){:target="_blank"}
