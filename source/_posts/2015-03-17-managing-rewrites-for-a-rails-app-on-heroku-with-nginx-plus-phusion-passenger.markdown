---
layout: post
title: "Managing rewrites for a Rails app on Heroku with Nginx + Phusion Passenger"
date: 2015-03-17 23:26:06 -0400
comments: true
categories: [rails, nginx, passenger, rewrites, heroku]
---
The custom CMS that we built for kaboom.org has to manage a lot of rewrite rules because the
information architecture keeps getting rejiggered. We were managing them with Rack::Rewrite
but I always hated having a huge stack of rewrites in the application.rb configuration. Since the app
is hosted Heroku there weren't really any other options until I realized that Phusion Passenger
will now run in standalone mode on Heroku.

It's quite easy to get Passenger running on Heroku: https://github.com/phusion/passenger-ruby-heroku-demo

replace the app server in your Gemfile (unicorn, thin, puma, etc) with

    gem 'passenger'

Update your Procfile

    web: bundle exec passenger start -p $PORT --max-pool-size 3

This works great but doesn't come with an obvious was to edit the nginx.conf to add rewrite rules or an include directive.
According to the [advanced configuration](https://www.phusionpassenger.com/documentation/Users%20guide%20Standalone.html#advanced_configuration) section
of the Passenger standalone manual, we see that there is an option to pass a configuration template.

After adding Passenger to your Gemfile and running a ```bundle install``` tun this from you application root directory
to create a configuration template.

    cp $(passenger-config about resourcesdir)/templates/standalone/config.erb config/nginx.conf.erb

You could just add rewrites to this file but because the default configuration template may change from time to time
I would recommend adding any custom configuration to another file. Just after the ```server {``` line add

    include '<%= app[:root] %>/config/nginx_custom_config.conf';

If you are using this to perform rewrites and you are running in Heroku, this needs to be included

    port_in_redirect off;

Because Nginx is listening on the random port assigned by the Heroku routing layer Nginx will, by default, include this port
number in the rewrite rules that do HTTP redirects.

Other use cases include enforcing SSL on your app, adding HTTP basic auth to an app, and easier generation of CORS headers.
So far I'm loving Phusion Passenger standalone on Heroku. In a future post I plan to examine the performance and memory usage
versus the existing Unicorn setup.
