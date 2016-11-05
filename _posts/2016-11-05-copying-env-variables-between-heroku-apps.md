---
layout: post
title:  Copying env variables between Heroku apps
date:   2016-11-05 19:30:30 +0300
---

#### This ruby script will be useful when you need to move a lots of ENV variables between two heroku apps.

#### You must be logged in heroku CLI, to successfully run this script.

#### In DEFAULT_ENV constant in script you can set ENVs which you don't want to copy.

![Heroku ENV](http://imgur.com/qtk3xar.png "Heroku ENV")

## Usage:

{% highlight shell %}
  ruby heroku_env.rb
  'Enter app name with your env variables, e.g: my-blog-api, twitter-clone-pr-23'
  my-blog-app
   'Enter app name where you want to copy env variables, e.g: my-new-blog, my-new-twitter-clone-25'
  my-new-blog

  Setting TWITTER_API_KEY, INSTAGRAM_API_KEY and restarting ⬢ my-new-blog... !
{% endhighlight %}

{% highlight ruby %}
class HerokuEnv
  def self.call(from_app, to_app)
    new(from_app, to_app).send(:copy)
  end

  private

  # This ENVs will not be copied to destination app

  DEFAULT_ENV = %w(DATABASE_URL STAGING_DATABASE_URL).freeze

  # @from_app = 'my-blog-pr-25'
  # @to_app   = 'my-blog-pr-26'

  attr_reader :from_app, :to_app

  def initialize(from_app, to_app)
    @from_app  = from_app
    @to_app    = to_app
  end

  def copy
    envs = fetch_envs_from_app
    envs = envs.split("\n").drop(1)
    text = format_variable(envs).join(' ')

    add_envs_to_heroku(text)
  end

  def add_envs_to_heroku(text)
    heroku_url = format_host_name(to_app)
    command    = heroku_url.gsub('config', "config:set #{text}")

    system(command)
  end

  def fetch_envs_from_app
    command = format_host_name(from_app)

    system(command)
    `#{command}`
  end

  def format_host_name(app)
    "heroku config --app #{app}"
  end

  def format_variable(envs)
    vars = []

    envs.each_slice(1) do |env|
      str = env.join.delete(' , []').split(':').join('=')

      vars << str
    end

    remove_defaults(vars)
  end

  def remove_defaults(envs)
    envs.reject { |s| DEFAULT_ENV.include?(s.split('=')[0]) }
  end
end

puts 'Enter app name with your env variables, e.g: my-blog-api, twitter-clone-pr-23'

app_name = gets.chomp

puts 'Enter app name where you want to copy env variables, e.g: my-new-blog, my-new-twitter-clone-25'

app_name2 = gets.chomp

HerokuEnv.call(app_name, app_name2)
{% endhighlight %}
