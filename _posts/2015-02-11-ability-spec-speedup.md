---
layout: post
title:  "How we sped up our model spec to run 12 times faster"
date:   2015-02-11
summary:    When using factories to set up your test data, a lot of time is potentially wasted in persisting your models, which might not necessary.
tags:   rails specs factories
published: true
---

We are using [cancancan](https://github.com/CanCanCommunity/cancancan) as an authorization gem for one of our applications. To make sure that our authorization rules are correct, we unit-tested the `Ability` object. In the beginning, the test was quite fast, but the more rules we added, the longer it took to run the whole model test.

When we analyzed what was slowing down our test, we saw that quite some time is actually used persisting our models to the database with [factory_girl](https://github.com/thoughtbot/factory_girl) as part of the test setup. It took a bit more than 60 seconds to run the whole ability spec, which is far too much for a model test.

Let's look at an excerpt of our ability and its spec:

{% highlight ruby %}

# ability.rb

def acceptance_modes
  can [:read], AcceptanceMode
  if @user.admin?
    can [:create, :update], AcceptanceMode
    can :destroy, AcceptanceMode do |acceptance_mode|
      acceptance_mode.policies.empty?
    end
  end
end

{% endhighlight %}

{% highlight ruby %}

# ability_spec.rb

describe Ability do

  let!(:admin_user) { create(:admin_user) }
  subject!(:ability) { Ability.new(admin_user) }

  context 'acceptance mode' do

    let!(:acceptance_mode) { create(:acceptance_mode) }

    before(:each) do
      create(:policy, :acceptance_mode => acceptance_mode)
    end

    [:read, :create, :update].each do |action|
      it { should be_able_to(action, acceptance_mode) }
    end

    it { should_not be_able_to(:destroy, acceptance_mode) }

  end
end

{% endhighlight %}

{% highlight ruby %}

# ability_matcher.rb

module AbilityHelper
  extend RSpec::Matchers::DSL

  matcher :be_able_to do |action, object|
    match do |ability|
      ability.can?(action, object)
    end

    description do
      "be able to #{action} -- #{object.class.name}"
    end

    failure_message do |ability|
      "expected #{ability.class.name} to be able to #{action} -- #{object.class.name}"
    end

    failure_message_when_negated do |ability|
      "expected #{ability.class.name} NOT to be able to #{action} -- #{object.class.name}"
    end
  end
end

RSpec.configure do |config|
  config.include AbilityHelper
end

{% endhighlight %}

We first set up a user -- in this case it's an admin user -- and then initialize our ability object with this user. We further have a model called `AcceptanceMode`, which offers the usual CRUD operations. An acceptance mode has many policies. If any policy is attached to an acceptance mode, we don't want to allow it to be deleted.

Note that a lot of models are `created`, meaning these are persisted to the database. In this excerpt, we have 4 test cases. Each of these test cases needs to create the admin user, acceptance mode and also create a policy. This is a lot of persisted models, even more so if you realize that this is not all the acceptance mode specs and acceptance mode specs are only a small fraction of the whole ability spec. Other models are even more complex and require more tests for other dependencies.

But is this really necessary? Do we really need to persist the models or could we work with in-memory versions of these?

Let's take a look at this modified spec:

{% highlight ruby %}

describe Ability do

  let(:stub_policy) { Policy.new }
  let!(:admin_user) { build(:admin_user) }
  subject!(:ability) { Ability.new(admin_user) }

  context 'acceptance mode' do

    let(:acceptance_mode) { build(:acceptance_mode, :policies => [stub_policy]) }

    [:read, :create, :update].each do |action|
      it { should be_able_to(action, acceptance_mode) }
    end

    it { should_not be_able_to(:destroy, acceptance_mode) }

  end
end

{% endhighlight %}

Note that all the `create` calls are replaced with `build`. We actually don't need the models to be persisted to the database. The ability mainly checks if the user has admin rights (with `admin?`), which can be tested with an in-memory version of a user. Further, the acceptance mode can be built with an array that contains an in-memory stub policy. If you look closely at the `Ability` implementation, you will see that that's not even necessary. Any object could reside in the array and the spec would still pass. But we decided to use an in-memory policy nonetheless.

With this approach, no model is persisted to the database. All models are in-memory but still collaborate the same way as they would have when loaded from the database first. However, no time is wasted on the database. The whole ability spec run time was reduced from 60 seconds to 5 seconds, by simply avoiding to persist models to the database in the test setup.

As an aside: there's a lot of [discussions](http://brandonhilkert.com/blog/7-reasons-why-im-sticking-with-minitest-and-fixtures-in-rails/) around the topic of factories and fixtures. Fixtures load a fixed set of data into the database at the start of the test suite, which avoids these kinds of problems entirely.

That's it. We hope you can re-visit some of your slow unit tests and try to use in-memory models, or avoid persisting your models for the next unit test you write!
