.. _getting-started:

Getting Started with Humus AMQP Module, RabbitMQ and Zend Framework 2
=====================================================================

About this guide
----------------

This guide is a quick tutorial that helps you to get started with
RabbitMQ and `HumusAmqp <https://www.github.com/prolic/HumusAmqp>`_.  It should
take about 20 minutes to read and study the provided code
examples. This guide covers:

 * Installing RabbitMQ, a mature popular messaging broker server.
 * Installing HumusAmqp via `Composer <http://www.getcomposer.org/>`_.
 * Producing and consuming messages from cli.
 * Running a JSON-RPC server and client.


Installing RabbitMQ
-------------------

The `RabbitMQ site
<http://rabbitmq.com>`_ has a good `installation guide
<http://rabbitmq.com/install.html>`_ that addresses many operating systems.
On Mac OS X, the fastest way to install RabbitMQ is with `Homebrew
<http://mxcl.github.com/homebrew/>`_:

.. code-block:: bash

    $ brew install rabbitmq

then run it:

.. code-block:: bash

    $ rabbitmq-server

On Debian and Ubuntu, you can either `download the RabbitMQ .deb
package
<http://rabbitmq.com/server.html>`_ and install it with
`dpkg
<http://www.debian.org/doc/FAQ/ch-pkgtools.en.html>`_ or make use
of the `apt repository
<http://rabbitmq.com/debian.html#apt_>`_ that
the RabbitMQ team provides.

For RPM-based distributions like RedHat or CentOS, the RabbitMQ team
provides an `RPM package
<http://www.rabbitmq.com/install.html#rpm>`_.


Installing HumusAmqp
--------------------

a) You can use composer to install HumusAmqp

.. code-block:: bash

    $ php composer.phar require prolic/humus-amqp dev-master

b) Select a driver to use, currently only php-amqplib and php-amqp (PHP extension) are supported.

Configure your container to return the DriverFactory for the Driver class. If your container supports configuration by
array like Zend\ServiceManager f.e., this should look similar to this.

.. code-block:: php

    return [
        'dependencies' => [
            'factories' => [
                Driver::class => Container\DriverFactory::class,
            ]
        ]
    ];

c) Configure your application to use the desired driver.

.. code-block:: php

    return [
        'humus' => [
            'amqp' => [
                'driver' => 'amqp-extension',
            ]
        ]
    ];

d) Notes about drivers:

1) The PHP Extension (php-amqp) is the fastest one, we strongly recommend using 1.7.0 or building yourself from master to
be able to use all features.

There is currently a bug in PhpAmqpLib, see: https://github.com/php-amqplib/php-amqplib/pull/399
As long as this is not merged and release, you have to manually apply the patch, sorry!

You can do this from the command-line with:

`sed -i '/$message = $this->get_and_unset_message($delivery_tag);/a \ \ \ \ \ \ \ \ \ \ \ \ $message->delivery_info["delivery_tag"] = $delivery_tag;' vendor/php-amqplib/php-amqplib/PhpAmqpLib/Channel/AMQPChannel.php`

