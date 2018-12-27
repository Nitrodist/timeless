Title: BDD with Rspec and Steak
Subtitle: Tasty Testing Terminology
Author: steveklabnik

Behavior Driven Development is a test-first software development methodology. It's a fairly straightforward process if you're familiar with other agile development methodologies. (snip) Here's a general outline of how software is developed with BDD:

* All of the stakeholders in the project come together and decide what features the project needs. These "stories" are usually written up on index cards and then organized by priority.
* Each story is broken up into smaller "features" that represent one aspect of the story.
* An acceptance test is written for the feature.
* "The Outer Loop": Run the acceptance test, watch it fail.
* Fix any sort of non-logic error that the failing test gives you.
* Re-run the test. Repeat until you get a logical error. This transitions you to the "Inner Loop"
* Write a unit test that expresses the desired logic.
* Run the test. Watch it fail.
* Write the simplest code that passes the unit test.
* Run the test, see that it passes.
* Refactor code as appropriate.
* Pop back into the outer loop: Run the acceptance test again. If your acceptance test passes, congrats! Otherwise, drop back into the inner loop to fix the new logical error.
* Refactor one last time, run all of your tests, make sure they all pass.
* Congrats! Your feature is ready!

This might seem pretty complicated, but it's actually relatively straightforward. Let's go through a story that I'm writing for the Hackety Hack website. I'm the only stakeholder, so it's easy to make executive decisions. Here's the story:

As a user, I'd like the option to subscribe to forums and threads, and get notifications about updates.

Sounds pretty useful to me! I use email as a to-do list, and I want to make sure I'm kept up-to-date with any discussions. This story can be broken up into a few different features. Here they are:

    Scenario: Subscribing to a thread
      Given I've commented in a thread
      When someone else makes a comment
      Then I should receive an email
      And it should have a link to that thread

    Scenario: Unsubscribing from a thread
      Given I've subscribed to a thread
      And I'm on the page for that thread
      When I click the unsubscribe link
      And someone makes a comment
      Then I should not receive an email

    Scenario: Subscribing to a forum
      Given I'm on the index page for a forum
      When I click the subscription link
      And someone makes a new thread in that forum
      Then I should receive an email
      And it should have a link to that forum

    Scenario: Unsubscribing from a forum
      Given I've subscribed to a forum
      And I'm on the index page for that forum
      When I click the unsubscribe link
      And someone makes a new thread in that forum
      Then I should not receive an email

Cool! Four scenarios to make up this feature. You'll notice that I'm using certain language to talk about each scenario: Given, When, and Then. These words, while not mandatory are helpful: Most features involve some sort of initial system setup, an event that kicks something off, and then an effect that you want to see created. Thinking in these terms can be a useful way of getting started with a story or feature.

Okay! So let's work on the first scenario: subscribing to a thread. There are a few different libraries that can used with Ruby for acceptance testing. Cucumber can actually be used to read these specifications as I've written them, but lately, I've come around to a simpler tool: Steak. If you'd like to follow along, you can check out a copy of the site like this:

    $ git clone https://github.com/hacketyhack/hackety-hack.com.git
    $ rvm install 1.8.7
    $ rvm use 1.8.7
    $ rvm gemset create hackety-hack.com
    $ cd hackety-hack.com
    $ git checkout 311d19
    $ gem install bundler
    $ bundle install

This'll get your ruby and gemsets set up, check out the revision right before this feature was implemented, and install the gems you need. You might also want to install a copy of MongoDB, you'll need that, too.

I like to make a separate branch for every feature. This keeps things nice and organized, and if I want to work on multiple features at once, I can make sure they don't depend on each other in awkward ways. Let's do that now:

    $ git checkout -b subscription_management

Awesome. Time to write an acceptance test! Steak works with RSpec, so the tests are all in the spec/acceptance directory:

    $ mvim spec/acceptance/subscription_spec.rb

This'll open up a blank file. We need to do a little bit of setup, and comment out our English descriptions:

    require File.dirname(__FILE__) + '/acceptance_helper'

    feature "Subscriptions" do

    # Scenario: Subscribing to a thread
    #   Given I've commented in a thread
    #   When someone else makes a comment
    #   Then I should receive an email
    #   And it should have a link to that thread

    # Scenario: Unsubscribing from a thread
    #   Given I've subscribed to a thread
    #   And I'm on the page for that thread
    #   When I click the unsubscribe link
    #   And someone makes a comment
    #   Then I should not receive an email

    # Scenario: Subscribing to a forum
    #   Given I'm on the index page for a forum
    #   When I click the subscription link
    #   And someone makes a new thread in that forum
    #   Then I should receive an email
    #   And it should have a link to that forum

    # Scenario: Unsubscribing from a forum
    #   Given I've subscribed to a forum
    #   And I'm on the index page for that forum
    #   When I click the unsubscribe link
    #   And someone makes a new thread in that forum
    #   Then I should not receive an email

    end
{: lang=ruby }


