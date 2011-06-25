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

There are boot steps that don't enable nor require others to be run. They are used to signal that a group of boot steps have happened as a whole, so the next group can start running. For example we have the external infrastructure step:

    {external_infrastructure,
        [{description,"external infrastructure ready"}]}

As you see it lacks the `requires` and the `enables` properties. But since many steps declare that their enable it, then `external_infrastructure` won't be run until those steps are run. Also many steps that come after in the chain require `external_infrastructure` to have run before, so they won't be started either until it had been processed.

But the story doesn't ends here. RabbitMQ can be extended with plugins that add new exchanges or authentication methods to the broker. Taking the exchanges as an example, each exchange type is registered into the broker via the `rabbit_registry` module, that means the `rabbit_registry` has to be started __before__ we can register a plugin. If we want to add new exchanges we don't have to worry about when they will be started by the broker, neither we have to care of managing the functional dependencies of our exchange. We just add a `-rabbit_boot_step` declaration to our exchange module where we say that our custom exchange depends on `rabbit_registry` et voilà, the exchange will be ready to use.

There's more to it too. In the same way your custom exchange can add their own boot steps to hook up into the server boot process, you can add extra boot steps that perform some stuff in between of RabbitMQ's predefined boot steps. Keep in mind that you have to know what you are doing if you start to customize RabbitMQ's booting process.

Now, if you have been doing some Erlang programming you may be wondering at this point how does this even work at all. Erlang modules can have attributes, like the list of exported functions, or the declaration of which behaviour is implemented by the module, but there's no where a mention in the Erlang documentation about `boot_steps` and of course there's nothing about `-rabbit_boot_steps`. How do they work then?

When the broker is starting it builds a list of all the modules defined in the loaded applications. Once the list of modules is built it's scanned for attributes called `rabbit_boot_steps`. If there are any, they are added to a new list. This list is further processed and converted into an acyclic digraph which is used to maintain an order between the boot steps, that is the boot steps are ordered according to their dependencies. Here is where I think relies the elegance of this solution: add declarations to modules in the form of custom module attributes, scan for them and do something smart with the information. This speaks about the flexibility of Erlang as a language.

### pre_boot ###

### external_infrastructure ###

### kernel_ready ###

### core_initialized ###

### routing_ready ###

### log_relay ###

### networking ###

### direct_client ###

### notify_cluster ###