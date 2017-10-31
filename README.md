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
