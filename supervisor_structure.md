## RabbitMQ Supervisor Structure ##

rabbit_sup

Global supervisor started after the `rabbit` app has started.
```
|
--- rabbit_node_monitor_sup
|    |
|    --- rabbit_node_monitor
|		Notifies other nodes in the clusters about its own node presence.
|		Takes cares of the situation when other node dies.
|
--- msg_store_transient
|	|
|	--- rabbit_msg_store
|
--- msg_store_persistent
|	|
|	--- rabbit_msg_store
|		Both handle the message storage.
|
--- rabbit_tcp_client_sup
|	See drawing on paper.
|	Started during the networking boot step. See rabbit_networking:start/0
|
--- rabbit_direct_client_sup
|	Started during the direct_client boot step. See rabbit_direct:boot/0
|
|
--- tcp_listener_sup <--- not registered. This supervisor structure is repeated for each network interface on which RabbitMQ listens too
|	|
|	--- tcp_acceptor_sup_IP:PORT
|	|	 |
|	|	 --- tcp_acceptor
|	|
|	--- tcp_listener
|
--- rabbit_amqqueue_sup
|	|
|	--- rabbit_amqqueue_process  <- one child per queue?
|
--- rabbit_mirror_queue_slave_sup
|	|
|	--- rabbit_mirror_queue_slave <-- one child per queue? Used for HA queues.
|
--- rabbit_guid_sup
|	|
|	--- rabbit_guid		
|		Globally Unique Identifier server.
|
--- rabbit_memory_monitor_sup
|	|
|	--- rabbit_memory_monitor
|       Monitors queues memory usage.
|
--- delegate_sup <-- instance of rabbit_sup.erl
|	|
|	--- delegate_x <-- x goes 0-15 according to app config argument: delegate_count. Used to parallelize calls to processes. For example when routing messages, the delegates take care of sending the messages to each of the queues that ought to receive the message.
|
--- rabbit_registry
|	Keeps a registry of plugins and their modules. For example it maps authentication mechanisms to modules with the actual implementation. The same thing is done from exchange type to exchange type implementation.
|
--- rabit_log_sup
|	|
|	--- rabbit_log
|		 Delegates logging calls to the native error_logger module.
|
--- vm_memory_monitor_sup
|	|
|	--- vm_memory_monitor
|		 Monitors Erlang's memory usage to prevent it from crashing or swapping.
|
--- rabbit_event_sup
|	|
|	--- rabbit_event
|		 Handles event notification for statistics collection. For example when a new channel is created, then a notification like `rabbit_event:notify(channel_created, infos(?CREATION_EVENT_KEYS, State))` is fired.
|
--- file_handle_cache_sup <-- instance of rabbit_sup.erl
|	|
|	--- file_handle_cache
|		Manages file handles to synchronize reads and writes to them. See `file_handle_cache.erl` for an in depth explanation of its purpose.
|
--- worker_pool_sup <-- instance of rabbit_sup.erl
	|
	--- worker_pool
	|
	--- worker_pool_worker
	The worker pool process manages a pool of up to `N` number of workers where `N` is the return of `erlang:system_info(schedulers)`. Used to parallelize function calls.
```
