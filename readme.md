## Promises

Many things in life are better when they're varied: the food you eat, the places you hang out, the books you read, the problems you solve. Life and most things in it are better for a dash of the unexpected.

Some things, though, don't get better from unpredictability -- APIs among them. Like any user interface, an API works best when it is uniform and regular.  No one wants to code for different styles of responses or errors from different endpoints on the same system. (I talked about this recently at BaRuCo, for those who enjoy audio and video.)

Keeping your API uniform can be a challenge, though. Like any other program, as your code grows and your programming style changes, little inconsistencies creep in.  Add in more programmer, the risk grows.  (Anyone who's worked with a massive API, like Facebook's, has seen this in action.)  It's entropy, and it's pretty inevitable.

### Application Data

An API provides data to its users: internal developers, external developers, or you yourself. Some of that data is user data, and that's straightforward.  It maps pretty clearly to data in your datastore.

What's more interesting (today) is that most APIs also provides data that you as a developer create: error responses are the primary example, but statuses, object types, flags, and so on also count.  These sets should be consistent and informative; for instance, whenever a given problem occurs, regardless of where in your code, the error response should be the same.  Whoever receives a value from your system should have a clear understanding of what it is and what it means for them as a developer.

We all know how frustrating it is when this doesn't happen -- to use the error example again, who hasn't received an opaque and unexpected error message from someone else's system? (Like my all-time [favorite Facebook error](https://github.com/arsduo/koala/issues/211)). The people behind the API are, of course, not out to make your life annoying; it's just simply easier to type in whatever seems right at the time and call it a day.

### Make Promises

Promises offers a way around this mess: instead of defining values at the point of use, you create discrete entries in a data source, and then reference them. It's absurdly simple; in fact, I'm guessing most of us have written something like this before.  It's a bit like a useful enum on steroids.

Basically:

* _Definition_: Whatever data you're sending out (errors, event types, etc.), you define in a data source (YAML, JSON, etc.).  This includes not only the minimal data, but also the human-readable context a developer (or you, coming back to your code) would want.
* _Referencing_: You reference these data via Promises, rather than directly. This is the vital enforcement step: if you try to reference a key that doesn't exist, you get an error.
* _Verifying_: In your tests, you can use the included be_promise matcher to ensure an endpoint is giving back the right data.
* _Explaining_: Optionally but highly encouragedly, you can mount the provided Rack app to provide information to your developers on the data you've defined.

Let's look at a simple example of using Promises to manage errors -- actually, a simplified version of promises_of_trouble, a [Promises-based error handler](link).

```yml

# config/data/errors.yml

not_authorized:
  status: 401
  message: You're not authorized to perform this action
  explanation: You've made a request without the appropriate authorization parameters.  Go get an access token, then we'll talk.

validation_error:
  status: 422
  message: Oops!  You sent in some bogus data.
  explanation: One or more of the values you sent in, we couldn't work with.  Take a look at the rest of the error hash; it should be straightforward to figure out what's wrong.
```

```ruby
# initializers/error_promises.rb
module MyApp
  ErrorPromises = Promises.make do
  # data_source could also be json or a Proc that returns a Hash
  data_source yaml: "config/data/errors.yml"
  # there are other options, but this is all we need

  # transform defines how the Promise is turned to JSON
  # for our error, we don't want to include the status in the response body
  prepare_for_output do |data|
    data.except(:status)
  end
  end
end

# ApplicationController

rescue_from StandardError do |error|
  render error(error.is_a?(NoUserError) ? :not_authorized : :unexpected_error)
end

# a convenience method
def error(type, additional_data = {})
  error = MyApp::ErrorPromises.data_for(type)
  # return a hash that can be provided directly to render
  {
    json: error, # excludes status, as defined above in prepare_for_output
    status: error[:status]
  }
end


# ResourceController

def update
  if object.update(:params)
    render json: object, status: :ok
  else
    render error(:validation_error, object.errors)
  end
end

# ResourceControllerTest

it "renders validation_error with the problems if bad data is supplied" do
  post "/update", {id: object_id}.merge(some_bad_data)
  body = JSON.parse(response.body)
  body.should keep_promise(:validation_error) # error data optional
end

# routes.rb
match "/info/errors" => MyApp::ErrorPromises::Documentation
```

```bash

curl http://myapp.com/info/errors

  not_authorized:
    status: 401
    message: You're not authorized to perform this action
    explanation: You've made a request without the appropriate authorization parameters.  Go get an access token, then we'll talk.

  validation_error:
    status: 422
    message: Oops!  You sent in some bogus data.
    explanation: One or more of the values you sent in, we couldn't work with.  Take a look at the rest of the error hash; it should be straightforward to figure out what's wrong.
```

This example was error handling, but you can easily also use this for sending out events of a specific type, managing statuses, etc. -- anything where you're defining a set of data by choosing arbitrary strings.  In most of those cases, your data file would be much simpler -- a status and an explanation -- but the principle of documenting and sticking to a list of types (and publicizing them) is the same.

Promises can be used for both externally-facing data (as originally intended) or for internal sets.  The benefits of the Promises approach isn't so much to expose certain data in certain formats as to force you to think constantly about how your app behaves and should behave.

Just as research shows that stopping and photographing your food makes you a better, healthier eater, I think having to stop and document your application's behavior will make you a better app designer.

### Thanks

To 6Wunderkinder, for their support for open source software.