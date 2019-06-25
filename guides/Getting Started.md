# Getting started

EventStore is [available in Hex](https://hex.pm/packages/eventstore) and can be installed as follows:

  1. Add eventstore to your list of dependencies in `mix.exs`:

      ```elixir
      def deps do
        [{:eventstore, "~> 0.16"}]
      end
      ```

      Run `mix deps.get` to install the new dependency.

  2. Define an event store module for your application:

      ```elixir
      defmodule MyApp.EventStore do
        use EventStore, otp_app: :my_app

        # Optional `init/1` function to modify config at runtime.
        def init(config) do
          {:ok, config}
        end
      end
      ```

  3. Add a config entry containing the PostgreSQL database connection details for your event store module to each environment's mix config file (e.g. `config/dev.exs`):

      ```elixir
      config :my_app, MyApp.EventStore,
        serializer: EventStore.JsonSerializer,
        username: "postgres",
        password: "postgres",
        database: "eventstore",
        hostname: "localhost",
        # OR use a URL to connect instead
        url: "postgres://postgres:postgres@localhost/eventstore",
        pool_size: 10
        pool_overflow: 5
      ```

  The database connection pool configuration options are:

  - `:pool_size` - The number of connections (default: `10`).
  - `:pool_overflow` - The maximum number of overflow connections to start if all connections are checked out (default: `0`).

  4. Add your event store module to the `event_stores` list for your app in mix config:

      ```elixir
      # config/config.exs
      config :my_app, event_stores: [MyApp.EventStore]
      ```

      This ensures the event store mix tasks can be run without specifying the event store as a command line argument (e.g. `mix event_store.init -e MyApp.EventStore`).

  5. Create and initialize the event store database and tables using the `mix` tasks:

      ```console
      $ mix do event_store.create, event_store.init
      ```

  6. The final piece of configuration is to setup your event store as a supervisor within your application's supervision tree (e.g. in `lib/my_app/application.ex`, inside the `start/2` function):

      ```elixir
      defmodule MyApp.Application do
        use Application

        def start(_type, _args) do
          children = [
            MyApp.EventStore
          ]

          opts = [strategy: :one_for_one, name: MyApp.Supervisor]
          Supervisor.start_link(children, opts)
        end
      end
      ```

## Using an existing database

You can use an existing PostgreSQL database with EventStore by running the following `mix` task to create and initialize the event store tables:

```console
$ mix event_store.init
```

## Reset an existing database

To drop an existing EventStore database and recreate it you can run the following `mix` tasks:

```console
$ mix do event_store.drop, event_store.create, event_store.init
```

*Warning* this will delete all EventStore data.

## Event data and metadata data type

EventStore has support for persisting event data and metadata as either:

  - Binary data, using the [`bytea` data type](https://www.postgresql.org/docs/current/static/datatype-binary.html), designed for storing binary strings.
  - JSON data, using the [`jsonb` data type](https://www.postgresql.org/docs/current/static/datatype-json.html), specifically designed for storing JSON encoded data.

The default data type is `bytea`. This can be used with any binary serializer, such as the Erlang Term format, JSON data encoded to binary, and other serialization formats.

### Using the `jsonb` data type

The advantage this format is that it allows you to execute ad-hoc SQL queries against the event data or metadata using PostgreSQL's native JSON query support.

To enable native JSON support you need to configure your event store to use the `jsonb` data type. You must also use the `EventStore.JsonbSerializer` serializer, to ensure event data and metadata is correctly serialized to JSON, and include the Postgres types module (`EventStore.PostgresTypes`) for the Postgrex library to support JSON.

```elixir
# config/config.exs
config :my_app, MyEventStore,
  column_data_type: "jsonb"
  serializer: EventStore.JsonbSerializer,
  types: EventStore.PostgresTypes
```

Finally, you need to include the Jason library as a depencency in `mix.exs` to enable Postgrex JSON support and then run `mix deps.get` to install.

```elixir
# mix.exs
defp deps do
  [{:jason, "~> 1.1"}]
end
```

These settings must be configured *before* creating the EventStore database. It's not possible to migrate between `bytea` and `jsonb` data types once you've created the database. This must be decided in advance.
