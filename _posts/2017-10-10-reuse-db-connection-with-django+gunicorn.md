

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



How django manage connections?

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


What does gunicorn do if launched with "-k gevent"?

You may need read source code of gunicorn for this question. But, for this scenario, one most import thing gunicorn does  is "patching codes" for asynchronic I/O operation purpose. 

Lets come to code directly. 

	1.	select gevent worker class
	2.	process of start work
	3.	detail of patch_all


Then, how problem happen?

The key point is `local()` function. Before and after patching



How to reslove this?


More thinkings:


