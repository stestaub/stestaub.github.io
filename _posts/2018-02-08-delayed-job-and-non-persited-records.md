---
layout: post
title:  "Delayed Job AR and non persisted records"
date:   2018-02-08 12:00:00 +0100
categories: rails delayedjob
---

[Delayed Job](https://github.com/collectiveidea/delayed_job) 
> Delayed::Job (or DJ) encapsulates the common pattern of asynchronously executing longer tasks in the background.

I like this gem as it is pretty easy to integrate, can work on the same database as your other parts of the application (which
is good for getting started fast, although not recommended for production environments)

You can also use it to perform mails in the background:

```ruby
LicenceMailer.delay.licence_order_notification licence, tenant
```

This schedules a new Job to send mails in the background. I use this often for mailers, and as long as the environment is configured
correctly (mail username/pw etc.) this works just fine.

# No Mails sent
Just recently, we noticed, that mails didn't get sent. As we use sendgrid in the stage environment, we could make sure, that
the mails of one Mailer never were sent, yet other mailers were sent.

First thought, there might be an error in the mailer, but checking the background jobs didn't show anything. There were no failed jobs...

Calling the mailer manually did send the mail, when using the background job, it was not delivered.

So I parsed the handler stored in the background job and ran perform on it:
```
handler_yml = "--- !ruby...."
handler = YAML::load(handler_yml)
handler.perform
```

And it worked... So why did it not work in the background worker?

# What happened
Looking into the log files I realized an error, that an api key for Rollbar (our error reporting tool) was missing. After fixing that problem
and finally getting some errors reported from the workers, I could see that there was a deserialization error for the background job.

The error message was something like:

`ActiveRecord::RecordNotFound, class: MyModel, primary key:  (Couldn't find Example without an ID)`

I was aware that I was passing in a record without an id, but wait, when I was deserializing the handler manually, it seemed to work fine. So obviously, delayed_job did something different when deserializing.
I decided to reproduce it in a separate environment.

I created a sample rails app with delayed job, one model called `Example` one custom worker called `ExampleJob` and using Rspec to reproduce this. 
The code for the job looks like this:
 
`app/jobs/example_job.rb`
```
ExampleJob = Struct.new(:example) do

  def perform
    puts example.to_s
  end

end
```

I decided for three scenarios:
* Scenario 1: Run job with a PORO
* Scenario 2: Run job with a persited ActiveRecord model
* Scenario 3: Run job wit non persisted ActiveRecord model

## Scenario 1
First we pass a PORO:

```ruby
it 'should perform the job without error' do
  Delayed::Job.enqueue ExampleJob.new({ name: 'test' })
  expect{Delayed::Job.last.invoke_job}.not_to raise_error
end
```
As expected, this test will pass

## Scenario 2
Now we pass a persisted ActiveRecord model:

```ruby
it 'should perform the job without error' do
    record = ::Example.create name: 'test'
    Delayed::Job.enqueue ExampleJob.new(record)
    expect{Delayed::Job.last.invoke_job}.not_to raise_error
end
```

And of course, also this one works, as it is a very common case to pass Models to a job.
 
## Scenario 3
But when passing a record that is not persisted:

```ruby
it 'should perform the job without error' do
    record = ::Example.new name: 'test'
    Delayed::Job.enqueue ExampleJob.new(record)
    expect{Delayed::Job.last.invoke_job}.not_to raise_error
end
```

The test will no pass.

Turns out that the last one results in the described error. The reason is that dj hooks into the deserialization
and tries to load the record from the database:

`delayed/psych_ext.rb`
```ruby
    #...
    when %r{^!ruby/object}
      result = super
      if jruby_is_seriously_borked && result.is_a?(ActiveRecord::Base)
        klass = result.class
        id = result[klass.primary_key]
        begin
          klass.unscoped.find(id)
        rescue ActiveRecord::RecordNotFound => error # rubocop:disable BlockNesting
          raise Delayed::DeserializationError, "ActiveRecord::RecordNotFound, class: #{klass}, primary key: #{id} (#{error.message})"
        end
      else
        result
      end
     #...
```

Additionally, DJ will remove failed jobs that were failed due to Deserialization errors, meaning the following test will fail:

```ruby
context 'on Deserialization error' do
    before do
      Delayed::Job.enqueue ExampleJob.new nil
      expect_any_instance_of(Delayed::Job).to receive(:payload_object).and_raise(Delayed::DeserializationError.new)
    end
    
    it 'should not remove the job from queue' do
      expect {
        Delayed::Worker.new.work_off
      }.not_to change(Delayed::Job, :count)
    end
end
```

# Conclusion an learning
So first and probably most important learning:
 
 **Allways be sure that error reporting is working properly. That would have saved me a lot of time...**

Other learnings are

1. Delayed job does not work with Records without IDS
2. and it does not keep that failed job
3. avoid passing around non persisted records (And definitely do not pass them to DJ)

It would be nice if we could pass non persisted records to a job in order to do something with it. I'm not sure
what is the reason that is not supported.

But at least it would be nice if they stay in the job queue as failed (of course without reschedule them)