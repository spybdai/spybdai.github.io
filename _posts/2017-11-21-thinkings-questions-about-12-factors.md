# Thinkings & Questions about 12-factors

### III. Config

Q: "The twelve-factor app stores config in environment variables". How to manage these env variables? For exmaple, if a developer check out a project from repo, how should he set up the evn variables one by one?

A: I think put the variables directly into environment would be troublesome to maintenance. Our approach is, keep all the configurations, expecially the credential ones, into another stand alone project. With different environments (dev, test, stage or live env.), just link corresponding config files to services. So that, we can achive the standard that open the code repo open source without any update.

### V. Build, Release, Run

Q: How to understand the term "**moving parts**" in sentence "Therefore, the run stage should be kept to as few **moving parts** as possible, since problems that prevent an app from running can cause it to break in the middle of the night when no developers are on hand."?

### VI. Processes

Q: "**Twelve-factor processes are stateless and share-nothing**", which type (share-nothing, share-memory, share-disk) of design the following scenarios are of? 

1. Sometime, to reduce the load of database server, there is a data-collecting service dumping the data out of database to files based on time-line, for example, by day. And, web app will fetch data from the dumped data files for analyse purpose rather than database. The data-collecting service and the web app **shares the disk**.
2. There are some asynchronous works in a web app. For example, after a transaction is finished, the web app will push the result to a message queue, and a result-handle servie is fetching message from message queue for analyse purpose. The web app and the result-handle service is **sharing nothing**.
3. 2 processes communicate with each other via FIFO, they are **sharing memory**.

Q: How to understand "Asset packagers like django-assetpackager use the filesystem as a cache for compiled assets. **A twelve-factor app prefers to do this compiling during the build stage.**"?

A:?

### VII. Port Binding

T: There are few statements of which the translation from English to Chinese is not so precise:

1. "**The twelve-factor app is completely self-contained** and does not rely on ***runtime injection*** of a webserver into the execution environment to create a web-facing service. " —> "**12-Factor 应用完全自我加载** 而不***依赖***于任何网络服务器就可以创建一个面向网络的服务。" I think the key word "**runtime injection**" is translated uncorrectly. 
2. "This is typically implemented by using dependency declaration to add a webserver library to the app, such as Tornado for Python, Thin for Ruby, or Jetty for Java and other JVM-based languages. **This happens entirely in *user space*, that is, within the app’s code.**" —> "通常的实现思路是，将网络服务器类库通过 依赖声明 载入应用。例如，Python 的 Tornado, Ruby 的Thin , Java 以及其他基于 JVM 语言的 Jetty。**完全由 *用户端* ，确切的说应该是应用的代码，发起请求。**" I don't know how does this transalation make sense.

T: More thinkings about the statement "This is typically implemented by using dependency declaration to add a webserver library to the app, ... **This happens entirely in *user space*, that is, within the app’s code.**"

And, the main idea here is, by adding the webserver libray to a app,  the code of webserver is just part of the web app, and they are of "**cooperation relationship**" within user space, rather than that the web app is "**hosted**" by the webserver. 

For example, running django apps by runfcgi or gunicorn. The libraries of runfcgi or gunicorn are imported directly into django apps, and therefore become part of djnaog apps themselves. They are actually one same app in runtime.

For another example, if we want to let a django app start with server booting, then, we may use supervisor to host the app. For this situation, there is "**runtime injection** of supervisor into the execution environment of Django app". As we can see, django app dens't import the dependencies of supervisor, and actually they are 2 disperate apps. And off course, supervisor is not a web server, and this is the only difference with the original statement.

### VIII. Concurrency

T: Key points: "Processes in the twelve-factor app take strong cues from [the unix process model for running service daemons](https://adam.herokuapp.com/past/2011/5/9/applying_the_unix_process_model_to_web_apps/). Using this model,"

Q: How to understand this statement "This does not exclude individual processes from handling their own internal multiplexing, via threads inside the runtime VM, or the async/evented model found in tools such as EventMachine, Twisted, or Node.js. But an individual VM can only grow so large (vertical scale), so the application must also be able to span multiple processes running on multiple physical machines."?

A: It means, **if an individual process handle it's internal multiplexing** via threads inside the runtime VM, or other tools which support async/evented model, **that's OK. We considier it comply with the unix process model, as long as** this process also support herizontal scale.

T: "Twelve-factor app processes **should never daemonize or write PID files**. Instead, rely on the operating system’s process manager (such as systemd, a distributed process manager on a cloud platform, or a tool like [Foreman](http://blog.daviddollar.org/2011/05/06/introducing-foreman.html) in development) to manage output streams, respond to crashed processes, and handle user-initiated restarts and shutdowns."

We have been running web service with runfcgi of Django, and indicate the PID file as a parameter of runfcgi. But afterward we replace this with supervisor.

