---
layout: post
title: "RSpec Tech Talk"
date: 2012-07-04 20:14
comments: true
categories: [Technology, Testing, Tech Talk]
author: Mukund Lakshman
---
Midway through '10 we found our product market fit and with the increased usage, [new focus](/blog/2012/05/30/a-transitional-year/),
and a real desire to building something massive we started a huge internal effort to increase code quality. Testing is a cornerstone
of everything that we write now. RSpec has changed the format in which tests are to be expressed and recently we got together to
share that information with all devs with the aim being that all new tests you write should be in the new format and convert the tests
written in the old format when you come across them.

<!-- more -->

_Everything is lazy now_


The most significant change is that test evaluation is lazy now. This is implemented with a new DSL. The proposed benefits of this
change are that the minimum set of infrastructure is spun up for each test case. We haven't noticed an increase or slowdown in the
runtime performance of tests written in this new format - this is likely that our tests are bound by db IO (we've recently migrated
all of our tests to use [FactoryGirl](https://github.com/thoughtbot/factory_girl/) and as a result we can use a non centralized
test db). However we prefer the expressiveness of this format and when dealing with tests it really helps to write more with fewer
lines.

``` ruby Old style basic test
require 'spec_helper'


describe BasicController do
  render_views

  before(:each) do
    @user = FactoryGirl.create(:user)
  end

  describe '#show' do
    context "error cases" do
      context "when no login parameter is provided" do
        get :show

        response.code.should eq("404")
        Yajl.load(response.body).should eq({:error => :missing_login})
      end

      context "when an invalid login parameter is provided" do
        get :show, :login => SecureRandom.hex(6)

        response.code.should eq("404")
        Yajl.load(response.body).should eq({:error => :invalid_login})
      end
    end

    context "when a login parameter is provided" do
      get :show, :login => @user.login

      response.code.should eq("200")
      json = Yajl.load(response.body)
      json.should eq(@user.to_json)
    end
  end
end
```
would now be written

``` ruby New style basic test
require 'spec_helper'

describe BasicController do
  render_views

  describe '#show' do
    subject do
      get :show, :login => login
      response
    end


    context "error cases" do
      let(:expected_body) { {:error => error_symbol } }

      context "when no login parameter is provided" do
        subject do
          get :show
          response
        end

        let(:error_symbol) { :missing_login }

        its(:code) { should eq("404") }
        it { Yajl.load(subject.body).should eq(expected_body) }
      end

      context "when an invalid login parameter is provided" do
        let(:login) { SecureRandom.hex(6) }
        let(:error_symbol) { :invalid_login }

        its(:code) { should eq("404") }
        it { Yajl.load(subject.body).should eq(expected_body) }
      end
    end

    context "when a login parameter is provided" do
      let(:user) { FactoryGirl.create(:user) }
      let(:login) { user.login }

      its(:code) { should eq("200") }
      it { Yajl.load(subject.body).should eq(user.to_json) }
    end
  end
end
```

These two files will provide the base for summarizing the changes between the old format and the new format. The main keywords we'll
focus on are:
 * subject
 * let
 * it / its()

Not mentioned here are _describe_ and _context_, these haven't changed in the DSL however you'll find that you use them more in the
new format.

subject
-------

_subject_ blocks are evaluated when rspec hits an it/its call. The return value is available in it/its blocks as the varaible _subject_.
This permits you to build up the properties of your subject as you evaluate the file, you can see this in the new format where we
describe the basic subject on line 6. _login_ does not exist until we have evaluated all the in-scope let blocks after encountering an it.

The [rspec docs](https://www.relishapp.com/rspec/rspec-core/v/2-10/docs/subject/explicit-subject) naturally cover the concept of subjects
in detail. Something we don't make enough use of is the ability to
[implicitly define a subject](https://www.relishapp.com/rspec/rspec-core/v/2-10/docs/subject/implicitly-defined-subject)

let
---

_let_ is the most important part of the new DSL. Fully grokking how _let_ and _let!_ work and differ from one another permits you to
really DRY up your tests. The first thing to understand is that _let_ attempts to provide a concept of lexical scoping:

``` ruby let stacking
  ...
  describe "something" do
    let(:the_thing) { :foo }
    context "when interesting" do
      let(:the_thing) { :bar }

      it { the_thing.should eq(:bar) } # run first
    end

    it { the_thing.should eq(:foo) }   # run second
  end
  ...
```
Clearly this is not like ruby and after a while of completely abusing variable scopes it doesn't feel "natural". Sometimes your _let_ has
a side effect that needs to be realized immediately _let!_ provides this ability. We've rarely found the need to use it, though it has
helped out in an occasion or two.

It is very easy to conflate _subject_ and _let_, as the example above does; it would be far more correct to have the_thing in a _subject_.
You should keep an eye on this, by using _subject_ your _it/its_ calls be far more concise.

_let_ has begun replacing our use of _before(:each)_, particularly when stubbing out values; something that we'll go into depth in another
post.

it and its()
------------

_it_ provides the basis of an assertion. The default behavior is to operate on the result of a subject block, in our examples above we're
returning _response_. When _should_ is encountered in an _it_ block without being called on an explicit receiver it is applied to the result
of _subject_:

``` ruby Implicitly called on subject
  ...
  describe "something" do
    subject { expectation }
    context "when exciting" do
      let(:expectation) { :exciting }
      it { should eq(:exciting) }
    end

    context "when boring" do
      let(:expectation) { :boring }
      it { should eq({:boring}) }
    end
  end
  ...
```
_its(:sym)_ is a special form of _it_ which permits you to assert that a given property is what you expect it to me. Given

``` ruby its(:sym)
  ...
  describe "something" do
    subject do
      get :index
      response
    end

    its(:code) { should eq("200") }
  end
  ...
```

The _its(:code)_ call calls #code on the subject, the block then uses this as the implicit subject.

This new DSL actually has one significant downside - every time you encounter an _it_ it evaluates the _subject_ and _let_ blocks that it should.
This means if you're testing lots of assertions you'll end up with lots of test environment spin up. The best solution we have for that right now
is that if you are testing lots of things you should employ good judgement and wrap your assertions in the block form of it.

``` ruby block form of it
   ...
   describe "something" do
    subject do
      get :index
      response
    end

    it do
      subject.code.should eq("200")
      subject.body.should eq("foo")
    end
   end
   ...
```


In summary the trade offs of the new format are:
 * less flexible, the old format lends to writing tests however you want
 * more syntax, and understanding that syntax - i.e. everything being lazy will burn you once or twice
 * more DRY
 * easier on the eyes, when you become acccustomed to the format it is much easier to navigate
 * less state, you're less likely to be burned by a rouge before block which establishes an expectation of an object


The expectation of our users that our site works all the time is a reasonable one, and the time investment in tests has already paid back
dividends by preventing breaking changes from hitting production. As we're writing more tests we're becoming far more opinionated about
the _how_ and _why_ of writing tests - something that I hope to delve into in more detail in future posts, along with details of our 
testing infrastructure and the tools that make it possible.
