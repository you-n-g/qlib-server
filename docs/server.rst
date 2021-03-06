.. _server:
==========================
``Qlib-Server`` Structure
==========================

.. currentmodule:: qlib_server

The ``Client/Server structure`` of ``Qlib`` is based on WebSocket considering its capability of **bidirectional communication** between client and server in **async** mode.

Server
================

The server is based on `flask<http://flask.pocoo.org/>`_. Flask is a micro-framework for Python and here we use `flask-SocketIO<https://flask-socketio.readthedocs.io>`_ for websocket connection. The server provides the following functions:

Listening to incoming request from client
--------------------------------------------

The clients will propose several types of requests to server. The server will parse the requests, collect the identical requests from different clients, record their session-ids, and submit these parsed tasks to a pipe. ``Qlib`` use `RabbitMQ<https://www.rabbitmq.com>`_ as this pipe. The tasks will be published to a channel `task_queue`.

**RequestListener** is used for this function:

.. autoclass:: qlib_server.request_handler.RequestListener


After receiving these requests, the server will check whether different clients are asking for the same data. If so, to prevent repeated generation of data or repeated generation of cache files, the server will use `redis<https://redis.io/>`_ to maintain the session-ids of those clients. These session-ids will be deleted once this task is finished. To avoid IO conflicts, `redis_lock<https://pypi.org/project/python-redis-lock/>`_ is imported to make sure no tasks in redis will be read and written at the same time.

Responding clients with data
-------------------------------

The server consumes the result from `message_queue` and get the session-ids of the clients requring this result. Then it responds to these clients with results.

**RequestResponder** is used for this method.

.. autoclass:: qlib_server.request_handler.RequestResponder

The two class above is combined as **RequestHandler**, which is responsible for communicating with clients.

.. autoclass:: qlib_server.request_handler.RequestHandler

Accepting tasks from RabbitMQ and processing data
--------------------------------------------------

The server will automatically collect tasks from RabbitMQ and process the relevant data. RabbitMQ provides a mechanism that when the server consumes a task, a callback function is triggered. The data processing procedure is implemented within these callbacks and it currently supports the three types of tasks corresponding to above:

- Calendar
- Instruments
- Features

**DataProcessor** is used for this function.

.. autoclass:: qlib_server.data_processor.DataProcessor

The server will use `qlib.data.Provider` to process the data. RabbitMQ also provides a mechanism that can make sure all tasks is succesfully consumed and completed by consumers. This requires the consumer call `ch.basic_ack(delivery_tag=method.delivery_tag)` after succesfully processing the data. If the task is not **acked**, it will return to the pipe and wait for another consuming.

Once the task is finished, a result *(could be data or uri)* will be published to another channel `message_queue`.
