---
layout: post
title:  "Comparing Hashes with RSpec"
date:   2017-02-07 21:57:37 +0100
categories: rails coding
---

When you are working with rails and write tests with rspec, a lot of times you'll need to compare hashes.

Lets say have a service that creates a new instance of our model:

{% highlight ruby %}
class FooService

  def create_model(key)
    Foo.create(key: key, name: key.to_s.uppercase)
  end

end
{% endhighlight %}


Now we would like to test if key and name are set correctly. Rspec actually does pretty well when comparing hashes:

{% highlight ruby %}
subject { FooService.new.create_model :my_key }
let(:expected_attributes) { { key: :my_key, name: "MY_KEY" } }

it 'should have the correct attribtues' do
  expect(subject).to include( expected_attributes )
end
{% endhighlight %}

But wait, the test fails! Why? Because attributes have string keys. To get the test passing, just stringify the
keys of our model:


{% highlight ruby %}
subject { FooService.new.create_model :my_key }
let(:expected_attributes) { { key: :my_key, name: "MY_KEY" } }

it 'should have the correct attribtues' do
  expect(subject).to include( expected_attributes.stringify_keys )
end
{% endhighlight %}

Yeay, its all green now. You can find more information about the include matcher [here](https://www.relishapp.com/rspec/rspec-expectations/docs/built-in-matchers)