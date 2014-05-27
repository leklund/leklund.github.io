---
layout: post
title: "Safely ignoring Heroku errors on rake db:migrate"
date: 2014-05-27 17:37:30 -0400
comments: true
categories: [heroku, postgresql, rails]
---
(Repurposing my answer to this [stackoverflow question](http://stackoverflow.com/questions/17300341/migrate-not-working-on-heroku/17306077) as a blog post)

If you are using Rails 4.x on Heroku and you have your schema format set to sql (which you probably should), you will see this error when running ```heroku run rake db:migrate```

```
Aborted!
Error dumping database
.../activerecord-4.0.4/lib/active_record/tasks/postgresql_database_tasks.rb:55:in `structure_dump'
.../activerecord-4.0.4/lib/active_record/tasks/database_tasks.rb:143:in `structure_dump'
```

This is an interesting bug that, as it turns out, can be ignored.

ActiveRecord will only run the task ```db:structure:dump``` if you are using 'sql' as
your ```active_record.schema_format```.  And that rake task (```db:structure:dump```) will
fail on Heroku because the executable ```pg_dump``` is (unsurprisingly) not in the
binary path.  Interestingly enough, Heroku states that ```db:schema:dump``` is not
supported but if you set the schema format to ruby, it works fine. Of course,
you would end up with a ```schema.rb``` file on one-off dyno that will not exist
shortly after the rake task finished running.

In Rails 3, the dump task would only raise an error is the exit code of the
command was 1. On unix based systems if a command is not found, the exit code
is 127.  So even if the ```pg_dump``` command fails on Rails 3 (which it does), it
won't raise an error and it won't halt the rake task. So anyone using a sql
schema format with Rails 3 wouldn't have this issue because it would fail
silently. The code was refactored in Rails 4 to properly raise an error if the dump failed.
This means that ```db:migrate``` will raise an error on Heroku.

**However, even though it errors with rake aborted the DDL from the migration is actually executed and committed.**


Here are some possible solutions (if the error really bothers you):

* Ignore the error as the migrations are actually running.
* Since you don't care about the structure dump on production, set the schema_format to ruby. In config/environments/production.rb:

``` ruby
config.active_record.schema_format = :ruby
```

* If for some reason you don't want to change the config file: add a rake task with the following to suppress the error:

``` ruby
if Rails.env == 'production'
    Rake::Task["db:structure:dump"].clear
end
```

