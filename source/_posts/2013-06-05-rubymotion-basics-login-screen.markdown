---
layout: post
title: "RubyMotion Basics: Login Screen"
date: 2013-06-05 09:13
comments: true
categories: [iOS, RubyMotion]
---

RubyMotion is a neat tool that allows you to build iOS apps using Ruby code that eventually gets compiled into Objective-C. Unfortunately, there aren't a ton of blog posts demonstrating how to implement some of the more basic features of an iOS app using the Objective-C APIs, so hopefully this post will be of some use to aspiring RubyMotion developers. I'm assuming basic or better Ruby knowledge, and I definitely recommend checking out [this tutorial](http://rubymotion-tutorial.com/) for more of the basics.

Let's get started building our login screen by making a new RM app:

```
$ motion create login
```

And don't forget to cd into the directory.

First we'll get our app_delegate.rb into basic working order. But before that, a quick note: one issue I have with most of the RM samples I've found is that they cram everything into app_delegate.rb. I haven't figured out if this is a reflection of how iOS apps are built in Objective-C, or just for the convenience of demoing code, but as Rubyists let's try and work toward better adherence to MVC.

We're going to initialize a window to hold our views, then initialize our login controller and a navigation controller to manage it.

```ruby
def application(application,didFinishLaunchingWithOptions:launchOptions)
  @window = UIWindow.alloc.initWithFrame(UIScreen.mainScreen.bounds)

  login_controller = LoginController.alloc.init

  @navigationController = UINavigationController.alloc.initWithRootViewController(login_controller)

  @window.rootViewController = @navigationController
  @window.makeKeyAndVisible

  true
end
```

If you rake now, the compiler will complain since we haven't actually created our login controller, so let's go ahead and make one in app/controllers:

```ruby
class LoginController < UIViewController
end
```

This will get us... a blank screen. Exciting! Time to start on our login. We'll start by overriding the viewDidLoad method, a UIViewController callback that gets called after a controller's view is loaded into memory. Inside it, we'll add a subview to the root view that'll hold our text fields and login button.

```ruby
def viewDidLoad
  super

  # let's give it a title while we're at it
  self.title = "Login"

  # initialize the view with the whole screen as the frame
  @login_view = LoginView.alloc.initWithFrame(UIScreen.mainScreen.bounds)   # add our new view as a subview of the root view   self.view.addSubview(@login_view)

  true
end
```

Again our compiler will complain here because we haven't defined our LoginView, let's do that now in app/views:

```ruby
class LoginView < UIView
end
```


Before we move on, I want to introduce Teacup, a handy gem that allows you to extract some of the verbose aesthetics related methods into stylesheets. The easiest way to do this is with bundler. In your rakefile:

```ruby
require 'motion/project/template/ios'

# Add these lines:
require 'bundler'

Bundler.require
```

Then create a Gemfile in the root of your app and add:

```ruby
source 'https://rubygems.org'

gem 'rake'
gem 'teacup'
```


And run bundle on the command line.

```
$ bundle
```

Let's take teacup out for a spin. Similar to the controller's viewDidLoad method, we override the view's initWithFrame method to work our own subview and styling elements in. Let's start by setting up a white background. In the view:

```ruby
def initWithFrame(rect)
  super.tap do
    # tell teacup which stylesheet to use
    self.stylesheet = :login_stylesheet
    # tell teacup which style to apply to the view itself
    self.stylename = :root
  end

  self
end
```

Now let's setup our stylesheet in app/styles:

```ruby
Teacup::Stylesheet.new :login_stylesheet do
  style :root,
    backgroundColor: UIColor.whiteColor
end
```

Ok! We finally have our blank white screen! Let's get started by adding a container for our login interface. Teacup provides a simple DSL to implement and scaffold views. We're going to initialize a new UIView object to act as our container, then set its dimensions in our stylesheet. The subview command (which Teacup provides) accepts an object and a symbol that points to the appropriate 'styles' to add to that object, as defined by the stylesheet.

``` ruby
# in the view
def initWithFrame(rect)
  super.tap do
    # ...code ommitted
    self.stylename = :root

    @container = subview(UIView, :login_container)
  end

  self
end

# in the stylesheet
Teacup::Stylesheet.new :login_stylesheet do
  style :root,
    backgroundColor: UIColor.whiteColor

  # styles for our new login container:
  style :login_container,
    frame: [[70, app_size.height / 2 - 45],
      [180, 108]]
end
```

At this point, launching our app with rake will show no visible changes, however if you hold down the command key and mouse over the middle of the screen, a red box, our container, should be highlighted. It's also worth taking a second to decode what we're doing to set the position and dimensions of our frame in the stylesheet. A view's location is defined relative to its superview- in this case the entire window since that's what we passed into the initWithFrame method. The first array represent the x/y coordinates of the top left point of our container, and the second array represents the width and height. This relative relationship will become more clear when we create our text fields, which will be subviews of our container. Let's do that now using Teacup's handy block syntax:

```ruby
# in the view
def initWithFrame(rect)
  super.tap do
  # ...code ommitted

    @container = subview(UIView, :login_container) do
      @email_field = subview(UITextField, :email_input)
      @password_field = subview(UITextField, :password_input)
    end
  end

  self
end

# in the stylesheet
Teacup::Stylesheet.new :login_stylesheet do
  # ...code ommitted
  # we'll first define a parent input_field style from which our email and password fields
  # will inherit common attributes
  style :input_field,
    textAlignment: UITextAlignmentCenter,
    autocapitalizationType: UITextAutocapitalizationTypeNone,
    borderStyle: UITextBorderStyleRoundedRect

  # now let's define our email and password specific attributes:
  style :email_input, extends: :input_field,
    placeholder: 'email',
    frame: [[10, 10], [160, 26]],
    returnKeyType: UIReturnKeyNext

  style :password_input, extends: :input_field,
    placeholder: 'password',
    secureTextEntry: true,
    frame: [[10, 42], [160, 26]],
    returnKeyType: UIReturnKeyDone
```

Finally, something of substance!

{% img /images/2013-06-05-rubymotion-basics-login-screen/text_fields.png %}

However, you might notice that when we actually click into the text field, the keyboard pops up and blocks the field itself...not a great user experience. Let's get to work fixing that using animations, which will shift our view appropriately up and down so our user can see what they're typing.

We'll tap into more callbacks to achieve this effect. The textFieldDidBeginEditing method is, as the name implies, triggered whenever a text field is selected for editing. We'll use this, and a UIView animation, to replace the container's frame with a new one positioned so the user can see the text fields.

```ruby
# in the view
def textFieldDidBeginEditing(textField)
  # if we've already shifted the view up, don't do it again
  return if @offset

  # grab our current frame and modify it so it's visible
  container_frame = @container.frame
  container_frame.origin.y -= 70

  # animate the replacement of the current frame with the new one
  UIView.animateWithDuration(0.3,
    animations: lambda {
      @container.frame = container_frame
    },
    completion: lambda { |finished|
      @offset = true
    }
  )
end
```

To actually get this working, we need to set our text fields' delegates to themselves. More on what this is doing (here)[http://developer.apple.com/library/ios/#documentation/General/Conceptual/DevPedia-CocoaCore/Delegation.html].

```ruby
def initWithFrame(rect)
  # ...code ommitted
    @container = subview(UIView, :login_container) do
      @email_field = subview(UITextField, :email_input)
      @email_field.delegate = self

      @password_field = subview(UITextField, :password_input)
      @password_field.delegate = self
    end
  end

  self
end
```

Now if you rake and click into one of the text fields, the fields should shift up and remain visible.

Let's finish this off by implementing the reverse- if a user has finished entering info, shift the container back down. We'll also make it so that clicking the return button in the email field shifts the cursor to the password field. We'll use textFieldShouldReturn, which determines how the processing of the return button should be handled.

```ruby
def textFieldShouldReturn(textField)
  # if we're moving from email field to password, activate the password field for editing
  if textField == @email_field
    @password_field.becomeFirstResponder
  else
  # otherwise, shift the container back down
    container_frame = @container.frame
    container_frame.origin.y += 70

    UIView.animateWithDuration(0.3,
      animations: lambda {
        @container.frame = container_frame
      },
      completion: lambda { |finished|
        @offset = false
      }
    )

    # this will hide the keyboard
    textField.resignFirstResponder
  end

  false
end

```

And that's it! Run rake and take the new login screen for a test drive. Here's [all the code in one place](https://github.com/mhriess/rubymotion_login).







