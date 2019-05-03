---
title: "Simple Recurring Jobs in Phoenix Apps"
date: 2019-05-03T19:18:44+02:00
draft: false
toc: false
images:
tags: 
  - elixir
  - phoenix
---

There is often a need to do some work repeatedly in any non-trivial application. Maybe send a daily report or purge unused resources. Let's see how we can do that in a Phoenix app. 

## It's all Erlang underneath
It is pretty straightforward to do recurring jobs due to Elixir being a language that targets Erlang VM and therefore has access to lightweight concurrency with  [supervisors](https://hexdocs.pm/elixir/Supervisor.html) and child processes. In fact, let's check the `application.ex` file in a typical Phoenix app. This file is auto-generated when bootstrapping a new Pheonix app with `mix phx.new` command and is responsible for starting the app:

```elixir
defmodule DemoApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      DemoApp.Repo
    ]

    Supervisor.start_link(
      children,
      strategy: :one_for_one,
      name: DemoApp.Supervisor
    )
  end
end

```

Ok, so our DemoApp creates a supervisor that starts child processes. In this example it's just one child, but there's nothing keeping us from having more. And that's exactly what we'll do to run custom jobs.

Just a short note on supervisor strategies. A supervisor strategy instructs supervisor what to do when a child process dies. The one used in the example, `:one_for_one`, means that if a child process terminates, then that process is restarted. There are other strategies as well, like killing all children if one dies or doing nothing and so on, but for our use case `:one_for_one` is exactly what we want.

## Simple job

Let's do following:

1. Implement our job as child process
2. Schedule the work one minute after initialization.
3. After work is done, schedule it again.

Once we have our job, we just have to pass it to `children` list for the supervisor. Building upon the [example](https://hexdocs.pm/elixir/Supervisor.html#module-examples) in Elixir docs our job could look like this:

```elixir
defmodule DemoApp.Job do
  use GenServer

  def start_link(state) do
    GenServer.start_link(__MODULE__, state, name: __MODULE__)
  end

  def init(state) do
    schedule()
    
    {:ok, state}
  end

  def handle_info(:work, state) do
    # the actual work to be done
    IO.puts("Running job ...")
    
    # work is done, let's schedule again
    schedule()
    
    {:noreply, state}
  end
  
  defp schedule() do
    # schedule after one minute
    Process.send_after(self(), :work, 60000)
  end

end
```

In essence, our Job is a process that waits for `:work` message to do its work. What we want is that `:work` message arrives once per 60 seconds and this is done in two steps:

1. By calling `schedule/0` function during initialization.
2. And by calling `schedule/0` every time after work is done.

You may be thinking, "This won't work properly. What if there's an error in `handle_info/2` function? We should catch it, otherwise the job won't be re-scheduled.". But that's in fact not necessary, because the supervisor will take care of restarting a failed child process. This keeps the code simple and is one of the benefits we get from Erlang's concurrency.

Now that we have the Job implementation, we have to make sure it's started when the app starts:

```elixir
defmodule DemoApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      DemoApp.Repo,
      DemoApp.Job
    ]

    Supervisor.start_link(
      children,
      strategy: :one_for_one,
      name: DemoApp.Supervisor
    )
  end
end
```

That's right, it's just one line added to the children array.

## Wait, it's not that simple

It really is, but only in case your jobs don't require complex fallback logic. For example, it may not be desired that a job is just re-run if it fails or you'd want that it's only retried couple of times before giving up. If that's the case, then you may want to look into third party dependencies such as [Quantum](https://hexdocs.pm/quantum/readme.html) or [Verk](https://github.com/edgurgel/verk). Or implement this behaviour yourself, but be ready for a rabbit hole of job scheduling.

## Code reuse

If you have many different jobs (well, more than one), then it's annoying to copy all that boilerplate for every job implementation. Ideally, the only two things we'd want in a job is the time interval and the code to do the work. We can achieve this using Elixir [macros](https://elixir-lang.org/getting-started/meta/macros.html). Here's how such a job could look like:

```elixir
defmodule DemoApp.DemoJob do
  use DemoApp.Job

  # every 10 minutes
  @job_interval 600

  def get_interval(), do: @job_interval

  def work() do
    IO.puts("DemoJob working ...")
  end
end
```

The `use DemoApp.Job` line means that the DemoJob will use `Job` macro, which is composed mostly of the boilerplate we've seen in the 'Simple job' section:

```elixir
defmodule DemoApp.Job do
  @callback work() :: any
  @callback get_interval() :: Integer.t()

  defmacro __using__(_params) do
    quote do
      @behaviour DemoApp.Job

      use GenServer

      def start_link(state) do
        GenServer.start_link(__MODULE__, state, name: __MODULE__)
      end

      def init(state) do
        schedule()

        {:ok, state}
      end

      def handle_info(:work, state) do
        work()

        schedule()

        {:noreply, state}
      end

      defp schedule() do
        Process.send_after(self(), :work, get_interval() * 1000)
      end
    end
  end
end
```

The `DemoApp.Job` module specifies two things:

1. The [behaviour](https://elixir-lang.org/getting-started/typespecs-and-behaviours.html#behaviours) with two callbacks. Behaviours are essentially the same as interfaces in some other languages, which means that other modules using this module will have to implement `work/0` and `get_interval/0` functions.
2. The macro with the job boilerplate code. You can think of macros as text blocks that are copied verbatim to every module using the macro.

Now any job implementation that uses `DemoApp.Job` module will be much shorter and in case we have to make changes to boilerplate code, we only have to do it in one place.
