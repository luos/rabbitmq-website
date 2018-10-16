<!--
Copyright (c) 2007-2018 Pivotal Software, Inc.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Quorum Queue NOSYNTAX

## Overview

The quorum queue is a new (since 3.8) queue type for RabbitMQ implementing a persistent,
replicated fifo queue using [the Raft consensus algorithm](https://raft.github.io/).

The quorum queue type is an alternative to persistent [highly available (mirrored)
queues](ha.html) (HA queues) for use cases where data
safety is paramount.

#### [Motivation](#motivation)

Existing HA queues have technical limitations that makes it difficult to provide
comprehensible guarantees and simple failure semantics within RabbitMQ.

Certain network partitions / failure scenarios can result in HA queues
losing confirmed messages. Quorum queues are designed to provide simpler
semantics that users should find easier to reason about when designing
and operating their systems.

##  Usage

#### [Declaring](#declaring)

To declare a quorum queue set the `x-queue-type` queue argument to `quorum`
(the default is `classic`). This will declare a quorum queue with a member on
every currently configured RabbitMQ node.
After that the queue can be bound to any exchange just as any other
RabbitMQ queues.

It can also be created through the management UI using the queue type drop down
menu.

#### [Controlling the quorum queue size](#controlling-size)

By default the quorum queue will launch a member node on every configured RabbitMQ
node. This may not be desirable for very large RabbitMQ clusters or for cluster with
an uneven number of RabbitMQ nodes.
To control the number of quorum queue members set the `x-quorum-queue-size`
queue argument when declaring the queue. The queue size argument provided should
be an integer that is
smaller or equal to the current RabbitMQ cluster size. The quorum queue will be
launched to run on a random subset of the RabbitMQ cluster.

#### [Changing quorum queue members](#changing-members)

Once declared the RabbitMQ nodes a quorum queue resides on won't change even if the
members of the RabbitMQ cluster change. To re-balance after a RabbitMQ cluster change
quorum queues will have to be manually adjusted using the `rabbitmq-queues`
command line tool.

Two commands are provided:

`rabbitmq-queues add_member [-p <vhost>] <queue-name> <node>`

`rabbitmq-queues delete_member [-p <vhost>] <queue-name> <node>`

NB: to add and remove members a quorum needs to be available.

Care needs to be taken not to accidentally make a queue unavailable by losing
the quorum whilst performing membership changes.


## Behaviour to expect

Quorum queues are quorum systems and use consensus to ensure data consistency.

#### [Fault tolerance](#fault-tolerance)

Consensus systems can provide certain guarantees with regard to data safety.
These guarantees do mean that certain conditions need to be met before they
become relevant such as requiring a minimum of 3 RabbitMQ nodes to provide
fault tolerance and requiring more than half of members to be available to
work at all.

*Failure tolerance table*:

| Node count | Tolerated no. of failures |
| :--------: | :------: |
| 1          | 0        |
| 2          | 0        |
| 3          | 1        |
| 4          | 1        |
| 5          | 2        |
| 6          | 2        |
| 7          | 3        |

As the table above shows RabbitMQ clusters with fewer than 3 nodes do not
benefit fully from the quorum queue guarantees. RabbitMQ clusters with an even
number of RabbitMQ nodes do not benefit from having quorum queue members spread
over all nodes. For these systems the quorum queue size should be constrained to a
smaller uneven number of nodes.

Performance tails off quite a bit for quorum queue node sizes larger than 5.
We do not recommend running quorum queues on more than 7 RabbitMQ nodes. The
default quorum queue size is 5 and is controllable using the
`x-quorum-queue-size` queue argument.

#### [Data safety](#data-safety)

Quorum queues are designed to provide data safety under network partition and
failure scenarios. A message that was successfully confirmed back to the publisher
using the [publisher confirms](confirms.html) feature should not be lost as long as at
least a majority of RabbitMQ nodes hosting the quorum queue are not
permanently made unavailable.

Generally quorum queues favours data consistency over availability.

*_No guarantees are provided for messages that have not been confirmed using
the publisher confirm mechanism_*. They may never reach the queue.


#### [Availability](#availability)

Quorum queues should be able to tolerate a minority of queue members becoming unavailable
with no or little affect on availability. (NB: RabbitMQ may restart itself during
recovery and perform various actions that make this harder to ensure but in
principle this should be true).

If a quorum of nodes cannot be recovered (say if 2 out of 3 RabbitMQ nodes are
permanently lost) the queue is permanently unavailable and
will need to be force deleted and recreated.


## [Features compared to other queues](#feature-comparison)



| Feature | Classic / HA | Quorum |
| :-------- | :------- | ------ |
| Non-durable queues | yes | no |
| Exclusive queues | yes | no |
| Persistence | per message | always |
| Membership changes | automatic | manual  |
| TTL | yes | no |
| Queue size limits | yes | no |
| Lazy | yes | no |
| Priority | yes | no |
| Dead letter queues | yes | yes |
| Adheres to policies | yes | no |
| Reacts to memory alarm | yes | no |


## [Primary use case](#use-case)

Quorum queues are _not_ designed to be used everywhere. Their intended use is for
topologies where queues exist for a long time and are critical to some business
facility where fault
tolerance and data safety is more important than, say, low latency and advanced
queue features.
Examples would be incoming orders in a sales system or votes cast in an
election system where potentially losing messages would have a significant
business impact.

Stock tickers and instant messaging systems benefit less or not at all from
quorum queues.

Publishers should use publisher confirms as this is how clients can interact with
the quorum queue consensus system. Publisher confirms will only be issued once
a published message
has been successfully replicated to a quorum of nodes and is considered "safe"
within the context of the system.

Consumers should use manual acknowledgements to ensure messages that aren't
successfully processed are returned to the queue so that
another consumer can re-attempt processing.

## [When not to use it](#when-not-to-use-it)

When requiring:

* Low latency
* Transient or exclusive queue features.
* Very long queue backlogs (all nodes keep all messages in memory at all times).
* When not using manual acks and publisher confirms.


## [Semantics under failure](#failure-semantics)

The quorum queue requires a quorum of the declared
nodes to be available to function.
When a RabbitMQ node hosting a current quorum queue "leader" (in Raft parlance)
crashes or is stopped another node hosting a quorum queue "follower" will
be elected leader and resume operations. In contrast with classic HA
queues there is no explicit synchronization of "followers" (mirrors) following
a leader change
which means that availability should be resumed very shortly after the failure
event even when the queue has a significant backlog.

## [Performance](#performance)

Quorum queues are designed to trade latency for throughput and have been tested
and compared against HA queues in 3, 5 and 7 node configurations at several
message sizes. In all scenarios quorum queues have been observed to be have
higher or equal throughput to HA queues. When message sizes are small they can
achieve nearly 3x the throughput of HA queues.

As quorum-queues persist all data to disks before doing anything it is recommended
to use the fastest disks possible.

In general throughput performance decreases as message sizes increase.

Also larger quorum
queue cluster sizes are slower than smaller ones as more work need to be done
to replicate data and achieve consensus.

## [Configuration](#configuration)

There are a few new configuration parameters that can be tweaked.

Currently you have to use the [advanced](configure.html#advanced-config-file) RabbitMQ config for this.

The `ra` application (which is the Raft library that quorum
queues use) has it's own configuration.
See [https://github.com/rabbitmq/ra#configuration](https://github.com/rabbitmq/ra#configuration)
for details.

The `rabbit` application has a couple of quorum queue related configuration items:

* `quorum_cluster_size`

Sets the default quorum queue cluster size (can be over-ridden by the `x-quorum-cluster-size` queue argument at declaration time. Default: 5

* `quorum_commands_soft_limit`

This is a flow control related parameter defining
the maximum number of unconfirmed messages a channel accepts before entering flow.
Default: 256

Example:

```
[
 {rabbit, [{quorum_cluster_size, 7},
           {quorum_commands_soft_limit, 512}]}
]
```


## [Limitations](#limitations)

#### Atom use

The internal implementation of quorum queues converts the queue name
into an Erlang atom. If queues with arbitrary names are continuously
created and deleted it _may_ threaten the long term stability of the
RabbitMQ system (if the size of the atom table reaches the maximum limit,
typically 1,048,576). We do not recommend using quorum queues in this manner
at this point.