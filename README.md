# Notes from Refactoring Rails

Personal notes from the course [Refactoring Rails](https://www.refactoringrails.io/buy) by Ben Orenstein.

# Notes

These notes are grouped together by topic and do not correspond to the exact chapters (or sections, if you may) of the course. I have also added my own addendums wherever found necessary both in form of notes as well as explanatory code, which are different from the original videos.

## Callbacks

Callbacks are hooks that happen at different stages of the life-cycle and are inflexible and too magical. They should be refactored into Service Objects as far as possible. Service Objects wrap the functionality of the action the callbacks were supposed to hook into and add operations before and after it optionally.

E.g:

This is a commonly encountered pattern in Rails repositories:

``` ruby
class UsersController < ApplicationController
  after_action :send_confirmation_email, only: [:create]

  def create
    @user = User.new(user_params)
    @user.save!
  end

  private

  def send_confirmation_email
    AccountCreationMailer.new(@user).deliver!
  end
end
```

Here `send_confirmation_email` sends a confirmation mail every time the `create` action is triggered. But what if the `create` action fails, or raises an error? Or worse yet, it fails without raising an error? The user won't be created but Rails will still try to send the email. To counter this, we need to introduce conditions inside the `create` action like so:

``` ruby

class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save!
      AccountCreationMailer.new(@user).deliver!
    end
  end
end
```

Now we've solved the problem of the invalid `user_params` but we've introduced branching inside the action. If we wanted to do anything even slightly more complex, the cyclomatic complexity of the `create` action will increase rapidly - which is an antipattern. It is hard to test and introduces complexity at the wrong layer - the controller.

This problem is better solved by refactoring the mailing functionality into a Service Object like so:

``` ruby
class UserConfirmationMailer
  attr_reader :user

  def initialize(user)
    @user = user
  end

  def send
    if user.persisted?
      UserConfirmationMailer.new(user).deliver!
    elsif
      # do something else here etc
    end
  end
end

class UsersController < ApplicationController
  def create
    @user = User.create!(user_params)
    UserConfirmationMailer.new(@user).send
  end
end
```

The `UserConfirmationMailer` service object wraps all the complexity around sending confirmation mails and provides an explicit, visible, non-surprising implementation of the logic which can be tested easily.


The cons of this pattern are that it introduces some duplication of logic as well as mutations outside of the controllers which can be hard to track down at a glance.

### Alternative

An alternative approach is to use the `SimpleDelegator` mechanism provided by Ruby. [This post on Arkency](http://blog.arkency.com/2015/05/extract-a-service-object-using-simpledelegator/) brilliantly documents how `SimpleDelegator` can be used to construct _composable_ service objects. Composability allows us to chain multiple `SimpleDelegator` based service objects to accomplish complex actions without making the controller complicated.


## Form Objects

Often we have to design forms for objects that take nested objects as attributes. Rails has the `accepts_nested_attributes_for` helper for this very purpose. However, testing models using `accepts_nested_attributes_for` is hard. Also the parameters returned from the form are in an inflexible format. This approach is also very _magical_ and forces iteration over nested parameters.

These situations can be better handled by Form Objects. Form objects are POROs that include the `ActiveModel::Model` module. This allows them to act like an `ActiveRecord` model and so they can be used in place of actual models in the view with the `form_for` helper.

This allows the construction of flat forms along with making the relation explicit. [This blog post from Thoughtbot](https://robots.thoughtbot.com/activemodel-form-objects) is a good primer for Form Objects.

There are some cons that come along with the pros though. Form objects introduce duplication of validations and does not support uniqueness validations with the rails validation helpers. This forces us to duplicate the whole uniqueness validation logic if we require it in our application.


## Refactor Compound Conditionals into Methods

Consider the following block of code:

``` ruby
def eligible_for_return?
  expired_orders.exclude?(self) && self.value > MINIMUM_RETURN_VALUE
end
```

It checks if an order is eligible for return by checking two conditions: whether it has expired and if the value is greater than the minimum required value for an order to be returnable. This logic is encoded in the code but the code doesn't say that at first glance. We have to look at the `expired_orders` method to see what it returns and we also can't be sure what the `self.value > MINIMUM_RETURN_VALUE` is supposed to mean. These conditions are just examples and can be replaced by any other non-trivial conditions. We also can't test this method properly without jumping through hoops.

This can be improved by refactoring the two conditions out into single-line methods (preferably private).

``` ruby
def eligible_for_return?
  not_expired? && over_minimum_return_value?
end

private

def not_expired?
  expired_orders.exclude?(self)
end

def over_minimum_return_value?
  self.value > MINIMUM_RETURN_VALUE
end
```

The conditions are clearer now and we don't need to jump through hoops to test the method as we can stub the predicates out with confidence to return what we want them to.


### Refactor multiple Compound Conditionals into a Policy Class

If in the previous example, there were more than three conditions involved in determining whether an order is returnable, we can do better than just refactoring them out into methods. It is better to create a Policy Object that handles these conditions internally and provides a single point to add more conditions in the future or modify the existing ones.

E.g.:

``` ruby
def eligible_for_return?
  ReturnEligibilityVerifier.new(self).eligible?
end

class ReturnEligibilityVerifier
  attr_reader :order

  def initialize(order)
    @order = order
  end

  def eligible?
    not_expired? && over_minimum_return_value? && customer_not_fraudulent?
  end

  def not_expired?
    Order.expired_orders.exclude?(order)
  end

  def over_minimum_return_value?
    order.value > Order::MINIMUM_RETURN_VALUE
  end

  def customer_not_fraudulent?
    order.user.not_fraudulent?
  end
end
```

The advantages offered by a Policy object is that it gets rid of the private methods (and hence makes them testable). This is a special case of the [Extract Class]() refactoring method.

## Have a bin/setup

Having an easy way to setup the dev environment (preferably with a single command) is a great advantage for teams. This script can be in the form of a rake task or a combination of many of them or even a class that sets up and creates required records to get the application in a runnable state.

Testing this script is not high priority as it is a development tool.

## Promote TODO comments to separate issues

Having TODO comments in code is not generally productive as the author of those comments imagines. They get lost and nobody really checks for them everyday. Adding these comments to the team's backlog as issues prioritizes their fix and avoids the situation where TODO comments linger on in the codebase for years without anyone noticing.

## Comments are OK with non-standard approaches

Certain times we need to do something non-standard or clever to achieve some much needed performance benefit or behavior that the standard approach doesn't give us. Explanatory comment about the justification for such approaches are fine and help people understand the intention.

## Order methods by layer of abstraction

Consider the following code block:

``` ruby
# toplevel
def top_level_method
  level_1_method_1
  level_1_method_2
end

# level 1
def level_1_method_1
  level_2_method_1 && level_2_method_2
end

def level_1_method_2
  level_2_method_1 && !level_2_method_2
end

# level 2
def level_2_method_1
  do_something
end

def level_2_method_2
  do_something_else
end
```

The definitions of methods in this code block are ordered in the order these methods are encountered in the call stack. This property is nice in two ways:

- Each method is above all of its constituent methods. This means if we want to understand the underlying logic, we only need to move in one direction in the file - downwards.
- Since each abstraction level is isolated from the others, it is easy to extract one of the abstraction levels into a separate class if it becomes too big. E.g. if level 2 in the example comes out to have more than 4 closely related methods which are used by methods in level 1, we can extract all of level 2 into a new class with public methods which can be called by level 1 methods.


## Encourage use of Object#tap

`Object#tap` is a method that yields the object it is called on (the receiver) to its block, and at the end returns the same object. This property is very useful when you have a series of method calls on the same object one after the other.

E.g.:

``` ruby
def do_something
  obj.do_first_thing
  obj.do_second_thing
  obj.do_third_thing
  obj
end
```

This pattern can be refactored into :

``` ruby
def do_something
  obj.tap do |o|
    o.do_first_thing
    o.do_second_thing
    o.do_third_thing
  end
end
```

This doesn't save in number of lines, but it does increase the readability as it groups the operations on the same object into a nested block and also returns the object, removing the need to explicitly return it at the end if the previous method doesn't return the same thing.

## Prefer early returns

Early returns to guard against undesired conditions is a good practice. It lets the reader of the code avoid reading the whole logic if the guarding condition is not met. It also avoids the unnecessary nesting that would have been created had the early return not been used.

E.g.:

``` ruby
def charge_purchase(order)
  return unless order.fulfilled?
  OrderChargeConfirmation.new(order).create!
end
```

The early return here signals that the charge can only be processed if the order has been fulfilled. Compare that with the non-early return version:

``` ruby
def charge_purchase(order)
  if order.fulfilled?
    OrderChargeConfirmation.new(order).create!
  else
    nil
  end
end
```

This version has the actual logic nested inside `if` statements. It also demotes the early return to the end of the block. This is both less readable and is prone to complexity as any added conditions will increase the cyclomatic complexity of this method. The early return is preferable.

## Prefer the `!` version of methods

If there are two versions of a method, and one of them raises an exception on failure and the other doesn't, prefer using the exception raising version. This is useful to surface the errors in your code early and avoid unwanted side-effects from happening. It also helps you to avoid writing defensive code riddled with conditionals.

It is important to note that exception-rescuing mechanism shouldn't be used in lieu of conditionals.

## Avoid the Control Couple

Consider the following code:

``` ruby
status = order.paid?
update_charge_status(status)

def update_charge_status(paid)
  if paid
    send_confirmation_notification
    mark_as_completed
  else
    initiate_payment
    update_payment_status
  end
end
```

Here, the calling code knows what the `update_charge_status` method is going to do. This is an implicit coupling which we are trying to switch on by using the `status` argument. This argument and the corresponding method is called a `Control Couple` because the argument essentially controls what happens inside the method. This abstraction is unnecessary and as the calling code already has enough information to predict what the method will do we can refactor that method into two different methods and extract the conditional up the stack like so:

``` ruby
if order.paid?
  confirm_charge
else
  initiate_charge
end

def confirm_charge
  send_confirmation_notification
  mark_as_completed
end

def initiate_charge
  initiate_payment
  update_payment_status
end
```

This enforces the single responsibility principle for the methods and avoids the coupling of the update methods with the order payment status.

## Avoid using instance variables in partials

Partials are a valuable feature because they add reusability to our views. Using instance variables in partials directly couples them to specific controllers, essentially erasing their reusability. It is preferable to keep partials reuasble across controllers and actions as much as possible, which means that we should avoid directly using instace variables in them, and instead rely on locals for passing in values using the `locals` option to `render`.

See https://guides.rubyonrails.org/layouts_and_rendering.html#local-variables

## Replace foo(a), bar(a), baz(a) with a.foo, a.bar, a.baz

If you see several methods that depend on the same parameter, it's a good candidate for applying the Extract Class refactoring, and specifically the [Combine Functions into Class](https://refactoring.com/catalog/combineFunctionsIntoClass.html) method. The repeated use of the same argument is a smell that there's a set of common behaviour screaming to be extracted out.

## Tell, don't ask

We should aim to colocate data with methods that operate on that data. This is one of the core tenets of object oriented design and manifests quite prominently in the form of classes and modules. However, it is also easy to overlook this rule by violating the _tell, don't ask_ principle.

E.g.

Instead of

``` ruby
if @user.has_address?
  @user.address.street_name
else
  "Unknown street"
end
```

We can do:

``` ruby
class Adress
  def street_name
    @street_name
  end
end

class NullAddress
  def street_name
    "No street name"
  end
end

class User
  def address
    @address || NullAddress.new
  end

  def street_name
    address.street_name
  end
end
```

Some caveats do apply, of course. One of them being the potential to violate the single responsibility principle by trying to colocate data and methods dogmatically. Another one is the inclination of applying this rule in a way that discourages all kinds of query methods (e.g. getters).

Also see: [Replace conditional with polymorphism](https://refactoring.com/catalog/replaceConditionalWithPolymorphism.html)

## Utilize the four-stage test pattern

Unit tests (almost) without exception do four things: setup the fixture for the test, do something to the fixture, verify that the result of the exercise matches expectations, and clear up any side-effects at the end. The four phases here are _setup, exercise, verify_ and __teardown_. By clearly identifying these four steps in each of our test cases, we can make tests both easier to understand and change.

E.g.

``` ruby
class UserTest < Minitest::TestCase
  def setup
    @user = User.new
  end

  def teardown
  end

  test "user without address is invalid" do
    # exercise
    @user.address = nil

    # verify
    refute_predicate @user, :valid?
  end
end
```

With Minitest, by using the `setup` and `teardown` methods, we can further clean-up each test case by only requiring the exercise and verification to be present. Judicious use of whitespace between these steps can also help increase legibility.

## Compare properties instead of whole objects

By comparing individual properties of objects instead of the complete objects themsleves, we can get clearer and easier to read error messages in tests. This is doubly true if we're comparing a collection with another.

Instead of:

``` ruby
test "it has same users"
  assert_equal [User.first, User.second], [first_user, second_user]
end
```

We can do

``` ruby
test "it has same users"
  assert_equal [User.first.id, User.second.id], [first_user.id, second_user.id]
end
```

And instantly get the benfit of increased inspectability in case our test fails.

## Avoid mystery guest

Continuing from our earlier point about the four-phase test pattern, if the relationship between the setup and verification logic is unclear to the test reader, it's possible this is caused by some part of the test being executed _outside_ the test itself. This invisible part is called a Mystery Guest. Sometimes, avoiding it is as simple as using proper variable names for our fixtures but often it's a matter of not making our tests _too_ DRY. Having shared setup (e.g. in the form of widely used fixtures) also causes coupling between different tests that is hard to resolve if the problem grows.

Some possible solutions for this problem are:
- Using fresh fixtures by inlining setups
- Using clear names for variables and helper methods
- Repeating yourself for the sake of clarity

Avoiding Mystery Guests help make our tests into actual _documentation_ for our code.

Also see: [Obscure Test](http://xunitpatterns.com/Obscure%20Test.html)

## Don't test private methods

Private methods should be used to support the public interface of a class and tests should only be written for this public interface. If you ever find yourself struggling to test some behaviour because it's provided by a private method, it's a signal that the behaviour should instead be extracted into a class that exposes it as a public method.

## Don't test things that you don't need to

It's important to only test the things that matter to keep your tests clean and noise-free. An example of an unnecessary test is testing behavior provided by ActiveRecord in your own models as that behaviour is already tested in the Rails codebase.

So don't do things like:

``` ruby
# bad
test "saves user correctly"
  @user = User.new

  @user.save

  assert_equal User.last.id, @user.id
end
```

## Use REST

Rails provides excellent support for RESTful design and it's beneficial to use as much of it as possible to get the most out of the convention-over-configuration approach. An example of this is using the `Noun#verb` pattern for controller actions that can be then tied to routes and views very easily.
