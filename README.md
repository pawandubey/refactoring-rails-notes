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


The cons of this pattern are that it introduces some duplication of logic as well as it can introduce mutations outside of the controllers which can be hard to track down by the first glance.

### Alternative

An alternative approach is to use the `SimpleDelegator` mechanism provided by Ruby. [This post on Arkency](http://blog.arkency.com/2015/05/extract-a-service-object-using-simpledelegator/) brilliantly documents how `SimpleDelegator` can be used to construct _composable_ service objects. Composability allows us to chain multiple `SimpleDelegator` based service objects to accomplish complex actions without making the controller complicated.


## Form Objects

Many times we have to design forms for objects that take nested objects as attributes. Rails has the `accepts_nested_attributes_for` helper for this very purpose. However, testing models using `accepts_nested_attributes_for` is hard. Also the parameters returned from the form are in an inflexible format. This approach is also very _magical_ and forces iteration over nested parameters.

These situations can be better handled by Form Objects. Form objects are POROs that include the `ActiveModel::Model` module. This allows them to act like an `ActiveRecord` model and so they can be used in place of actual models in the view with the `form_for` helper.

This allows the construction of flat forms as well as makes the relation explicit. [This blog post from Thoughtbot](https://robots.thoughtbot.com/activemodel-form-objects) is a good primer for Form Objects.

There are some cons that come along with the pros though. Form objects introduce duplication if validations and does not support uniqueness validations with the rails validation helpers. This forces us to duplicate the whole uniqueness validation logic if require it in our application.


## Refactor Compound Conditionals into Methods

Consider the following block of code:

``` ruby
def eligible_for_return?
  expired_orders.exclude?(self) && self.value > MINIMUM_RETURN_VALUE
end
```

It checks if an order is eligible for return by checking two conditions: whether it is not expired and if the value is greater than the minimum required value for an order to be returnable. This logic is encoded in the code but the code doesn't say that at first glance. We have to look at the `expired_orders` method to see what it returns and also we can't be sure what the `self.value > MINIMUM_RETURN_VALUE` is supposed to mean. These conditions are just examples and can be replaced by any other non-trivial conditions. We also can't test this method properly without jumping through hoops.

This can be made better by refactoring the two conditions out into single-line methods (preferably private).

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

The conditions are clearer now and we don't need to jump through hoops to test the method as we can with good confidence stub the predicates out to return what we want them to.


### Refactor multiple Compound Conditionals into a Policy Class

If in the previous example, there were more than three conditions involved in the determining whether an order is returnable, we can do better than refactoring them out into methods. It is better to create a Policy Object that handles these conditions internally and provides a single point to add more conditions in the future or modify the existing ones.

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

The advantages offered by a Policy object is that it gets rid of the private methods (and hence makes them testable).
