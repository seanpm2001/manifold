# Manifold

[![Master](https://travis-ci.org/discordapp/manifold.svg?branch=master)](https://travis-ci.org/discordapp/manifold)
[![Hex.pm Version](http://img.shields.io/hexpm/v/manifold.svg?style=flat)](https://hex.pm/packages/manifold)

Erlang and Elixir make it very easy to send messages between processes even across the network, but there are a few pitfalls.

- Sending a message to many PIDs across the network also copies the message across the network that many times.
- Send calls cost about 70 µs/op so doing them in a loop eventually gets too expensive.

[Discord](https://discordapp.com) runs a single `GenServer` per Discord server and some of these have over 30,000 PIDs connected
to them from many different Erlang nodes. Increasingly we noticed some of them getting behind on processing their message queues
and the culprit was the cost of 70 µs per `send/2` call multiplied by connected sessions. How could we solve this?

Inspired by a [blog post](http://www.ostinelli.net/boost-message-passing-between-erlang-nodes/) about boosting performance of
message passing between nodes, Manifold was born. Manifold distributes the work of sending messages to the remote nodes of the
PIDs, which guarantees that the sending processes at most only calls `send/2` equal to the number of involved remote nodes.
Manifold does this by first grouping PIDs by their remote node and then sending to `Manifold.Partitioner` on each of those nodes.
The partitioner then consistently hashes the PIDs using `:erlang.phash2/2`, groups them by number of cores, sends to child
workers, and finally those workers send to the actual PIDs. This ensures the partitioner does not get overloaded and still provides
the linearizability guaranteed by `send/2`.

The results were great! We observed packets/sec drop by half immediately after deploying. The Discord servers in question also
were finally able to keep up with their message queues.

![Packets Out Reduction](priv/packets.png)

There is an optional and experimental `:offload` send mode which offloads on the send side the `send/2` calls to the receiving
nodes to a pool of `Manifold.Sender` processes. To maintain the linearizability guaranteed by `send/2`, the same calling process
always offloads the work to the same `Manifold.Sender` process. The size of the `Manifold.Sender` pool is configurable. This send
mode is optional because its benefits are workload dependent. For some workloads, it might degrade overall performance. Use with
caution.

Caution: To maintain the linearizability guaranteed by `send/2`, do not mix calls to Manifold with and without offloading. Mixed
use of the two different send modes to the same set of receiving nodes would break the linearizability guarantee.

## Usage

Add it to `mix.exs`

```elixir
defp deps do
  [{:manifold, "~> 1.0"}]
end
```

Then just use it like the normal `send/2` except it can also take a list of PIDs.

```elixir
Manifold.send(self(), :hello)
Manifold.send([self(), self()], :hello)
```

To use the experimental `:offload` send mode, make sure the `Manifold.Sender` pool size is appropriate for the
workload:

```elixir
config :manifold, senders: <size>
```

Then:

```elixir
Manifold.send(self(), :hello, send_mode: :offload)
```

### Configuration
Manifold takes a single configuration option, which sets the module it dispatches to actually call send. The default
is GenServer. To set this variable, add the following to your `config.exs`:

```elixir
config :manifold, gen_module: MyGenModule
```

In the above instance, `MyGenModule` must define a `cast/2` function that matches the types of `GenServer.cast`.


## License

Manifold is released under [the MIT License](LICENSE).
Check [LICENSE](LICENSE) file for more information.
