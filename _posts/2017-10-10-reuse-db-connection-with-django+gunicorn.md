---
categories: [python-web]
tags: [django, gunicorn]
---

Problem:
	django + gunicorn

Key Pionts of Configuration:
	
	gunicorn:	async worker. Be launched with "-k gevent"

	django:	DB settings. set "CONN_MAX_AGE" non-zero, say, 60.
			Use MySQLdb as db interface


CONN_MAX_AGE

Default: 0

The lifetime of a database connection, in seconds. Use 0 to close database connections at the end of each request — Django’s historical behavior — and None for unlimited persistent connections.



https://docs.djangoproject.com/en/1.10/ref/databases/#persistent-connections



## How does django manage connections?

https://docs.djangoproject.com/en/1.10/ref/databases/#connection-management

>Django opens a connection to the database when it first makes a database query. It keeps this connection open and reuses it in subsequent requests. Django closes the connection once it exceeds the maximum age defined by CONN_MAX_AGE or when it isn’t usable any longer.
>
>At the beginning of each request, Django closes the connection if it has reached its maximum age....
>
>At the end of each request, Django closes the connection if it has reached its maximum age or if it is in an unrecoverable error state. If any database errors have occurred while processing the requests, Django checks whether the connection still works, and closes it if it doesn’t....

Accroding to django documents, connections are managed:

	1.	By default, each thread maintains its own connection. The connection is left open to be reused, unless the close conditions triggered.
	2.	Close connection at beginning of each request, with the following conditions:
		a.	CONN_MAX_AGE is 0.
		b.	Connection reaches its maximum age.
	3.	Close connection at end of each reqeust, with the following conditions:
		a.	CONN_MAX_AGE is 0.
		b.	Connection reaches its maximum age.
		c.	Connection in an unrecoverable error.
	4.	If a connection is set to be sharable, then it can shared by multi threads. But if you are using MySQLdb as the db interface, it maybe a bad idea to do this.
	
Lets see the code for this.

	1.	code for ConnectionHandler, CONN_MAX_AGE and share
	2.	code for singnal
	3.	code for close(), checking CONN_MAX_AGE


## What does gunicorn do if launched with "-k gevent"?

You may need read source code of gunicorn for this question. But, for this scenario, one most important work is that "threads are patched" by gunicorn for asynchronic I/O operation purpose. 

Lets see the code. 

	1.	select gevent worker class
	2.	process of start work
	3.	detail of patch_all

The main master loop of gunicorn, which is invoked by `WSGIApplication.run` method.
<pre>
    def run(self):
        "Main master loop."
        self.start()
        util._setproctitle("master [%s]" % self.proc_name)

        try:
            <b>self.manage_workers()</b>
            while True:
                self.maybe_promote_master()
				...
</pre>
And, in calling to method `self.manage_workes`, the prefored childrens will do some prepare work for handling requests.
```
    def manage_workers(self):
        """\
        Maintain the number of workers by spawning or killing
        as required.
        """
        if len(self.WORKERS.keys()) < self.num_workers:
            *self.spawn_workers()*
			...

    def spawn_worker(self):
        self.worker_age += 1
        worker = self.worker_class(self.worker_age, self.pid, self.LISTENERS,
                                   self.app, self.timeout / 2.0,
                                   self.cfg, self.log)
        self.cfg.pre_fork(self, worker)
        pid = os.fork()
        if pid != 0:
            self.WORKERS[pid] = worker
            return pid

        # Process Child
        worker_pid = os.getpid()
        try:
            util._setproctitle("worker [%s]" % self.proc_name)
            self.log.info("Booting worker with pid: %s", worker_pid)
            self.cfg.post_fork(self, worker)
            *worker.init_process()*
            sys.exit(0)
			...
```
As we can see, the forked children processes will do several work, but most important here, they invoke method `init_process` of `GeventWorker`, since `-k gevent` is picked up. And `init_process` invokes `patch` to patch the codes.

```
        def init_process(self):
            # monkey patch here
            self.patch()
			...

    def patch(self):
        from gevent import monkey
        monkey.noisy = False

        # if the new version is used make sure to patch subprocess
        if gevent.version_info[0] == 0:
            monkey.patch_all()
        else:
            monkey.patch_all(subprocess=True)		
```
And `patch_all` method patches codes related to `threads and threading`.
```
	...
    if thread:
        patch_thread(Event=Event)
	...

	...
    patch_module('thread')
    if threading:
        threading = patch_module('threading')	
	...
	...
    if _threading_local:
        _threading_local = __import__('_threading_local')
        from gevent.local import local
        patch_item(_threading_local, 'local', local)
	...
```
As we can see, `local` from `thread` and `_threading_local` are replaced by that from `gevent.local`.

## Problem is, whats the difference between `local` from `thread` (or `_threading_local`) and from `gevent`?

Then, how problem happen?

The key point is `local()` function. Before and after patching



How to reslove this?


More thinkings:


