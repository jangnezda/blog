---
title: "Upgrading Phoenix projects to latest Elixir"
date: 2019-08-08T18:53:37+02:00
draft: false
toc: false
images:
tags: 
  - elixir
  - phoenix
---

Elixir 1.9 has been [released](https://elixir-lang.org/blog/2019/06/24/elixir-v1-9-0-released/) about a month ago. This release is important, because of two things:

- It adds releases functionality to the core. This means that Elixir projects no longer need to depend on Distillery.
- According to authors, there are no plans for new features to the language:
    
    > As mentioned earlier, releases was the last planned feature for Elixir. We don’t have any major user-facing feature in the works nor planned. I know for certain some will consider this fact the most excing part of this announcement!
    
    This means a couple of things. Firstly, any further upgrades should be painless. And secondly, a project using Elixir 1.9 can be reasonably sure that it has access to all features for the foreseeable future.
    
This spurred me to upgrade all our Elixir projects to 1.9 and I've also decided to upgrade Phoenix at the same time to the [latest version](https://github.com/phoenixframework/phoenix/blob/v1.4.9/CHANGELOG.md) (1.4.9). Below is a list of problems that you may encounter and/or recommendations if you decide to upgrade as well:

### 1. Erlang compilation error

We're using [asdf](https://github.com/asdf-vm/asdf) and upgrading meant changing `.tool-versions` file to new Erlang/OTP and Elixir versions:

```
erlang 22.0.7
elixir 1.9.1-otp-22
```

and running `asdf install` in the project directory. Unfortunately, compilation of new Erlang failed on my laptop with 'missing separator' error as outlined in [this issue](https://github.com/asdf-vm/asdf-erlang/issues/100). To fix it, I had to remove and add Erlang plugin:

```
asdf plugin-remove erlang
asdf plugin-add erlang
```

Then run `asdf install` again and it worked.
 

### 2. Config files

All config files in an umbrella app are now in the top-level `config` directory. If you have any libraries, plugs, etc. configs that have configuration depend on paths, then the app may fail unless those paths are updated.

### 3. Bureaucrat

Bureaucrat is a tool to generate API documentation. After upgrading Phoenix to 1.4.9, it failed to generate the docs due to following error:

```
[error] GenServer #PID<0.595.0> terminating
** (Protocol.UndefinedError) protocol Enumerable not implemented for nil of type Atom. This protocol is implemented for the following type(s): Ecto.Adapters.SQL.Stream, Postgrex.Stream, DBConnection.Stream, DBConnection.PrepareStream, Timex.Interval, HashSet, Range, Map, Function, List, Stream, Date.Range, HashDict, GenEvent.Stream, MapSet, File.Stream, IO.Stream
    (elixir) lib/enum.ex:1: Enumerable.impl_for!/1
    (elixir) lib/enum.ex:141: Enumerable.reduce/3
    (elixir) lib/enum.ex:3023: Enum.each/2
    (elixir) lib/enum.ex:789: anonymous fn/3 in Enum.each/2
    (stdlib) maps.erl:232: :maps.fold_1/3
    (elixir) lib/enum.ex:1964: Enum.each/2
    (bureaucrat) lib/bureaucrat/swagger_slate_markdown_writer.ex:238: Bureaucrat.SwaggerSlateMarkdownWriter.write_operations_for_tag/4
    (elixir) lib/enum.ex:783: Enum."-each/2-lists^foreach/1-0-"/2
    (elixir) lib/enum.ex:783: Enum.each/2
    (elixir) lib/enum.ex:1340: anonymous fn/3 in Enum.map/2
    (stdlib) maps.erl:232: :maps.fold_1/3
    (elixir) lib/enum.ex:1964: Enum.map/2
    (bureaucrat) lib/bureaucrat/formatter.ex:10: Bureaucrat.Formatter.handle_cast/2
    (stdlib) gen_server.erl:637: :gen_server.try_dispatch/4
    (stdlib) gen_server.erl:711: :gen_server.handle_msg/6
    (stdlib) proc_lib.erl:249: :proc_lib.init_p_do_apply/3
```

The root problem is in `phoenix_swagger` library that hasn't been updated yet to work with latest Phoenix versions. Because of it, the swagger file is generated without routes and so on, which causes `Bureaucrat.SwaggerSlateMarkdownWriter` module to fail.

The `phoenix_swagger` authors have already fixed this issue in master branch, but haven't yet published the new package to Hex registry. Check the [github issue](https://github.com/xerions/phoenix_swagger/issues/232) for more information. The temporary solution is to point your `mix.exs` to master branch instead:

```elixir
  defp deps do
    [
      {:phoenix, "~> 1.4.9"},
      {:phoenix_swagger, github: "xerions/phoenix_swagger", branch: "master"},
      ...
    ]
  end
```

### 4. Releases

Elixir 1.9 now has built-in [releases](https://hexdocs.pm/mix/Mix.Tasks.Release.html) functionality. It is similar to what Distillery was, but there are some differencies. Be sure to read the documentation to understand those differencies. Some observations:

1. There are `vm.args.eex` and `env.sh.eex` that replace `vm.args`. This split is meant to clearly move env vars like erlang cookie and node name out of `vm.args`, which should only be used to erlvang vm-specific settings (like kernel options and such).

2. Runtime env variables finally work in a sane way. You define them in `config/releases.exs`:

    ```elixir
    import Config

    config :my_app, MyApp.Repo,
      username: System.fetch_env!("DATABASE_USERNAME"),
      password: System.fetch_env!("DATABASE_PASSWORD")

    ...
    ```
    Note that import line does not use Mix's Config, because Mix is not available within app release. The variables defined in `config/releases.exs` will override those in `config/prod.exs` when running a release.
    
3. There is no notion of commands that were used in Distillery to define custom script commands (most commonly to migrate/seed database after deploy). Instead, you have to add a custom release step that copies your scripts to realease's `/bin` directory:

    1. Make sure to have your custom scripts in `./rel/bin`.
        
        For example, a `./rel/bin/migrate.sh` could look like:
    
        ```
        #!/bin/sh

        CURR_DIR=`dirname "$0"`

        $CURR_DIR/myapp eval "MyApp.ReleaseTasks.migrate"
        ```
    
    2. Then in `mix.exs`:
    
        ```
        def project do
          [
            ...

            releases: [
              myapp: [
                include_executables_for: [:unix],
                applications: [
                  myapp_web: :permanent
                ],
                steps: [:assemble, &copy_bin_files/1]
              ]
            ],
        
            ...
          ]
        end
    
        defp copy_bin_files(release) do
          File.cp_r("rel/bin", Path.join(release.path, "bin"))
          release
        end
        ```

### 5. Distillery

Distillery is not needed anymore, but if you still need it, remember that mix commands have been renamed: `mix release` -> `mix distillery.release`, etc.

### 6. Credo

If you are using [Credo](https://github.com/rrrene/credo), upgrade it as well to 1.1.2+, because it has updated rule for Elixir 1.9.

### 7. Docker

Don't forget to update the `Dockerfile` as well, if you are using Docker. IF you are using builder and runner stages, then you may update like this:

1. Builder:

    ```
    FROM elixir:1.9.0-alpine AS builder

    RUN apk update && \
        apk add --no-cache \
          gcc \
          git \
          make \
          musl-dev
    ...
    ```

2. Runner:

    ```
    FROM alpine:3.9 AS runner
    RUN apk add -U bash libssl1.1
    COPY --from=builder /app/_build/prod/rel/my_app /app
    ...
    ```

    Note that the runner stage builds from Alpine 3.9. It's common mistake to leave it at Alpine 3.8, but the app won't run properly, because `elixir:1.9.0-alpine` builds upon Alpine 3.9.
