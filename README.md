NotifyOn
==========

**WARNING: This branch is the bleeding edge. This documentation is in progress, and the code within this branch is not ready for production.**

NotifyOn generates automatic notifications as a result of state changes on a particular model in a Rails application. It supports email messages, along with third-party real-time delivery, while also storing each notification in your database so you can use, adjust, and display as you'd wish.

Getting Started
----------

This gem is built for Ruby on Rails projects. Add it to your Gemfile:

```ruby
gem 'notify_on'
```

Then run the bundle command to install it.

Most of the magic happens behind the scenes. We just need to add two files to your app (`config/initializers/notify_on.rb` and the `NotifyOn::Notification` migration).

```text
$ bundle exec rails g notify_on:install
```

Migrate your database and then you're all set.

Usage
----------

NotifyOn is driven entirely through a `notify_on` call within a model.

### notify_on

```ruby
notify_on(action, options = {})
```

#### Parameters

| Name | Type | Description |
| --- | --- | --- |
| `action` | `symbol` | **Required**. Can be: <ul><li>`:create` to create the notification _after_ an object is created and saved to the database.</li></ul> |
| `options` | `hash` | **Some options are required.** See below for details. |

#### Options

| Name | Type | Description |
| --- | --- | --- |
| `to` | `symbol` | **Required.** Who to send the notification _to_. It should be a method (or association) on the model that triggered the notification. It may be a single object or a collection (array) of objects. But, each individual object **must have an `email` attribute/method** if you are using email notifications. <small>(See note on `to_s` below.)</small> |
| `from` | `symbol` | Who the notification is sent _from_. It **must be a method on the model that triggered the notification**, and it **must have an `email` attribute/method** if you are are using email notifications. <small>(See note on `to_s` below.)</small> If you omit this and have email notifications enabled, the mailer will use your configured default email address. |
| `message` | `string` (interpolated) | **Required.** The message that describes the notification itself. <small>(See _String Interpolation_ for more information.)</small> |
| `link` | `string` (interpolated) | **Required**. Uses Rails' URL helper to generate a reference link for the notification. |
| `skip_if` | `symbol` | The name of an instance method that returns a boolean. If `true`, a notification will not be created. If `false` or `nil`, it will. |
| `to_class_name` | `string` | **Required** _if `to` is an array_. If you are generating notifications for a collection of objects, you must set this to the class name of the individual objects within the array (which, should all be of the same class). |
| `email` | `hash` | (Default: `nil`) If present, it will attempt to send an email notification. Options: <ul><li>`default_from` (`boolean`) Use the default email address as sender instead of the notification's sender.</li><li>`template` (`string` or `symbol`) The name of the email template to render.</li><li>`send_unless` (`symbol`) a method on the trigger that returns a boolean. If `true` the email **will not** be sent. It falls back to methods on the notification if it doesn't find a match.</li> |
| `pusher` | `hash` | (Default: `nil`) Pusher provides access to real-time notifications. This hash should contain the following: <ul><li>`channel` (interpolated `string` with added options) as the name of the channel</li><li>`event` (interpolated `string` with added options or `symbol`) as the name of the Pusher event</li></ul> <small>(See _Pusher_ section for more information.)</small> |

_Note: Many of the defaults within `notify_on` make use of the `to_s` method. It's a good idea to override `to_s` on the models representing your `to` and `from` objects on a notification. Most of the time, this is a `User` object, so your `User` model may have something like this:_

```ruby
def to_s
  name
end
```

#### Example

```ruby
notify_on(
  :create,
  :to => :user,
  :from => :author,
  :message => '{author_email} sent you a message.',
  :link => 'message_path(:self)',
  :skip_if => :stale?,
  :email => { :default_from => true, :template => 'new_message' },
  :pusher => {
    :channel => 'presence-{:env}-message-{id}',
    :event => :new_message
  }
)
```

### receives\_notifications

We've had issues attempting to dynamically generate an association on the model represented by the `to` argument in a `notify_on` call. This was specifically troublesome in cases using single table inheritance.

As a result, NotifyOn requires that you explicitly set associations for your notification recipients. All you have to do is put the following **in the model representing objects that _receive_ notifications**:

```ruby
receives_notifications
```

This will provide an association, `notifications`, to that model. See _Automatic Associations_ below for details on how NotifyOn associations work.

Custom Features
----------

### String Interpolation

