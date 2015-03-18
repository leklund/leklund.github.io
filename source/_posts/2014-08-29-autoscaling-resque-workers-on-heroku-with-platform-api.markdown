---
layout: post
title: "Autoscaling Resque workers on Heroku with the platform-api gem"
date: 2015-01-29 11:25:03 -0400
comments: true
categories: [resque, rails, heroku, papertrail]
---

Heroku is a great service and is extremely powerful. It's also not cheap so leaving workers idling can cost you a lot of money.
If you have background workers that do variable amounts of work it makes sense to scale up the number of workers as new jobs
are queued and then scale back down as the queue clears.

Here is how one might scale workers using [resque hooks](https://github.com/resque/resque/blob/master/docs/HOOKS.md) and the [Heroku platform-api](https://github.com/heroku/platform-api) gem.
This example scales up a worker for every three pending jobs. It will scale down once the queue is empty and
all the workers are finished. It will also scale down when the number of idling dynos is greater than 5.
Because it is not possible to control which dynos are killed off, this assumes that your resque workers are handling TERM
signals (and hopefully the jobs are idempotent). It is likely that the downscale could kill off working dynos so the jobs so be sure
to handle the Resque::TermException for jobs.


```
# config/initializers/heroku-platform-api.rb

$heroku = PlatformAPI.connect_oauth(ENV['HEROKU_OAUTH_TOKEN'])


# lib/scaled_job.rb

require 'platform-api'

module ScaledJob
  def after_enqueue_upscale(*args)
    worker_count = $heroku.formation.info('app-name','worker')["quantity"]
    job_count = Resque.info[:pending].to_i
    # one worker for every 3 jobs (minimum of 1)
    new_worker_count = ((job_count / 3) + 1).to_i
    if new_worker_count > worker_count
      $heroku.formation.update('app-name', 'worker', {"quantity" => new_worker_count})
    end
  end

  def after_perform_downscale
    worker_count = $heroku.formation.info('app-name','worker')["quantity"]
    working = Resque.info[:working].to_i
    if Resque.info[:pending].to_i == 0 && working <= 1
        $heroku.formation.update('app-name', 'worker', {"quantity" => 1})
    elsif Resque.info[:pending].to_i <= 1 && ((working + 5) < worker_count)
        new_worker_count = working + 5
        $heroku.formation.update('app-name', 'worker', {"quantity" => new_worker_count})
    end
  end
end


# model class
class Foo
  extend ScaledJob

  def self.perform(*args)
    # work it!
  end
end
```


