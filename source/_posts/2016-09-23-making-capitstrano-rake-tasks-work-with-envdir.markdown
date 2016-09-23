---
layout: post
title: "Making Capistrano rake tasks work with envdir"
description: 'How to setup Capistrano and SSHkit so that rake tasks load envdir first'
date: 2016-09-23 13:10:48 -0500
comments: true
categories: [ruby, rails]
keywords: ruby,rails,capistrano,envdir
---
Using envdir to manage environment variables for a Ruby app means that rake tasks run via
Capistrano can fail or have unexpected output because the ENV hash won't be populated with
what is located in the envdir.

__What is `envdir`?__

https://cr.yp.to/daemontools/envdir.html

> `envdir` runs another program with environment modified according to files in a specified directory.

__Why not use `dotenv`?__

`dotenv` isn't meant for production and `envdir` integrates nicely with `runit`. And if that's how your
SRE team wants to manage apps then that's what you use.

__So how can I fix it?__

Add to the rake command prefix in SSHKit.

Create a new Capistrano task

```
# lib/capistrano/tasks/envdir.rake

desc 'Prefix rake tasks with envidr'
namespace :envdir do
  task :prefix_rake do
      SSHKit.config.command_map.prefix[:rake].unshift 'envdir /path/to/env/directory'
    end
  end
end
```

This will create a new Capistrano task: `envdir:prefix_rake`. By using `unshift`
we ensure that if the prefix has already been set (for example by `capistrano/bundler`)
that we don't overwrite what is already there.

Wire it up.

Load the rake task
```
# Capfile

Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

Inject the new task
```
# config/deploy.rb

before 'deploy:starting', 'envdir:prefix_rake'
```

Capistrano will now run rake tasks prefixed. For example,
here is how the asset precompilation task will run (assumes `capistrano/bundler`
has been required as part of `capitrano/rails`)

```
envdir /path/to/env/directory bundle exec rake assets:precompile
```
