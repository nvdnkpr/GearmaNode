            ____                                 _   _           _
           / ___| ___  __ _ _ __ _ __ ___   __ _| \ | | ___   __| | ___
          | |  _ / _ \/ _` | '__| '_ ` _ \ / _` |  \| |/ _ \ / _` |/ _ \
          | |_| |  __/ (_| | |  | | | | | | (_| | |\  | (_) | (_| |  __/
           \____|\___|\__,_|_|  |_| |_| |_|\__,_|_| \_|\___/ \__,_|\___|


Node.js library for the [Gearman](http://gearman.org/) distributed job system.


## Features
* fully implemented Gearman Protocol
 * @TODO (RESET_ABILITIES, SET_CLIENT_ID, CAN_DO_TIMEOUT, ALL_YOURS, GRAB_JOB_UNIQ, JOB_ASSIGN_UNIQ)
* support for multiple job servers
 * load balancing strategy (`sequence` or `round-robin`)
 * recover time (when a server node is down due to maintenance or a crash, load balancer will use the recover-time as a delay before retrying the downed job server)
* support for miscellaneous string encoding supported by Node.js `Buffer` class
* careful API documentation
* rock solid tests
 * currently more than 90 test scenarios and 300 asserts
* in depth tested with gearman clients and workers written in other languages (Ruby, PHP, Java)


## Installation

    > npm install GearmaNode


## Usage
See [Geaman Manual](http://gearman.org/manual) to understand generic Gearman concepts.  
See [example](https://github.com/veny/GearmaNode/tree/master/example) folder for more detailed samples.

* [Client](#client)
 * [Submit job](#submit-job)
 * [Client events](#client-events)
* [Worker](#worker)
 * [Register function](#register-function)
 * [Worker events](#worker-events)
* [Job](#job)
 * [Job events](#job-events)
* [Job server](#job-server)
 * [Job server events](#job-server-events)
* [Multiple servers](#multiple-servers)
* [Error handling](#error-handling)

### Client
*The client is responsible for creating a job to be run and sending it to a job server. The job server will find a suitable worker that can run the job and forwards the job on.*  
-- Gearman Documentation --  

Instance of class `Client` must be created to connect a Gearman job server(s) and to make requests to perform some function on provided data.

```javascript
var gearmanode = require('gearmanode');
var client = gearmanode.client();
```

By default, the job server is expected on `localhost:4730`. Following options can be used for detailed configuration of the client:

 * **host** {string} hostname of single job server
 * **port** {number} port of single job server
 * **servers** {array} array of host,port pairs of multiple job servers
 * **loadBalancing** {'Sequence'|'RoundRobin'} name of load balancing strategy
 * **recoverTime** {number} delay in miliseconds before retrying the downed job server

```javascript
// special port
client = gearmanode.client({port: 4732});

// two servers: foo.com:4731, bar.com:4732
client = gearmanode.client({servers: [{host: 'foo.com', port: 4731}, {host: 'bar.com', port: 4732}]});

// two servers with default values: foo.com:4730, localhost:4731
client = gearmanode.client({servers: [{host: 'foo.com'}, {port: 4731}]});
```

#### Submit job

Client submits job to a Gearman server and futher processed by a worker via `client#submitJob(name, payload, options)`
where `name` is name of registered function a worker is to execute, `payload` is data to be processed
and `options` are additional options as follows:

* **background** {boolean} flag whether the job should be processed in background/asynchronous
* **priority** {'HIGH'|'NORMAL'|'LOW'} priority in job server queue
* **encoding** - {string} encoding if string data used
* **unique** {string} unique identifiter for the job @TODO

```javascript
// by default foreground job with normal priority
var job = client.submitJob('reverse', 'hello world!');

// background job
var job = client.submitJob('reverse', 'hello world!', {background: true});

// full configured job
var job = client.submitJob('reverse', 'hello world!', {background: false, priority: 'HIGH', encoding: 'utf8', unique: 'FooBazBar'});
```

Client-side processing of job is managed via emitted events. See [Job events](#job-events) for more info.

```javascript
var client = gearmanode.client();
var job = client.submitJob('reverse', 'hi');
job.on('complete', function() {
    console.log('RESULT: ' + job.response);
    client.close();
});
```

A client object should be closed if no more needed to release all its associated resources and socket connections. See the sample above.

#### Client events
* **submit** - when a job has been submited to job server, has parameter 'number of jobs waiting for response CREATED'
* **done** - when there's no submited job more waiting for state CREATED
* **connect** - when a job server connected (physical connection is lazy opened by first data sending), has parameter **job server UID**
* **disconnect** - when connection to a job server terminated (by timeout if not used or forcible by client), has parameter **job server UID**
* **close** - when Client#close() called to end the client for future use and to release all its associated resources
* **jobServerError** - whenever an associated job server encounters an error and needs to notify the client, has parameters **jobServerUid**, **code**, **message**
* **error** - when an unrecoverable error occured (e.g. illegal client's state, malformed data, socket problem, ...) or job server encounters an error and needs to notify client, has parameter **Error**


### Worker
*The worker performs the work requested by the client and sends a response to the client through the job server.*  
-- Gearman Documentation --  

Instance of class `Worker` must be created to connect a Gearman job server(s), where it then informs the server(s) of all different functions the Worker is capable of doing.

```javascript
var gearmanode = require('gearmanode');
var worker = gearmanode.worker();
```
By default, the job server is expected on `localhost:4730`. Following options can be used for detailed configuration of the worker:

 * **host** [see Client](#client)
 * **port** [see Client](#client)
 * **servers** [see Client](#client)
 * **withUnique** {boolean} flag whether a job will be grabbed with the client assigned unique ID @TODO

#### Register function

A function the worker is able to perform can be registered via `worker#addFunction(name, callback, options)`
where `name` is a symbolic name of the function, `callback` is a function to be run when a job will be received
and `options` are additional options as follows:

* **timeout** {number}  timeout value in seconds, the job server will mark the job as failed and notify any listening clients @TODO
* **toStringEncoding** {string} if given received payload will be converted to `String` with this encoding, otherwise payload turned over as `Buffer` @TODO

The worker function `callback` gets parameter [Job](#job) which is:

* job event emitter (see [Job events](#job-events))
* value object to turn over job's parameters
* interface to send job notification/information to the job server

```javascript
worker.addFunction('reverse', function (job) {
    var rslt = job.payload.toString().split("").reverse().join("");
    job.workComplete(rslt);
});
```
It tries to connect to ALL job servers and fires `error` if one registration fails.
A registered function can be unregistered via `worker#removeFunction`.

#### Worker events
* **close** - when Worker#close() called to close the worker for future use and to release all its associated resources
* **jobServerError** - whenever an associated job server encounters an error and needs to notify the worker, has parameters **jobServerUid**, **code**, **message**
* **error** - when a fatal error occurred while processing job (e.g. illegal worker's state, socket problem, ...) or job server encounters an error and needs to notify client, has parameter **Error**


### Job

The `Job` object is an encapsulation of job's attributes and interface for next communication with job server.
Additionally is the object en emitter of events corresponding to job's life cycle (see [Job events](#job-events)).

The `job` has following getters

* **name** - name of the function, [Client/Worker]
* **payload** - transmited/received data (Buffer or String) [Client/Worker]
* **jobServerUid** - unique identification (UID) of the job server that transmited the job [Client/Worker]
* **handle** - unique handle assigned by job server when job created [Client/Worker]
* **encoding** - encoding to use [Client]

and methods

* **getStatus** - sends request to get status of a background job [Client]
* **workComplete** - sends a notification to the server (and any listening clients) that the job completed successfully [Worker]
* **reportStatus** - reports job's status to the job server [Worker]
* **reportWarning** - sends a warning explicitly to the job server [Worker] @TODO
* **reportError** - to indicate that the job failed [Worker]
* **reportException** - to indicate that the job failed with exception (deprecated, provided for backwards compatibility) [Worker]
* **sendData** - send data before job completes [Worker]

#### Job events
* **submited** - when job submited via a job server; server UID stored on the job [Client]
* **created** - when response to one of the SUBMIT_JOB* packets arrived and job handle assigned [Client]
* **status** - to update status information of a submitted jobs [Client]
 * in response to a client's request for a **background** job
 * status update propagated from worker to client in case of a **non-background** job
* **complete** - when the non-background job completed successfully [Client]
* **failed** - when a job has been canceled by invoking Job#reportError on worker side [Client]
* **exception** - when the job failed with the an exception, has parameter **text of exception** [Client]
* **timeout** - when the job has been canceled due to timeout [Client/Worker]
* **close** - when Job#close() called or when the job forcible closed by shutdown of client or worker [Client/Worker]
* **error** - when communication with job server failed [Client/Worker]


### Job server
Class `JobServer` represents an abstraction to Gearman job server (gearmand).
Accessible job server(s) are stored in array `jobServer` on instance of Client/Worker.
The class introduces following methods:

* **echo** - sends the job server request that will be echoed back in response
* **setOption** - sends the job server request to set an option for the connection in the job server

```javascript
var client = gearmanode.client();
var js = client.jobServers[0];

js.once('echo', function(resp) {
    console.log('ECHO: response=' + resp);
    client.close();
});
js.echo('ping')
```

#### Job server events
* **echo** - when response to ECHO_REQ packet arrived, has parameter **data** which is opaque data echoed back in response
* **option** - issued when an option for the connection in the job server was successfully set, has parameter **name** of the option that was set
* **jobServerError** - whenever the job server encounters an error, has parameters **code**, **message**


### Multiple servers
Many of Gearman job servers can be started for both high-availability and load balancing.

[Client](#client) is able to communicate with multiple servers with one of the following load balancing strategy:

* default mode is `Sequence` which calls job server nodes in the order of nodes defined by the client initialization (next node will be used if the current one fails)
* `RoundRobin` assigns work in round-robin order per nodes defined by the client initialization.

```javascript
// default load balancer
client = gearmanode.client({ servers: [{host: 'foo.com'}, {port: 4731}] });

// desired load balancer and recover time
client = gearmanode.client({ servers: [{host: 'foo.com'}, {port: 4731}], loadBalancing: 'RoundRobin', recoverTime: 10000 });
```

[Worker](#worker) can be initialized with multiple servers in order to register a function on each of them.

### Error handling
Although exceptions are supported in JavaScript and they can be used to communicate an error, due to asynchronous concept of Node.js it can be a bad idea.
According to Node.js best practices following error handling is introduced in GearmaNode.

#### Synchronous errors
A synchronous code returns an `Error` object if something goes wrong. This happens mostly in input value validation.

#### Asynchronous errors
In asynchronous code an error event will be emitted via `EventEmitter` on corresponding object if something goes wrong.
This happens mostly by network communication failure or if a gearman service fails.


## Class diagram

[![](https://raw.github.com/veny/GearmaNode/master/ooad/Classes.png)](https://raw.github.com/veny/GearmaNode/master/ooad/Classes.png)


## Tests

    > cd /path/to/repository
    > mocha

Make sure before starting the tests:

* job server is running on localhost:4730
* `mocha` test framework is installed


## Author

* vaclav.sykora@gmail.com
* https://plus.google.com/115674031373998885915


## License

* [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)
* see [LICENSE](https://github.com/veny/GearmaNode/tree/master/LICENSE) file for more details