2) When using php-amqplib as driver, it's worth point out, that a StreamConnection (same goes for SSLConnection) does not
have the possibility to timeout. If you want to let the consumer timeout, when no more messages are received, you should
use the SocketConnection instead (assuming you don't need an SSL connection).

Sample-Configuration
--------------------

A sample configuration might look like this, more details an explanation will be in the coming chapters.

.. code-block:: php

    return [
        'dependencies' => [
            'factories' => [
                Driver::class => Container\DriverFactory::class,
                'default-amqp-connection' => [Container\ConnectionFactory::class, 'default'],
                'demo-producer' => [Container\ProducerFactory::class, 'demo-producer'],
                'topic-producer' => [Container\ProducerFactory::class, 'topic-producer'],
                'demo-consumer' => [Container\CallbackConsumerFactory::class, 'demo-consumer'],
                'topic-consumer-error' => [Container\CallbackConsumerFactory::class, 'topic-consumer-error'],
                'demo-rpc-server' => [Container\JsonRpcServerFactory::class, 'demo-rpc-server'],
                'demo-rpc-server2' => [Container\JsonRpcServerFactory::class, 'demo-rpc-server2'],
                'demo-rpc-client' => [Container\JsonRpcClientFactory::class, 'demo-rpc-client'],
                'my_callback' => $my_callback_factory,
            ],
        ],
        'humus' => [
            'amqp' => [
                'driver' => 'php-amqplib',
                'exchange' => [
                    'demo' => [
                        'name' => 'demo',
                        'type' => 'direct',
                        'connection' => 'default-amqp-connection',
                    ],
                    'demo.error' => [
                        'name' => 'demo.error',
                        'type' => 'direct',
                        'connection' => 'default-amqp-connection',
                    ],
                    'topic-exchange' => [
                        'name' => 'topic-exchange',
                        'type' => 'topic',
                        'connection' => 'default-amqp-connection',
                    ],
                    'demo-rpc-client' => [
                        'name' => 'demo-rpc-client',
                        'type' => 'direct',
                        'connection' => 'default-amqp-connection',
                    ],
                    'demo-rpc-server' => [
                        'name' => 'demo-rpc-server',
                        'type' => 'direct',
                        'connection' => 'default-amqp-connection',
                    ],
                    'demo-rpc-server2' => [
                        'name' => 'demo-rpc-server2',
                        'type' => 'direct',
                        'connection' => 'default-amqp-connection',
                    ],
                ],
                'queue' => [
                    'foo' => [
                        'name' => 'foo',
                        'exchange' => 'demo', // must be defined as exchange before
                        'arguments' => [
                            'x-dead-letter-exchange' => 'demo.error', // must be defined as exchange before
                        ],
                        'connection' => 'default-amqp-connection',
                    ],
                    'demo-rpc-client' => [
                        'name' => '',
                        'exchange' => 'demo-rpc-client',
                        'connection' => 'default-amqp-connection',
                    ],
                    'demo-rpc-server' => [
                        'name' => 'demo-rpc-server',
                        'exchange' => 'demo-rpc-server',
                        'connection' => 'default-amqp-connection',
                    ],
                    'demo-rpc-server2' => [
                        'name' => 'demo-rpc-server2',
                        'exchange' => 'demo-rpc-server2',
                        'connection' => 'default-amqp-connection',
                    ],
                    'info-queue' => [
                        'name' => 'info-queue',
                        'exchange' => 'topic-exchange',
                        'routingKeys' => [
                            '#.err',
                        ],
                        'connection' => 'default-amqp-connection',
                    ],
                ],
                'connection' => [
                    'default' => [
                        'type' => 'socket',
                        'host' => 'localhost',
                        'port' => 5672,
                        'login' => 'guest',
                        'password' => 'guest',
                        'vhost' => '/',
                        'persistent' => true,
                        'read_timeout' => 3, //sec, float allowed
                        'write_timeout' => 1, //sec, float allowed
                    ],
                ],
                'producer' => [
                    'demo-producer' => [
                        'type' => 'plain',
                        'exchange' => 'demo',
                        'qos' => [
                            'prefetch_size' => 0,
                            'prefetch_count' => 10,
                        ],
                        'auto_setup_fabric' => true,
                    ],
                    'topic-producer' => [
                        'exchange' => 'topic-exchange',
                        'auto_setup_fabric' => true,
                    ],
                ],
                'callback_consumer' => [
                    'demo-consumer' => [
                        'queue' => 'foo',
                        'callback' => 'echo',
                        'idle_timeout' => 10,
                        'delivery_callback' => 'my_callback',
                    ],
                    'topic-consumer-error' => [
                        'queues' => [
                            'info-queue',
                        ],
                        'qos' => [
                            'prefetch_count' => 100,
                        ],
                        'auto_setup_fabric' => true,
                        'callback' => 'echo',
                        'logger' => 'consumer-logger',
                    ],
                ],
                'json_rpc_server' => [
                    'demo-rpc-server' => [
                        'callback' => 'poweroftwo',
                        'queue' => 'demo-rpc-server',
                        'auto_setup_fabric' => true,
                    ],
                    'demo-rpc-server2' => [
                        'callback' => 'randomint',
                        'queue' => 'demo-rpc-server2',
                        'auto_setup_fabric' => true,
                    ],
                ],
                'json_rpc_client' => [
                    'demo-rpc-client' => [
                        'queue' => 'demo-rpc-client',
                        'auto_setup_fabric' => true,
                    ],
                ],
            ],
        ],
    ];


Running from CLI
----------------

In order to run cli commands, you need to setup your connection, exchange and queue configuration.
See here on how to do this:

You can run cli commands like this:

.. code-block:: bash

    $ ./vendor/bin/humus-amqp

To start a consumer:

.. code-block:: bash

    $ ./vendor/bin/humus-amqp consumer -n myconsumer -a 100

This will start the myconsumer and consume 100 messages until if stops.


To start a JSON-RPC server

.. code-block:: bash

    $ ./vendor/bin/humus-amqp json_rpc_server -n myserver -a 100

This will start the myserver and consume 100 messages until if stops.

What to read next
-----------------

Documentation is organized as a number of :ref:`guides <guides>`, covering all
kinds of topics including use cases for various exchange types,
fault-tolerant message processing with acknowledgements and error
handling.

We recommend that you read the following guides next, if possible, in this order:

 * `AMQP 0.9.1 Model Explained <http://www.rabbitmq.com/tutorials/amqp-concepts.html>`_. A simple 2 page long introduction to the AMQP Model concepts and features. Understanding the AMQP 0.9.1 Model
   will make a lot of other documentation, both for Bunny and RabbitMQ itself, easier to follow. With this guide, you don't have to waste hours of time reading the whole specification.
 * :ref:`connecting`. This guide explains how to connect to an RabbitMQ and how to integrate Bunny into standalone and Web applications.
 * :ref:`queues`. This guide focuses on features that consumer applications use heavily.
 * :ref:`exchanges`. This guide focuses on features that producer applications use heavily.
 * :ref:`error_handling`. This guide explains how to handle protocol errors, network failures and other things that may go wrong in real world projects.


Tell Us What You Think!
-----------------------

Please take a moment to tell us what you think about this guide: `Send an e-mail <saschaprolic@googlemail.com>`_,
say hello in the `HumusAmqp gitter <https://gitter.im/prolic/HumusAmqp>`_ chat.
or raise an issue on `Github <https://www.github.com/prolic/HumusAmqp/issues>`_.

Let us know what was unclear or what has not been covered. Maybe you
do not like the guide style or grammar or discover spelling
mistakes. Reader feedback is key to making the documentation better.
