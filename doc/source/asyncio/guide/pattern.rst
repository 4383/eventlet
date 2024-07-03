.. _asyncio-facade-pattern:

AsyncIO Facade Pattern
======================

This section present a code pattern that you can use to migrate your
application from Eventlet to AsyncIO.

Many mainstream libraries offer synchronous and asynchronous behaviours.
`SQLAlchemy <https://www.sqlalchemy.org/>`_ and
`Edgdb <https://docs.edgedb.com/>`_ to not name them, as libraries,
provide this kind of behavior. If you are a developer of a library you surely
used Eventlet to offer this kind of behavior. Therefore you may also want
provide these synchronous or asynchronous contexts. 

The goal is to allow to implement blocking and non-blocking behaviors in all
our libraries by only relying on a single design under the hood. one code
implementation to rule them all.

This pattern is inspired from the `edgedb-python
<https://github.com/edgedb/edgedb-python>`_ library.
Speaking examples of edgedb-python `usages are available here
<https://www.edgedb.com/docs/clients/python/usage#edgedb-python-examples>`_.

This pattern is based `the facade design pattern
<https://en.wikipedia.org/wiki/Facade_pattern>`_.

The goal of this pattern is to replace the mechanisms offered by the monkey
patching but by enterly relying on AsyncIO and so on a native Python
standard library.

By using this pattern, everything would become based on AsyncIO and
native Python coroutines. The facades will allow doing blocking IO calls by
relying on an uniform async code base. 

Lets give an example.

The Initial Context
-------------------

Imagine you are the developer of a third party library
that provide features to monitor `Openstack <https://www.openstack.org/>`_ VMs
through requesting the rest API of a running Openstack instance. Lets name
that hypothetical library `myStack```.

The following code is an example of someone using your library::

    # End user code using your myStack library
    import mystack

    client = mystack.Client()
    vm = client.request("https://rest.api.xyz/vm/12345")
  
The ``client`` module of the ``myStack`` library is based on
`requests <https://docs.python-requests.org/en/latest/index.html>`_.
It allow you requesting the Openstack rest API.

Here is an example of the hypothetical internal implementation of
this ``client`` module::
   
    # mystack.client
    import requests


    class Client():
        headers = {
            'User-Agent': 'sync_client',
        }

        def request(self, url):
            r = requests.get(url)
            return r.json()

The end user could be eager to execute this HTTP call in an asynchronous way
to avoid blocking his program on a network call. The end user may use your
lib to monitor numerous instances and so he would want to avoid loosing time.

The end user may use Eventlet, to monkey patch all is stack at runtime.
Then the request call would become async::

    import eventlet
    eventlet.monkey_patch()
    from mystack import Client # non-blocking now
    client = Client()
    client.request("https://rest.api.xyz/vm/12345")

But ``mystack`` could be also used the same way on services that do not
rely on Eventlet, where blocking call are not a problem, or where the
service manage async things internally by using threading or process,
who know...

The ``mystack`` lib is currently agnostic to network programming model used
by its end users - the environment defined by the service importing
``mystack``.

The Pattern
-----------

**The proposed pattern may be used either for migrating libraries or standard
applications.** The semantic will remains the same in both cases.

.. warning::

    Using the following pattern require to use the :mod:`Asyncio Hub
    <eventlet.hubs.asyncio>`

    :ref:`understanding_hubs`

Here is a refactor of the previous ``mystack`` code which is now based on the
AsyncIO facade pattern::

    # mystack.client
    import asyncio
    import aiohttp


    class AsyncClient():
        headers = {
            'User-Agent': 'async_client',
        }

        async def request(self, url):
            async with aiohttp.ClientSession(headers=self.headers) as session:
                async with session.get(url) as response:
                    res = await response.json()
                    return res


    class Client(AsyncClient):
        def __init__(self):
            self.headers = {
                'User-Agent': 'sync_client',
            }

        def _iter_coroutine(self, coro):
            loop = None
            try:
                loop = asyncio.get_running_loop()
                except RuntimeError:
                    loop = asyncio.new_event_loop()
                    return loop.run_until_complete(coro)

        def request(self, url):
            return self._iter_coroutine(super().request(url))

This module now allow to use either a synchronous or an asynchronous client.
Behind the scene the synchronous client will call the asynchronous client.

You can observe that we switched the `User-Agent` header depending on the
client used by the end users. It will allow you to observe the difference if
you decide to run this example.

If people want to use the asynchronous client, then their code will be
dotted with `async` and `await` keywords, making their asynchronous calls
explicit, not like with Eventlet.

If people want to use the synchronous client, then their code will remains
the same. From the end user point of view it is transparent.
   
Migrated end users would call the ``mystack`` `Client.request` method like to
this::

    client = AsyncClient()
    asyncio.run(client.request(url))

Or again like this::
   
    class ClassXYZ():

        def __init__(self):
            self.client = AsyncClient()

        async def feature_xyz(self):
            await self.client.request(url)

Thus making explicit all the asynchronous calls in the end user code.

`Here is a living PoC
<https://github.com/4383/snippets/blob/main/python/facade/facade.py>`_
more or less similar to this previous example, do not hesitate to have a look.

Readers may also be interested by the :ref:`awaitlet_alternative`.

Handle Shutdown Properly
------------------------

Applications using this pattern would also have to be adapted to starting up
and shutting down gracefully. By example, service would have to provide a
signal handler, to cancel tasks properly and control the life cycle of the
event loop.

Here is an implementation example::

    import asyncio
    from signal import SIGINT, SIGTERM


    async def main():
        loop = asyncio.get_running_loop()
        for sig in (SIGTERM, SIGINT):
            # Because asyncio.run() takes control of the event loop startup,
            # our first opportunity to change signal handling behavior will
            # be in the main() function.
            loop.add_signal_handler(sig, handler, sig)
        try:
            while True:
                print('<Your app is running>')
                await asyncio.sleep(1)
        except asyncio.CancelledError:
            for i in range(3):
                print('<Your app is shutting down...>')
                await asyncio.sleep(1)


    def handler(sig):
        loop = asyncio.get_running_loop()
        # Inside the signal handler, we can’t stop the loop as in previous
        # examples, because we’ll get warnings about how the loop was stopped
        # before the task created for main() was completed. Instead, we can
        # initiate task cancellation here, which will ultimately result in the
        # main() task exiting; when that happens, the cleanup handling inside
        # asyncio.run() will take over.
        for task in asyncio.all_tasks(loop=loop):
            task.cancel()

        print(f'Got signal: {sig!s}, shutting down.')
        loop.remove_signal_handler(SIGTERM)
        loop.add_signal_handler(SIGINT, lambda: None)


    if __name__ == '__main__':
        asyncio.run(main())

In the previous example, we launch our asynchronous code and inside it we
attach the signals that we want to handle to cancel our tasks properly.

AsyncIO Best Practices
----------------------

Code based on AsyncIO should not call functions based on synchronous I/O,
like `the built-in function open() <https://docs.python.org/3/library/functions.html#open>`_.

You should be careful if you use the logging module from the standard library,
`it can block the event loop <https://docs.python.org/3/library/asyncio-dev.html#logging>`_.

You may find useful to use `Python context variables
<https://docs.python.org/3/library/contextvars.html#asyncio-support>`_
which allow to have data that are local to a specific task.

Many `third-party libraries <https://docs.python.org/3/library/asyncio-dev.html#logging>`_
are designed to support asynchronous code that may help you during your
transition.