NotifyOn has a custom string interpolator for injecting an object's attributes. We can't use Ruby's built-in interpolatation because we don't have access to instance methods at the time we define `notify_on`.

NotifyOn's string interpolation syntax is similar to Ruby's. It just omits the `#` character. Here's an example:

```ruby
# ...
:message => '{author_email} sent you a message.'

def author_email
  author.email
end
```

You also have the option to daisy-chain methods together. So, this will work, too, assuming `author` has an `email` attribute or method.


```ruby
# ...
:message => '{author.email} sent you a message.'
```

#### Added Options

In some cases, we add a few options to string interpolation. These options must use a colon (`:`) before the method call. They are:

- `:env`: A lower-case string representing the current Rails environment.
- `:recipient_id`: The `id` of the recipient object attached to a notification.

For example, in a pusher channel, you might specify something like this:

```ruby
# ...
:pusher => {
  :channel => 'presence-{:env}-notification-{:recipient_id}',
  :event => :new_notification
}
```

The `channel` option here may resolve to something like `presence-development-notification-24`.

### Dynamic Link Generator

NotifyOn's dynamic link generator helps make your configuration a little simpler. We can't call the the URL helper directly from the `notify_on` call, so we are forced to get a little clever.

You can specify the relevant URL using a `_path` or `_url` helper, just as you would in your controllers/views. There are really only three differences.

First, the argument must be a string. So, instead of this:

```ruby
:link => messages_path
```

You need to do this:

```ruby
:link => 'messages_path'
```

Second, arguments in the path must be called using symbols. This way you can use a string as an argument. So, not this

```ruby
:link => 'author_path(author)'
```

But this:

```ruby
:link => 'author_path(:author)'
```

And last, a reference to the trigger itself is written as `:self`. For example:

```ruby
:link => 'author_message_path(:author, :self)'
```

### Override Default Email Message

NotifyOn has a very basic email template that uses the description to notify the `to` in your `notify_on` configuration.

You may override this default behavior with a custom email template. You specify this using the `template` option. The view itself goes in `app/views/notifications`.

Within this view, you have access to the notification as `@notification`. From there you can get to the following properties:

- `recipient` is the object that was sent the notification (`to`).
- `sender` is the object that sent the notification (`from`).
- `trigger` is the object that triggered the notification (`self`).
- `description` is the interpolated description for the notification (`message`).
- `link` is the interpolated/converted reference link for the notification (`link`).

### Automatic Associations

NotifyOn automatically attaches an association to the model in which you've called `notify_on`. For example, if you added notifications to a `Message` model, then an instance of a message -- let's call it `message` -- would have access to its notifications through `message.notifications`. It also eager loads the `recipient` and `sender` associations on the notification to avoid running into `N+1` problems.

You can attach notifications to the `to` argument you pass to `notify_on`, but you must state that explicitly. See the _receives\_notifications_ section above.

### Pusher (Real-Time Notifications)

[Pusher](https://pusher.com/) is a service that provides a websocket to access real-time information. While it is up to you to configure and support Pusher in your application, NotifyOn helps out a little.

You must specify `pusher` in a `notify_on` configuration. And if you do, you must also add your credentials to `config/initializers/notify_on.rb`. If everything is in place, NotifyOn will trigger an event in Pusher with the `notification` and `trigger`, so you can get any relevant associations.

### Overriding Mailer Class

Email notifications use the `NotifyOn::NotificationMailer` class. You may need to customize the default class for your application. You can do this by setting `mailer_class` in `config/initializers/notify_on.rb`.

### Conditionally Sending Email

If you're using email notifications, you can choose to conditionally trigger the message. For example, say you have a `Project` model, and you don't want to send notifications if the project is private or stale.

Using the `send_unless` option, you can pass a symbol representing an instance method on your trigger (a _project_ object in this case), and have it return a boolean.

It might look something like this:

```ruby
class Project < ActiveRecord::Base

  belongs_to :user

  notify_on :to => :user, :message => 'xxx', :link => 'project_path(:self)',
            :email => { :send_unless => :private_or_stale? }

  private

    def private_or_stale?
      # Assume `private` is a boolean field in your table, and `closed_at` is
      # a datetime field.
      private? || (closed_at.present? && closed_at <= DateTime.now)
    end

end
```

If `private_or_stale?` returns `true`, your email **will not** be sent, but if it returns `false`, then the email will be sent.