Okay, now let's flesh out our first scenario with some Steak:

    scenario "Subscribing to a thread" do

      #Given I've commented in a thread
      thread = Factory(:discussion)
      me = Factory(:hacker)
      reply = Factory(:reply, :author => me.username, :author_email => me.email)
      thread.replies << reply

      Pony.should_receive(:deliver) do |mail|
        mail.to.should == [ me.email ]
        mail.body.to_s.should =~ /\/forums\/#{thread.forum}\/#{thread.slug}/
      end

      #When someone else makes a comment
      somebody = Factory(:hacker)
      log_in somebody
      visit "/forums/#{thread.forum}/#{thread.slug}"
      click_link "Reply"
      fill_in "Body", :with => "Here's my take on things: Dream big!"
      click_button "Create Reply"

      #   Then I should receive an email
      #   (see pony block above)
      #   And it should have a link to that thread
      #   (see pony block above)
    end
{: lang=ruby }

Okay. So. There's one thing that's a little weird about this test: we have to say that we expect Pony to receive an email before we create comment #2. This happens in tests that are similar to this one; we're really testing a side-effect.

Other notable parts of this test: We're using Factory Girl to set up the state of the world, and Capybara to interact with our application to make the new comment. We're also using a log_in helper that I've written for some of my other tests.

Okay, so we've got a test. Let's watch it fail:

    $ rake spec:acceptance
    *snip*
    Failures:
      1) Subscriptions Subscribing to a thread
         Failure/Error: somebody = Factory(:hacker)
         Validation failed: Username has already been taken


Whoops! Looks like our factory wasn't set up to generate a series of usernames. The factories are in spec/factories.rb, and here's the changes we need to make:

    Factory.sequence :username do |n|
      "user#{n}"
    end

    #this factory defines a hacker
    Factory.define :hacker do |u|
      u.username { Factory.next(:username)}
{: lang=ruby }

Okay, let's run those tests again:

    $ rake spec:acceptance
    *snip*
    Failures:
      1) Programs can be created
         Failure/Error: page.should have_content "By steve"
         expected #has_content?("By steve") to return true, got false
         # ./spec/acceptance/programs_spec.rb:18

      2) Subscriptions Subscribing to a thread
         Failure/Error: Pony.should_receive(:deliver) do |mail|
         (Pony).deliver(any args)
             expected: 1 time
             received: 0 times
         # ./spec/acceptance/subscription_spec.rb:13

Whoops! This is both good and bad. Test failure #2 is our logical error: the test ran, but Pony was expecting an email, and it didn't get one. The first error is something that happens sometimes: we have a more brittle test than we thought. Rather than write a test that actually looks up a username, I hardcoded a username. Whoops! After fixing that, we just get the expectation failure. Time to drop into the inner loop.

Our unit tests should be testing a few things: making a Reply should create a subscription to a thread, and it should also send a mail to everyone who has a subscription. Unit tests are kept in the spec directory. Let's check out our Reply tests:

    $ mvim spec/reply_spec.rb

... and there aren't any. So let's set that up:

    require File.expand_path(__FILE__ + "/../spec_helper")

    describe Reply do

      it "should do something"

    end
{: lang=ruby }

And run it:

    $ rake spec
    *snip*
    Pending:
      Reply should do something
        # Not Yet Implemented

I currently have 'rake spec' set up to run all of my specs, so we also get the failing acceptance test as well as the pending unit test. As your test suite grows, you'll generally break these out in a more fine-grained way, but the tests only take about two seconds to run, so I haven't found the need to yet.

Okay, let's actually write our unit tests:

    describe "subscriptions" do
      it "adds one to its Discussion upon creation" do
        discussion = Factory(:discussion)
        discussion.subscribed_users.count.should == 0
        discussion.replies << Factory(:reply)
        discussion.subscribed_users.count.should == 1
      end

      it "triggers an email to others upon creation"
    end
{: lang=ruby }

One written, one pending. And run them:

    $ rake spec
    *snip*
    2) Reply subscriptions adds one to its Discussion upon creation
       Failure/Error: discussion.subscribed_users.count.should == 0
       undefined method `subscribed_users' for #<Discussion:0x10355b1e0>

Let's create that method, and re run the spec:

    $ rake spec
    *snip*
    2) Reply subscriptions adds one to its Discussion upon creation
       Failure/Error: discussion.subscribed_users.count.should == 0
       undefined method `count' for nil:NilClass

Okay, we've set up our method correctly, but it's returning nil rather than some kind of Enumerable. Simplest thing? An empty array. Rerun spec time!

    $ rake spec
    *snip*
    2) Reply subscriptions adds one to its Discussion upon creation
       Failure/Error: discussion.subscribed_users.count.should == 1
       expected: 1,
            got: 0 (using ==)

Now that I'm looking at this test, though, it seems a little bit bad. Why am I asking for all of this stuff about a discussion? This is a reply test? And besides, if I write the after_create handler for the reply right now, it's going to have to mess around with the internals of its Discussion to do its job. That's a poor separation of concerns. Let's back up:

    $ git checkout models/discussion.rb

