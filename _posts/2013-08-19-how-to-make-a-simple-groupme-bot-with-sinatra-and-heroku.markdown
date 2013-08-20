---
layout: post
title:  "How to make a deploy a simple GroupMe Bot with Sinatra and Heroku"
date:   2013-08-19
---

In this simple post, I'll show you how to make a very simple GroupMe bot.

Prerequisites
-------------
* [Homebrew](http://www.brew.sh) (well, not really required, but it makes installing everything very easy)
* [Ruby](http://www.ruby-lang.org/en/) (and some knowledge of it)
* A [Heroku](http://www.heroku.com) account, and [heroku-toolbelt](https://toolbelt.heroku.com/)
* A [GroupMe](http://groupme.com) account, of course

If you have homebrew, you can get ruby, and the heroku toolkit by typing:

        brew install ruby heroku-toolkit

Setting Up
==========
1. Login to (developers.groupme.com), and create a new application.
Don't worry about providing a correct callback url for now, we'll get to 
that later

2. Create a Heroku account. We'll use Heroku to deploy our bot when
we're ready. Don't worry its free. 

3. Login to Heroku:
  
		heroku login

Heroku will generate pub/private keys for you if you don't have them, and automatically upload them
so that you can deploy.

4. Create a project directory

        heroku create

Heroku has now created a new app for you, initialized an empty git repo in your current directory, and has added
Heroku's server details as a git remote called 'heroku'.

5. Heroku should spit out what the url of your hosted app will be. I'd recommend choosing a friendlier name using
**heroku rename** ie:

        heroku rename 'simple-gm-app'

6. Register your bot using steps 1-3 from the [official GroupMe bot tutorial](https://dev.groupme.com/tutorials/bots).
   Make sure you POST a callback_url in step 3. It should look something like: *simple-gm-app.herokuapp.com*   
   Example: 

        curl -X POST -d '{"bot": { "name": "Johnny Five", "group_id": "2000", \
        "callback_url": "http://simple-gm-app.herokuapp.com" } }' \
        https://api.groupme.com/v3/bots?token=token123

   Note: There is a syntax error in the curl example in step 3 of the GroupMe tutorial. There are no closing 
   braces.

Getting Started... for reals
----------------------------

First things first: we need to get Sinatra. Sinatra is a microframework for ruby 
that will make our task very easy to do. It'll handle routing, and basic HTTP.

While you could just install sinatra with "gem install sinatra", we're going to 
do this with a Gemfile instead. Heroku needs to see one of these when we deploy anyway.

Create a file called *Gemfile*, and make sure it contains the following:

{% highlight ruby %}
#Gemfile
gem 'sinatra' 
{% endhighlight %}

Install sinatra

    bundle install 

Now make a file where your bot's code will live. I'm calling mine *bot.rb*.
According to the GroupMe API Docs, every time a message is sent to the group your bot 
is registered to, GroupMe will make a POST request containing the message to the callback url you provided.
Here's an example of what GroupMe will send:

{% highlight javascript %}
{
  "text": "Hey",
  "name": "Jane",
  "group": "1234567"
}
{% endhighlight %}

So, all we need to do is read this, and respond with our own POST request back to GroupMe's 
servers. Lets start with receiving data first.

Since GroupMe will make a post request, we need to make a route that accepts POSTs.

{% highlight ruby %}
require 'sinatra'
require 'json'
#bot.rb

post '/callback_url' do
end
{% endhighlight %}

We can test this by running our server, and performing a POST using cURL.

    ruby bot.rb #This launches the server
    curl -v -X POST http://localhost:4567/callback_url -d ''

You should notice that we get an HTTP 200 OK response, which means everything is good. If you try a different route
like http://localhost:4567/i_dont_need_no_callback, you'll get a 404, because we have not defined such a route. Furthermore, if you try the same 
callback_url route with a GET instead of a POST, you'll get the same thing. We need to define routes before we can use them.

GroupMe POSTs a bunch of JSON to our route. We need to read the body of the request, and parse the JSON. 
Let me show you what I mean. 

{% highlight ruby %}
require 'sinatra'
require 'json'
#bot.rb

post '/test_json' do
    json_data = JSON.parse(request.body.read)
    puts json_data
end
{% endhighlight %}

Now I'll perform a POST with some JSON to this route, and see what sinatra spits out to the console:

    curl -v -X POST http://localhost:4567/test_json \
    -d '{ "name":"Rohit", "message":{ "body":"Hello World", "time": "8:45" } }'

We can see the response spit out in sinatra's console:

    {"name"=>"Rohit", "message"=>{"body"=>"Hello World", "time"=>"8:45"}}

You can see that JSON objects are transformed into [Ruby Hashes](http://www.ruby-doc.org/core-2.0/Hash.html) with the `parse` method. 

So now, lets write logic that will "Say hello", if someone asks us to:

{% highlight ruby %}
require 'sinatra'
require 'json'
#bot.rb

post '/callback' do
    json_data = JSON.parse(request.body.read)
    say_hello if json_data["text"] == 'Say hello'
end

def say_hi
    #TODO: say hello to the group
end

{% endhighlight %}

GroupMe's API Docs tell us that to send a message from our bot to the group,
we simply need to POST to https://api.groupme.com/v3/bots/post with our bot id,
and our message. We can use [Net::HTTP](http://ruby-doc.org/stdlib-2.0/libdoc/net/http/rdoc/Net/HTTP.html) to do this.
Lets fill in the implementation of `say_hi`:

{% highlight ruby %}

require 'sinatra'
require 'json'
require 'net/http'

BOT_ID = "YOUR_BOT_ID_HERE"
URI = URI.parse('https://api.groupme.com/v3/bots/post')

post '/callback' do
    json_data = JSON.parse(request.body.read)
    say_hi if json_data["text"] == 'Say hello'
end

def say_hi
    response = Net::HTTP.post_form(URI, { "bot_id" => BOT_ID, "text" => "Hello!" })
end

{% endhighlight %}

That's pretty much it for the code. We just need to make a config.ru file that Heroku needs:

{% highlight ruby %}
#config.ru
require './hello'
run Sinatra::Application
{% endhighlight %}

Now deploy by pushing to heroku. 

    git commit -a -m "initial commit"
    git push heroku master

And... that should be it. Try sending 'Say hello' in the group you've registered your bot to.

Resources
---------

[Heroku Ruby Help](https://devcenter.heroku.com/articles/ruby) - Read at least up to 'Visit your application'
[Deploying Rack-based Apps to Heroku](https://devcenter.heroku.com/articles/rack)
[GroupMe Basic 'bot' tutorial](https://dev.groupme.com/tutorials/bots)
[GroupMe API Reference](https://dev.groupme.com/docs/v3)
[Sinatra Docs](http://www.sinatrarb.com/intro.html)
[HTTP Methods](http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#Request_methods)

Let me know if anything is unclear, if there are errors, or if you just need help.
