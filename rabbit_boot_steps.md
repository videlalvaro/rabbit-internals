# RabbitMQ Internals #

## RabbitMQ Boot Process ##

When RabbitMQ starts running it goes through a series of what are called __boot steps__ that take care of initializing all the core components of the broker in a specific order. The whole boot step concept is –as far as I can tell– something unique to RabbitMQ. The idea behind it is that each subsystem that forms part of RabbitMQ as a whole will declare on which other systems it depends on and if it's successfully started, which other systems it will enable. For example, there's no point in accepting client connections if the layer that routes messages to queues is not enabled.

The implementation is very elegant, it relies on adding custom attributes to erlang modules that declare how to start a boot step, in which boot steps it depends on and which boot steps it will enable, here's an example:

    -rabbit_boot_step({recovery,
                       [{description, "exchange, queue and binding recovery"},
                        {mfa,         {rabbit, recover, []}},
                        {requires,    empty_db_check},
                        {enables,     routing_ready}]}).

Here the step name is `recovery`, which as the description says it manages _"exchange, queue and binding recovery"_. It requires the `empty_db_check` boot step and enables the `routing_ready` boot step. As you can see, there's a `mfa` argument which specifies the `Module` `Function` and `Arguments` to call in order to start this boot step.

So far this seems very simple and even can make us doubt of the usefulness of such approach: why is there a need for boot steps at all? Why there isn't just a call to functions one after the other and that's it? Well, is not that simple.

The boot steps can be separated into groups. A group of boot steps will enabled certain other group. For example `routing_ready` is actually enabled by many others boot steps, not just `recovery`. One of such steps is the `empty_db_check` that ensures that the `Mnesia` database has the default data inside, like the default `guest` user for example. Also we can see that the `recovery` boot step also depends on `empty_db_check` so this logic takes care of running them in the right order that will satisfy the interdependencies they have.

But the story doesn't ends here. RabbitMQ can be extended with plugins that add new exchanges or authentication methods to the broker. Taking the exchanges as an example, each exchange type is registered into the broker via the `rabbit_registry` module, that means the `rabbit_registry` has to be started __before__ we can register a plugin. If we want to add new exchanges we don't have to worry about when they will be started by the broker, neither we have to care of managing the functional dependencies of our exchange. We just add a `-rabbit_boot_step` declaration to our exchange module where we say that our custom exchange depends on `rabbit_registry` et voilà, the exchange will be ready to use.

Now, if you have been doing some Erlang programming you may be wondering at this point how does this even work at all. Erlang modules can have attributes, like the list of exported functions, or the declaration of which behaviour is implemented by the module, but there's no where a mention in the Erlang documentation about `boot_steps` and of course there's nothing about `-rabbit_boot_steps`. How do they work then?

When the broker is starting it builds a list of all the modules defined in the loaded applications. Once the list of modules is built it's scanned for attributes called `rabbit_boot_steps`. If there are any, they are added to a new list. This list is further processed and converted into an acyclic digraph which is used to maintain an order between the boot steps, that is the boot steps are ordered according to their dependencies. Here is where I think relies the elegance of this solution: add declarations to modules in the form of custom module attributes, scan for them and do something smart with the information. This speaks about the flexibility of Erlang as a language.

### pre_boot ###

### worker_pool ###

### external_infrastructure ###

### kernel_ready ###

### core_initialized ###

### routing_ready ###

### log_relay ###

### networking ###

### direct_client ###

### notify_cluster ###

## The parts of RabbitMQ ##

Supervisors:

rabbit_sup

Global supervisor started after the `rabbit` app has started.

-- rabbit_node_monitor_sup
---- rabbit_node_monitor
Notifies other nodes in the clusters about its own node presence.
Takes cares of the situation when other node dies.

-- msg_store_transient
---- rabbit_msg_store

-- msg_store_persistent
---- rabbit_msg_store

Both handle the message storage.

-- rabbit_tcp_client_sup
See drawing on paper.
Started during the networking boot step. See rabbit_networking:start/0

-- rabbit_direct_client_sup
Started during the direct_client boot step. See rabbit_direct:boot/0


-- tcp_listener_sup <--- not registered. This supervisor structure is repeated for each network interface on which RabbitMQ listens too
---- tcp_acceptor_sup_IP:PORT
------ tcp_acceptor
---- tcp_listener

-- rabbit_amqqueue_sup
---- rabbit_amqqueue_process  <- one child per queue?

-- rabbit_mirror_queue_slave_sup
----- rabbit_mirror_queue_slave <-- one child per queue? Used for HA queues.

-- rabbit_guid_sup
---- rabbit_guid
Globally Unique Identifier server.

-- rabbit_memory_monitor_sup
---- rabbit_memory_monitor
Monitors queues memory usage.

-- delegate_sup <-- instance of rabbit_sup.erl
---- delegate_x <-- x goes 0-15 according to app config argument: delegate_count. Used to parallelize calls to processes. For example when routing messages, the delegates take care of sending the messages to each of the queues that ought to receive the message.

-- rabbit_registry
Keeps a registry of plugins and their modules. For example it maps authentication mechanisms to modules with the actual implementation. The same thing is done from exchange type to exchange type implementation.

-- rabit_log_sup
---- rabbit_log
Delegates logging calls to the native error_logger module.

-- vm_memory_monitor_sup
---- vm_memory_monitor
Monitors Erlang's memory usage to prevent it from crashing or swapping.

-- rabbit_event_sup
---- rabbit_event
Handles event notification for statistics collection. For example when a new channel is created, then a notification like `rabbit_event:notify(channel_created, infos(?CREATION_EVENT_KEYS, State))` is fired.

-- file_handle_cache_sup <-- instance of rabbit_sup.erl
---- file_handle_cache
Manages file handles to synchronize reads and writes to them. See `file_handle_cache.erl` for an in depth explanation of its purpose.

-- worker_pool_sup <-- instance of rabbit_sup.erl
---- worker_pool
---- worker_pool_worker
The worker pool process manages a pool of up to `N` number of workers where `N` is the return of `erlang:system_info(schedulers)`. Used to parallelize function calls.