And rework our test:

    it "adds one to its Discussion upon creation" do
      email = "someone@example.com"
      @discussion = Factory(:discussion)
      @discussion.should_receive(:create_subscription!)
      @discussion.replies << Factory(:reply, :author_email => email)
      @discussion.save
    end
{: lang=ruby }

This is much more clear. We tell the discussion that someone would like to subscribe to it. And running our test tells us what is needed next:

    2) Reply subscriptions adds one to its Discussion upon creation
       Failure/Error: discussion.should_receive(:create_subscription!).with(email)
       (Double Discussion).create_subscription!("someone@example.com")
           expected: 1 time
           received: 0 times

So let's do that in models/reply.rb:

    after_save :send_subscription_notice

    private

    def send_subscription_notice
      _root_document.create_subscription! :author_email
    end
{: lang=ruby }

Two things about this: First of all, I'd really like this to be an after_create, not an after_save. It appears as though MongoMapper doesn't run after_create with embedded documents, though. I'm going to have to look into this a bit further before committing this code. after_save is good enough for now. Secondly, to reach back up out of the embedded document, we need to use the _root_document accessor. This starts with an \_, and so I'm not sure that it's the best way to accomplish this. \_s usually indicate private things that you shouldn't mess with. So I have two things that are kinda bad, but they're good enough for now. That's the whole point of being Timeless, eh?

Anyway, now we're getting three failures:

    3) Subscriptions Subscribing to a thread
       Failure/Error: click_button "Create Reply"
       undefined method `create_subscription!' for #<Discussion:0x1038a3348>
       # ./models/reply.rb:22:in `send_subscription_notice'

So it's time to add this method to Discussion. Let's pop open the (nonexistant) Discussion tests

    $ mvim spec/discussion_spec.rb

and add a test:

    describe Discussion do
      
      describe "#create_subscription!" do

        it "adds a new email to the subscription list" do
          discussion = Factory(:discussion)
          discussion.subscribed_users.count.should == 0
          discussion.create_subscription! "somebody@example.com"
          discussion.subscribed_users.count.should == 1
        end

      end

    end
{: lang=ruby }

And run them:

    $ rake spec
    *snip*
    4) Discussion#create_subscription! adds a new email to the subscription list
       Failure/Error: discussion.subscribed_users.count.should == 0
       undefined method `subscribed_users' for #<Discussion:0x1037ec3a0>

Okay, so we have no subscribed_users. Let's add an array of Strings to our model, to store the emails of subscribers:

  key :subscribed_users, Array
{: lang=ruby }

And run some specs:

    $ rake spec
    4) Discussion#create_subscription! adds a new email to the subscription list
       Failure/Error: discussion.create_subscription! "somebody@example.com"
       undefined method `create_subscription!' for #<Discussion:0x10384c570>

Okay! Time to get rid of most of these errors. Let's add an create_subscription! method:

    def create_subscription!
    end
{: lang=ruby }

And run some specs (tired of this yet?)

    $ rake spec
    4) Discussion#create_subscription! adds a new email to the subscription list
       Failure/Error: discussion.create_subscription! "somebody@example.com"
       wrong number of arguments (1 for 0)
       # ./spec/discussion_spec.rb:8:in `create_subscription!'

Whoops! Let's add that argument:

    def create_subscription! email
    end
{: lang=ruby }

And run our specs:

    $ rake spec
    Failures:
      1) Subscriptions Subscribing to a thread
         Failure/Error: Pony.should_receive(:deliver) do |mail|
         (Pony).deliver(any args)
             expected: 1 time
             received: 0 times
         # ./spec/acceptance/subscription_spec.rb:13

      2) Discussion#create_subscription! adds a new email to the subscription list
         Failure/Error: discussion.subscribed_users.count.should == 1
         expected: 1,
              got: 0 (using ==)
         # ./spec/discussion_spec.rb:9

Two big failures here: our acceptance test isn't seeing an email, and our subscription isn't adding anything to our array. Let's finish off our unit test before bouncing back into acceptance-land:

    def create_subscription! email
      subscribed_users << email
    end
{: lang=ruby }

And run our specs:

    $ rake spec
    Pending:
      Reply subscriptions triggers an email to others upon creation
        # Not Yet Implemented
        # ./spec/reply_spec.rb:12

    Failures:
      1) Subscriptions Subscribing to a thread
         Failure/Error: Pony.should_receive(:deliver) do |mail|
         (Pony).deliver(any args)
             expected: 1 time
             received: 0 times
         # ./spec/acceptance/subscription_spec.rb:13

    Finished in 1.44 seconds
    33 examples, 1 failure, 1 pending

And there it is: our second unit test that's still pending is testing our email, and the integration test isn't seeing it, either. But we've successfully navigated adding a subscription!

I'm going to continue updating this article in the future, going through the rest of the process of adding the rest of the necessary specs and features. As always, [comments are always welcomed](/comments).

Thanks to Dan Dorman and Lonny Eachus for a few suggestions and fixes.
