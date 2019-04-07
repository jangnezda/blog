---
title: "Custom Session Store in Phoenix App"
date: 2019-04-05T19:56:59+02:00
draft: false
toc: false
images:
tags: 
  - elixir
  - phoenix
---

In this post I'll show you how to build a custom session store in a Phoenix-based app. It's super easy!

The [Phoenix framework](https://github.com/phoenixframework/phoenix) is one of the most popular web frameworks in Elixir community. In fact, I and many other developers have been introduced to Elixir through a Phoenix app. One of the great things about Phoenix is that it's not opinionated framework - it provides sensible defaults, but may be changed or extended easily. This applies to session storage as well.

Phoenix uses [Plug.Session](https://hexdocs.pm/plug/Plug.Session.html) to handle http sessions. A plug is a function that receives a connection, does its thing and then returns a connection. You can think of it as middleware. The session plug provides two strategies for saving session state:

1. `Plug.Session.ETS`: session state is stored in an in-memory [ETS table](https://elixir-lang.org/getting-started/mix-otp/ets.html).
2. `Plug.Session.COOKIE`: session state is stored within the cookie itself (cookie is encrypted, of course).

These two strategies are totally fine for certain types of apps, but may not be the right choice for your app. Maybe you don't want session state to propagate to users' browsers or have a distributed deployment where different app instances should be able to recognize user's session cookie. Whatever the reason may be, you can tell Phoenix to use custom session store when initializing `Plug.Session` middleware.

Just a note before we dive into implementation. The session 'store' is a bit of misnomer. It won't actually store anything, but act more like a glue between Phoenix and a real backing store. Backing store may be a Postgres database, Redis or Mongo, for example.  

## Step one: implementation

Let's start by creating a new module that will be our custom store. We need to:

1. Use `@behaviour Plug.Session.Store` to tell that the module can be used as session store. `Plug.Session` will then call our store as needed (login, logout, etc.).
2. Implement four functions:
    1. `init/1` - do any needed initialization. The returned `opts` are later passed to following three functions.
    2. `get/3` - fetch session data.
    3. `put/4` - save (new) session.
    4. `delete/3` - remove invalidated session.

In our example, we'll assume that sessions will be stored in a Postgres database that has a table named `sessions` and our app already has an api and schema to read and write to the database. Further, we want our sessions to be valid for certain amount of time and after that become invalid. For example, this could be the Ecto schema for sessions:

```elixir
  schema "sessions" do
    field(:session_cookie, :string)
    field(:valid_from, :utc_datetime)
    field(:valid_to, :utc_datetime)

    belongs_to(:user, MyApp.Domain.User)
  end
```

Very basic structure as it only contains fields for cookie and validity interval.

Alright, let's start. Create a new file somewhere in app's lib directory structure. In this case, I'd put it right next to `endpoint.ex` in `lib/myapp` (or `apps/myapp_web/lib/myapp_web` in an umbrella project). Let's start with `init/1` and `get/3`:

```elixir
defmodule MyApp.SessionStore do
  @moduledoc """
  Session store that uses Postgres as backing storage.
  """
  
  @behaviour Plug.Session.Store

  alias MyApp.Api.Sessions
  alias MyApp.Domain.Session

  def init(opts) do
    # By default, sessions will be valid for 1 hour
    max_age = Keyword.get(opts, :store_max_age, 3600)

    %{max_age: max_age}
  end

  def get(_conn, cookie, _opts)
      when cookie == ""
      when cookie == nil do
    {nil, %{}}
  end
  
  def get(_conn, cookie, _opts) do
    session = Sessions.get_by_session_cookie(cookie)
    get_for_session(cookie, session)
  end

  defp get_for_session(_cookie, nil), do: {nil, %{}}

  defp get_for_session(cookie, %Session{user_id: user_id, valid_to: valid_to}) do
    now = DateTime.utc_now()

    if DateTime.compare(now, valid_to) == :lt do
      {cookie, %{"user_id" => id}}
    else
      {nil, %{}}
    end
  end
  
  # continued below ...
end
```

The `init/1` function is passed the `Plug.Session` configuration params, which we check for `store_max_age` or default to 1 hour if param wasn't set.

The `get/3` is more insteresting. First, we pattern match on empty cookie and in that case return empty values as well. Otherwise, we try to load the session using that cookie. If the session is found, then we check if it's still valid.

Next on, the `put/4` function:

```elixir
  def put(_conn, cookie, term, opts) do
    user_id = term["user_id"]

    if cookie == nil and user_id != nil do
      %{max_age: max_age} = opts

      valid_from = DateTime.utc_now()
      valid_to = DateTime.from_unix!(DateTime.to_unix(valid_from) + max_age)

      new_cookie = :crypto.strong_rand_bytes(64) |> Base.encode64()

      Sessions.create!(%{
        session_cookie: new_cookie,
        valid_from: valid_from,
        valid_to: valid_to,
        user_id: user_id
      })

      session_cookie
    else
      cookie
    end
  end
```

In `put/4`, we'll only create new session if there isn't already a session cookie and if the user is known. Otherwise, we just return the input `cookie`.

The `delete/3` is only couple of lines:

```elixir
  def delete(_conn, cookie, _opts)
      when cookie == ""
      when cookie == nil do
    :ok
  end

  def delete(_conn, cookie, _opts) do
    Sessions.delete_by_session_cookie(sid)
    :ok
  end
```

We apply similar pattern as we did with `get/3`.

And that's all there is to it! The heavy work is done by `Plug.Session` and `MyApp.Api.Sessions`. In essence, our custom store is just a way to bind the session plug with database.
 
## Step two: configuration

We need to update `endpoint.ex`, because that's where the session plug is configured. We'll tell it to use our custom store and set the max age to 1 day:

```elixir
# ...

max_age = 86_400

plug Plug.Session,
  store: MyApp.SessionStore,
  key: "_myapp_sid",
  max_age: max_age,
  store_max_age: max_age
  
# ...
```

The `key` config param is used to name the cookie in request headers (and subsequently in browser), for example:

```
set-cookie: _myapp_sid=bZsiztY6G+7UHr5BWMpWLVpUnr3paDTktFU/S1Jh5B1COqhaEyWaOuunrvJ/D8FvcMQzl1nw/z+1blhhtlFgAQ==; path=/; expires=Sun, 7 Apr 2019 11:28:31 GMT; max-age=86400; HttpOnly
```

Then there are two config params for max age. `max_age` is used internally by session plug and is unfortunately not exposed in options passed to the store. Therefore, we've defined another config param, which won't be 'eaten' by session plug.

## That's all folks

As you can see, it's very easy to implement a custom session store in Phoenix app. And for me personally, this is one of the best things about the framework: it's easy to extend it and the result is most of the time less than 100 lines of code.