T: Tips about [the unix process model for running service daemons](https://adam.herokuapp.com/past/2011/5/9/applying_the_unix_process_model_to_web_apps/):

1. process types models: web, worker, clock, etc.
2. all process types (web app, jobs, scheduled-jobs) should be managed by process manager, upstart, supervisor, .etc.
3. process types vs processes
4. Related tools: Profile & foreman?

procfile: declare process types

foreman: run the process both in development and deployment environments.

Examples. we once managed a python commands jobs in the following way, (off course, **it's violation to 12-factor**):

```shell
# job_name.sh
if [ `ps -ef | grep job_name | grep -v grep | grep -v "$0" -c` -lt 1 ]; then
python manage.py job_name
fi
```

and then, put the into the crontab:

```shell
# crontab
* * * * * root cd /path/to/job_name.sh && sh job_name.sh
```

and, the django job itself maybe like this:

```python
# job_name.py
class Command(BaseCommand):
    def handle(self, *args, **kwargs):
        while True:
            do_something
```

And, with the above discussion, we know, a better practice is that:

**drop crontab and the bash script, and instead, use superviosr to management the job directly. Meanwhile, we can also let the job more rubost with shutdown-gracefully, this is also recommended by [IX. Disposability](https://12factor.net/disposability)**:

```
; supervisor config for job_name
[program: job_name]
directory = /path/to/project
command = python management job_name
autostart = True
stdout_logfile = /path/to/stdout_file
stderr_logfile = /path/to/stderr_file
```

```python
# job_name.py

import signal

class SignalCommand(BaseCommand):
    def __init__(self, *args, **kwargs):
        super(SignalCommad, self).__init__(*args, **kwargs)
        self._continue = True
        self.register_signal()
        
	def register_signal(self):
        # which signal should be caught denpends on how process manager
        # communicate with this job
        signal.signal(signal.SIGTERM, self.stop)
    
    def stop(self):
        self._continue = False
       
    @property
    def continue(self):
        return self._continue
    
    def handle(self, *args, **kwargs):
        pass
    
class Command(SignalCommand):
    
    def handle(self, *args, **kwargs):
        while self.continue:
            do_something
```

### IX Disposability

T: fast startup. Greacely shutdown, rubost at unexpected error.

Q: How to understand "For a worker process, graceful shutdown is achieved by returning the current job to the work queue. … **Implicit in this model is that all jobs are [reentrant](http://en.wikipedia.org/wiki/Reentrant_%28subroutine%29), which typically is achieved by wrapping the results in a transaction, or making the operation [idempotent](http://en.wikipedia.org/wiki/Idempotence).**"

A: Two approach to make a job reentrant. with the above code as an example:

```python
def stop(self):
    self._coninute = False  # this is idempotent, since, no matter call this how many time, the result is same as the initial call: value of self._continue is False
```

1. wrap the results in a transaction. How to understand the term "**transaction**" here?
2. making the operation idempotent. This makes sense. The idempotent is defined like this: "In [computer science](https://en.wikipedia.org/wiki/Computer_science), the term **idempotent** is used more comprehensively to describe an operation that will produce the same results if executed once or multiple times.[[9\]](https://en.wikipedia.org/wiki/Idempotence#cite_note-IBM-9) This may have a different meaning depending on the context in which it is applied. **In the case of [methods](https://en.wikipedia.org/wiki/Method_(computer_science)) or [subroutine](https://en.wikipedia.org/wiki/Subroutine) calls with [side effects](https://en.wikipedia.org/wiki/Side_effect_(computer_science)), for instance, it means that the modified state remains the same after the first call**."

### X. Dev/Prod Parity

**Keep development, staging and production as similar as possible**

### XI. Logs

T: **A twelve-factor app never concerns itself with routing or storage of its output stream.** It should not attempt to write to or manage logfiles….In staging or production deploys, each process’ stream will be captured by the execution environment, collated together with all other streams from the app, and routed to one or more final destinations for viewing and long-term archival….**The event stream for an app can be routed to a file**, or watched via realtime tail in a terminal. 

C: "It should not attempt to write to or manage log file…" Here, it makes sense to statement "it should not manage logfile", but it's not precisely to say "It should not attempt to **write to**". Since 12-factor indeed could route it's event stream to stdout, a log file, or backend service (for example, pass the log to a webservice). But yes, a most common scenario is, app routes log to log files, and an agent of log-anaylise system collects the log from the log file to the system for further handle.

Comments from [Logs Are Streams, Not Files](https://adam.herokuapp.com/past/2011/4/1/logs_are_streams_not_files/):

C1: A program using `stdout` for logging can use syslog without needing to implement any syslog awareness into the program, by piping to the standard `logger` command available on all modern unixes. A program which uses `stdout` is equipped to **log in a variety of ways without adding any weight to its codebase or configuration format**.

```Shell
# piping to syslog
$ mydaemon | logger

# piping to syslog as well as to log file
$ mydaemon | tee /var/log/mydaemon.log | logger
```

C2: There are other distributed logging protocols such as Splunk and Scribe, HeroKu log system logplex, syslog-as-a-service product like Loggy and Papertrail. Programs that log to `stdout` can be **adapted to work with a new protocol without needing to modify the program.** Simply pipe the program’s output to a receiving daemon just as you would with the `logger` program for syslog. Treating your logs as streams is a form of [future-proofing](http://en.wikipedia.org/wiki/Future_proof) for your application. **The flexibility of usinge new log protocols is reserved**.

C3: Logs are a stream, and it behooves everyone to treat them as such. Your programs should log to `stdout`and/or `stderr` and omit any attempt to handle log paths, log rotation, or sending logs over the syslog protocol. **Directing where the program’s log stream goes can be left up to the runtime container: a local terminal or IDE (in development environments), an Upstart / Systemd launch script (in traditional hosting environments), or a system like Logplex/Heroku (in a platform environment)**.

### XII. Admin Processes

T: One-off admin processes should be run in an identical environment as the regular [long-running processes](https://12factor.net/processes) of the app. They run against a [release](https://12factor.net/build-release-run), using the same [codebase](https://12factor.net/codebase) and [config](https://12factor.net/config) as any process run against that release. ...The same [dependency isolation](https://12factor.net/dependencies) techniques should be used on all process types. 