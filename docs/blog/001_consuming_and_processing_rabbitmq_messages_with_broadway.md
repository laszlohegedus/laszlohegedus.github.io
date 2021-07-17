# Consuming and Processing RabbitMQ Messages with Broadway

`2020-01-15`

I recently got the chance to work with RabbitMQ again and the team decided to give BroadwayRabbitMQ a go. The online documentation is good enough to get up and running, but I would like to share a few tips and tricks. Introducing RabbitMQ and Broadway is not covered in this post, I assume that the reader is familiar enough with them.

## Starting Up and Basic Configuration

In order to process messages coming from RabbitMQ, we need to write a `Broadway` module that specifies `BroadwayRabbitMQ.Producer` as its producer:

```elixir
defmodule RabbitmqBlog.Processor do
  use Broadway

  alias Broadway.Message

  def start_link(_opts) do
    producer_config =
      Application.get_env(:rabbitmq_blog, :rabbitmq_broadway_producer)
      |> Keyword.put(:queue, "rabbitmq.blog")

    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [
        module: {BroadwayRabbitMQ.Producer, producer_config},
        stages: 2
      ],
      processors: [
        default: [
          stages: 50
        ]
      ]
    )
  end

  def handle_message(_, message, _) do
    IO.inspect(message.data, label: "Got message")
    message
  end
end

```

Where `:rabbitmq_broadway_producer` is defined in `config.exs` as follows:

```elixir
config :rabbitmq_blog, :rabbitmq_broadway_producer,
  connection: [
    host: System.get_env("RABBITMQ_HOST"),
    port: String.to_integer(System.get_env("RABBITMQ_PORT", "5672")),
    username: System.get_env("RABBITMQ_USERNAME"),
    password: System.get_env("RABBITMQ_PASSWORD")
  ]
```

The value of `connection` is passed on to [AMQP.Connection.open/1](https://hexdocs.pm/amqp/AMQP.Connection.html#open/1). See its documentation for the available options.

Storing the config in `config.exs` (or rather `releases.exs`) is my preferred choice, but not the only way. It gives a convenient way of reading secrets from environment variables in production.

If we start our app now and open the RabbitMQ Management UI, we can see that the Broadway producer created two connections with one channel on each connection. This is pretty much expected, as the number of connections depend on the number of stages.

### Queue declare options and bindings

I strongly believe that creating the queues and bindings in RabbitMQ is the responsibility of consumers and it should happen inside the application. This is pretty well supported by `BroadwayRabbitMQ` by the `:declare` and `:bindings` options that we can pass to the producer.

Consult [AMQP.Queue.declare/3](https://hexdocs.pm/amqp/AMQP.Queue.html#declare/3) for the options that you can pass in `:declare`.

For example, we can make `BroadwayRabbitMQ.Producer` take care of declaring the queue for us, if we add the `:declare` option to its config:

```elixir
producer_config =
  producer_config
  |> Keyword.put(:declare, [
      durable: false,
      arguments: [{"x-message-ttl", :long, 10_000}] # 10 seconds
    ])
```

The option `:bindings` is useful, if the routing logic is more or less static and does not involve exchange-to-exchange bindings. Each binding is a pair `{exchange_name, binding_options}` and they are passed to [AMQP.Queue.bind/4](https://hexdocs.pm/amqp/AMQP.Queue.html#bind/4). For example, `{"amq.direct", [routing_key: "blog-messages"]}` declares that a binding should be created from the exchange `amq.direct` to the queue `rabbitmq.blog` via routing key `blog-messages`:

```elixir
producer_config =
  producer_config
  |> Keyword.put(:bindings, [
      {"amq.direct", [routing_key: "blog-messages"]}
    ])
```

## Metadata

By default the processors receive only the body of the message in the `:data` field of `Broadway.Message`. This may be enough if the body contains all the information that is required for processing it, but RabbitMQ exposes several additional fields on the message. To see them, we have to specify the `:metadata` option as a list of atoms when configuring the producer. In general most of the option names from [AMQP.Basic.publish/5](https://hexdocs.pm/amqp/AMQP.Basic.html#publish/5-options) can be used, except for the special flags `:mandatory` and `:immediate`. In addition to basic publish metadata the server adds the following information: `:cluster_id`, `:consumer_tag`, `:delivery_tag`, `:exchange`, `:redelivered`, `:routing_key`.

For example, if we need `:routing_key` to process the message, we should state it in the processor config:

```elixir
producer_config =
  producer_config
  |> Keyword.put(:metadata, [
      :routing_key
    ])
```

## Message Acknowledgements

To configure how our `BroadwayRabbitMQ` producer should handle message acknowledgements we can specify the options `:on_success` and/or `:on_failure`. Their values can be `:ack`, `:reject`, `:reject_and_requeue`, `:reject_and_requeue_once`. Chosing the right method can be difficult. In general, setting `:on_success` to `:ack` makes sense, if `handle_message` returns successfully only when the message is actually processed. Deciding when to requeue a message is more complicated, since we cannot always be sure why processing failed. Logging and thorough testing comes handy. It also matters whether our application is prepared to handle duplicate messages, which I strongly recommend to take into account.
