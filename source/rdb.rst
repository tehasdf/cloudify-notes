Using the debugger with cloudify
==============================

While some parts of cloudify do run locally (eg. the CLI, or the system tests
framework) and can be run under a regular debugger (eg.
`pudb <https://pypi.python.org/pypi/pudb>`_), most of the interesting parts
run in celery tasks, so a regular debugger won't work with that.


Remote debugger in celery
-------------------------

With celery, use rdb, eg. use a text editor to

.. code-block:: python

    from celery.contrib import rdb
    rdb.set_trace()

Then it will start listening on port 6899 (by default), or if that is already
in use, 6900, 6901, ...

Connect to the debugger using telnet or nc (on centos, you need to install
them using eg. `yum install telnet` or `yum install nc`), like
`telnet 127.0.0.1 6900`.

Sometimes it might be hard to guess which port it's using; you can simply guess
or look at the open ports and make a more informed guess :)
The following command might be useful:

.. code-block:: bash

    $ netstat -tulpn | grep python

Shows ports that processes named "python" are listening on. (on a cloudify
manager usually it'll be just the REST service, and your debuggers)


.. note::

    Remember that after adding `.set_trace()`, you need to restart the process
    running that code (usually it'll be celery - or gunicorn if you're
    working on the REST service).

.. note::

    `cloudify-plugins-common` has a `celery` module, so to be able to import the
    original `celery` inside it (notably, inside `dispatch.py`), you need to
    disallow implicit relative imports (otherwise you'll get an ImportError).
    To do it, add the following as the first line of the module you're editing:

    .. code-block:: python

        from __future__ import absolute_import


Also read the `Celery RDB docs <http://docs.celeryproject.org/en/latest/tutorials/debugging.html>`_.


Where to put set_trace?
-----------------------

If you'd like to put a `set_trace` call inside cloudify code that will be
used by the management worker, edit files inside `/opt/mgmtworker/env/lib/python2.7/site-packages`,
eg. `/opt/mgmtworker/env/lib/python2.7/site-packages/cloudify/dispatch.py`

Restarting celery
-----------------

After adding a `set_trace`, you need to restart celery so it loads the new code.
To do it, best to simply do `systemctl restart cloudify-mgmtworker`.
Alternatively, you can send a HUP signal to the main celery process, eg. like this

.. code-block:: bash

    # first, find the PID
    $ pgrep -f celery
    # now, SIGHUP it
    $ kill -HUP 1234

Anyway, it is a good idea to tail the mgmtworker logs to verify that it did
restart successfully (`/var/log/mgmtworker/cloudify-mgmtworker.log`)

Celery on agent machines
------------------------

Celery is also used on agent machines - the virtualenv is usually located
in `/home/<agent_user>/<node_name>/env/`, eg `/home/centos/vm_a1b2c3/env/lib/python2.7/site-packages/cloudify/dispatch.py`.

Restarting celery there depends on the process management method, but with the
most commonly used initd, you would do

.. code-block:: bash

    $ /etc/init.d/celery_<node_name> restart


PDB primer
----------

rdb is simply pdb using a network socket; all the regular pdb commands can
still be used with it. See the `pdb docs <https://docs.python.org/2/library/pdb.html#debugger-commands>`_
and `this talk <https://www.youtube.com/watch?v=P0pIW5tJrRM>`_ to learn how to use pdb.


Using the debug signal
----------------------

You can also use the debug signal as explained in rdb docs, however this allows
for less granularity than placing `set_trace` calls manually. If you'd like to
use the signal, you need to add the `CELERY_RDBSIG=1` environment variable to
`/etc/sysconfig/mgmtworker` and restart celery.
